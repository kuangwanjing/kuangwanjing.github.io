---
layout: post
title:  "Write A Circuit Breaker In Go"
date:   2019-07-07 17:29:00 -0400
categories: 
 - designpattern 
---

This posts intrduces a way to write a [circuit breaker](https://martinfowler.com/bliki/CircuitBreaker.html) with Go channels.

To build a general circuit breaker for various purposes, the breaker is designed to decouple from the logic of the circuit. For example, a circuit breaker for an HTTP API can be used by the HTTP agent which handles the parameters, sends requests and retries for failures. The HTTP agent determines whether the calls can be made according to the state of the breaker before making requestsTherefore, the breaker should expose methods to accept the queries of state, receive the feedback from the agent to update the state. The exposed methods should be: 

1. IsAvailable: Determine whether the circuit is "good" enough to pass. 
2. MakeSuccess/MakeFailure: Enable the caller to give feedback to the breaker.

Hence, the workflow of using a circuit breaker to protect some circuit can be like the following:

![circuit breaker](/assets/img/circuitbreaker.png)

For a breaker, there are three possible states: closed, open and half-open. The result of IsAvailable() is determines by the state of the breaker. So the scatch of this method is like this:

{% highlight go %}
    // version 1 data race
    const State int
    const (
        Closed State = iota
        Open
        HalfOpen
    )
    type CircuitBreaker struct {
        state       State
        successCnt  int
        failureCnt  int
    }
    func (this *CircuitBreaker) IsAvailable() bool {
        switch this.state {
            case Closed:
                return true
            case Open:
                return false
            case HalfOpen:
                // roll a dice to determine whether the request can pass the circuit
                if ... {
                    return true
                } else {
                    return false
                }
        }
    }
{% endhighlight %}

And MakeSuccess or MakeFailure are just updating the numbers of successful calls and failed calls. If the failure rate reaches a certain threshold, the circuit breaker is switched to open. However, in reality the queries and state updates happen concurrently which causes *data race* potentially.

*Question: How to avoid data race while having update queries to the breaker?*

To avoid the data race, one of common technology is to use lock to protect critical section. However Here channel is used to serialize the read(query) and write(state update) towards the breaker data structure, since this is a more natural way than the lock-style protection. Because the breaker maintains the state by collecting external events(success or failure) and emitting signals(when state half-open times up, it should be back to open), thus the channel here delivers the events or signals to the "brain" of the circuit and let it decide what to do next.

{% highlight go %}
    // version 2: add some channels to avoid data race
    type CircuitBreaker struct {
        state       State
        successCnt  int
        failureCnt  int
        feedbackCh  chan bool       // to receive the calling feedback from the caller (writer)
        queryCh     chan struct{}   // to catch the query events from the caller (reader)
        queryRetCh  chan bool       // used to return the query
    }

    // run the "brain" of the circuit breaker
    func (this *CircuitBreaker) Run() {
        go func () {
            for {
                select {
                    case <- queryCh:
                        // determined by the state                                    
                        if this.state == Closed {
                            this.queryRetCh <- true
                        }
                        else {
                            // other logic in the IsAvailable() aforementioned... 
                            this.queryRetCh <- false
                        }
                    case <- this.feedbackCh:
                        this.successCnt += 1 // or this.failureCnt += 1
                        // if the breaker is closed and error rate reaches threshold, open the breaker
                        this.state = Open 
                        // or the breaker should be reset back to Open
                        this.state = Closed
                        // do some cleaning ... 
                    // well, some timer should be listened too, you know what a big clumsy chunk should be to implement the breaker.
                }
            }
        }()
    }

    func (this *CircuitBreaker) IsAvailable() {
        this.queryCh <- true
        return <-this.queryRetCh
    }

    func (this *CircuitBreaker) MakeSuccess() {
        this.feedbackCh <- true
    }

    func (this *CircuitBreaker) MakeFailure() {
        this.feedbackCh <- false
    }
{% endhighlight %}

The second version handles the data race correctly but too much logic into the Run method. A lots of details are not mentioned including the cleanup when switching state. Therefore, here define two kinds of activities that the breaker has: handling external events including state queries and calls feedback; maintaining internal state transition. State machine can help to simplify the state updating logic by encapsulating state maintainance and transition logic.

{% highlight go %}
    // version 3: employ state machines and simplify the event channels
    type CircuitBreaker struct {
        state       State
        feedbackCh  chan bool       // external events
        queryCh     chan struct{}   // also external events
        stateCh     chan State      // interal state transition events
        queryRetCh  chan bool       // used to return the query

        // define the state machine
        closedFn    *ClosedStateMachine 
        openFn      *OpenStateMachine
        halfFn      *HalfOpenStateMachine
    }

    // intialize state machines and make sure there is always only one machine is running.
    func (this *CircuitBreaker) Run() {
        // initializes the fields of the breaker
        // ... 
        this.closedFn = &ClosedStateMachine{}
        this.openFn  = &OpenStateMachine{}
        this.halfFn = &HalfOpenStateMachine{}

        // define a stopping channel to stop the state machine
        stopCh := make(chan struct{})
        stoppedCh := make(chan struct{}) // a stopped signal from the current state machine
        this.startNewState(newState, stopCh, stoppedCh)

        go func () {
            for {
                select {
                    case <- queryCh:
                        // determined by the state                                    
                        this.queryRetCh <- this.check() // encapsulate the details in check()
                    case newState := <- stateCh:
                        this.stopCurrentState(stopCh, stoppedCh)
                        stopCh = make(chan struct{})
                        stoppedCh = make(chan struct{})
                        this.startNewState(newState, stopCh, stoppedCh)
                }
            }
        }()
    } 
    
    func (this *CircuitBreaker) stopCurrentState(stopCh chan struct{},  stoppedCh chan struct{}) {
        close(stopCh) // sending a signal towards the current state machine
        <-stoppedCh // receive the stopped signal from the state machine
    }

    func (this *CircuitBreaker) startNewState(state State, stopCh chan struct{},  stoppedCh chan struct{}) {
        switch state {
            case Close:
                this.closedFn.Start(stopCh, stoppedCh, this.stateCh, this.feedbackCh)
            case Open:
                this.openFn.Start(stopCh, stoppedCh, this.stateCh)
            case HalfOpen:
                this.halfFn.Start(stopCh, stoppedCh, this.stateCh, this.feedbackCh)
        }
    }
{% endhighlight %}

Unlike implementation of the second version, the count events are redirected to state machines(both state Closed and state HalfOpen are interested in them). And for each state, implement the Start method. The Start method for each start machine is similar to each other: maintain the state and stop the running and do some cleaning while receiving the stop signal from the breaker.

{% highlight go %}
    // to illustrate the implementation, some details are hidden
    type ClosedStateMachine struct {
        successCnt      int
        failureCnt      int
        statsWindow     time.Ticker
    }

    func (this *ClosedStateMachine) Start(stopCh chan struct{},  stoppedCh chan struct{}, 
        stateCh chan State, feedbackCh chan bool) {

        successCnt = 0
        failureCnt = 0
        statsWindow = ... // initialize the size of window as needed

        go func() {
            defer func() {
                close(stoppedCh) // finish stopping the machine
            }() 

            for {
                select {
                    case <- stop:
                        // do cleaning:
                        statsWindow.Stop()
                        break
                    case rst := <-feedbackCh:
                        if rst {
                            successCnt += 1 
                        } else {
                            failureCnt += 1 
                        }
                    case <-this.statsWindow:
                        // if the error rate exceeds the threshold, signal a state transistion: 
                        stateCh <- Open
                }
            }
        }()
    }
{% endhighlight %}

In this way, a neat implementation of circuit breaker is defined. The remaining question is whether the serialization of the events is the potential bottleneck since all reads and writes are waiting in line? The answer is no for the following two reasons:

1. The critical section is short enough. For most cases, the circuit breaker receives more queries than the state transistion signals. And handling the queries is just to examine the state so this is fast enough for the breaker to handles as many queries as it can. When handling state transistions, the case is a bit more complicated than the first case. It follows this path: breaker catches the signal from the state machine, then signal a stop via stopChannel, the state machine catches this signal in the loop and do some cleaning and signal the breaker that it is done, then the breaker make another state machine run. But consider the frequency of these events are very low compared to the query events, the complication is acceptable.

2. Different events are handled by different components. For illustration, the breaker itself only handles the queries and the state transistion events; the state machines handle the success and failure event and handle the state transistion logic. Therefore, a complicated event would not become a bottleneck of other events. The routines of different components can be scheduled to different CPU cores to improve the performance too.

Other tips:

1. The initialization of the breaker can involve some parameterization.

2. Use buffered channel instead of unbuffered channel for some events so that the senders won't be blocked. For example, the feedbackCh can be a buffered channel so that the caller can continue the execution as soon as possible after giving the feedback.

Conclusion:

The implementation of circuit breaker follows this path:

1. Write a simple breaker but it is risky for concurrency.

2. Avoid the data race using channels to serialize the read and write events.

3. Employ state machines to decouple the state maintainance from the breaker and simplify the events handling of the breaker to remove the bottleneck of queue waiting.  

More details of implementation, please take a look at [Github example](https://github.com/kuangwanjing/designpatterns/tree/master/go/circuit_breaker). Thank you for reading!
