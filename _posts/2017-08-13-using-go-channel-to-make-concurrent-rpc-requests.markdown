---
layout: post
title:  "Using Go Channel To make Concurrent RPC Requests"
date:   2017-10-15 08:00:00 +0800
categories: go
---

This design is inspired by [Go Concurrent Pattern Video by Rob Pike](https://www.youtube.com/watch?v=f6kdp27TYZs&t=12s)

Problem:*Since individual business is encapsulated as independent service for decoupling purpose, several RPCs are provided. And here respective result of each RPC modules are needed. For the sake of saving running time, concurrent requests are necessary.*

This initial design of this problem is just like what is introduced in the video about the implementation of the simple searching engine.

![Alt text]({{ site.url }}/assets/concurrent_design1.jpg)

By using go channel and select statement, we could make concurrent RPCs, and merge the results correctly.

{% highlight go %}

    //Result has three properties, for each is acquired by different RPCs.
    type Result struct {
        Property1 *P1
        Property2 *P2
        Property3 *P3
    }

    func MakeRPCs() *Result {

        rst := new(Result)

        //making three data channels
        c1 := make(chan *P1)
        c2 := make(chan *P2)
        c3 := make(chan *P3)

        go func() { c1 <- Rpc1() } ()
        go func() { c2 <- Rpc2() } ()
        go func() { c3 <- Rpc3() } ()

        //timeout channel is a technique to avoid timeout request
        timeout := time.After(time.Duration(500) * time.Millisecond)

        for i := 0; i < 3; i++ {
            select {
                case data := <- c1:
                    rst.Property1 = data
                case data := <- c2:
                    rst.Property2 = data
                case data := <- c3:
                    rst.Property3 = data
                case <- timeout:
                    break
            }
        }

        return rst
    }
{% endhighlight %}

However, this implementation needs to be upgraded to satisfy the following requirements:

1. Not all the modules are needed to be requested under different circumstances. Here a filter for RPCs is necessary.

2. The availibility of these remotes modules should be supervised. If the availibilty of a certain module is below a threshold, that means the module is in bad condition. The calls to that module should be prevented to avoid cascading damage to the program itself.

Therefore we need:

1. A routine to check the availibilty of all modules and stop request to "bad" modules automatically.

2. A filter for determining the exact RPCs requests.

First We need to redesign the pattern by introducing status flags and a checking function corresponding to all modules.

{% highlight go %}
    type Module1Handler struct {
        status bool
    }

    type StatusInformation struct {
        ModuleID string
        Status   bool
    }

    type (this *Module1Handler) Check(statusCh chan bool) {
        for {
            //implement a checking procedure here
            if !check() {
                statusCh <- StatusInformation({"m1", false})
            } else {
                statusCh <- StatusInformation({"m1", true})
            }
        }
    }

    type (this *Module1Handler) Request(query *Query, dataCh chan *P1) {
        //implement of making request to module1.
        dataCh <- request1(query)
    }
{%endhighlight %}

Give each module an ID and the filter outputs a list of ID. 

{% highlight go %}
    func Filter(query *Query) []string {
        //....
        return moduleIDs
    }
{%endhighlight %}

The main flow of the program starts checking routine, accepts ID list and checks the status of each requested modules, create routines to request to available modules and rejects those not available. Finally it merges data and makes output.

{% highlight go %}

    type RPCMaker struct {
        status1 bool
        status2 bool
        status3 bool
        statusCh chan *StatusInformation
        handler1Channel chan chan *Module1Handler
        handler2Channel chan chan *Module2Handler
        handler3Channel chan chan *Module3Handler
    }

    func (this *RPCMaker) Init() {
        statusCh := make(chan bool)
        this.status1 = true
        this.status2 = true
        this.status3 = true

        handler1 := new(Module1Handler) 
        handler2 := new(Module2Handler) 
        handler3 := new(Module3Handler) 

        //starts checker
        go handler1.Check(statusCh)
        go handler2.Check(statusCh)
        go handler3.Check(statusCh)

        for {
            select {
                case information := <- this.statusCh:
                    switch information.ModuleID {
                        case "m1":
                            this.status1 = information.Status
                            break
                        case "m2":
                            this.status2 = information.Status
                            break
                        case "m3":
                            this.status3 = information.Status
                            break
                    }
                case moduleRequestCh := <- this.handler1Channel:
                    //if status of module1 is true, then return a 
                    if this.status1 {
                        moduleRequestCh <- new(Module1Handler)
                    } else {
                        moduleRequestCh <- nil
                    }
                case moduleRequestCh := <- this.handler2Channel:
                //do the same for module2
                case moduleRequestCh := <- this.handler3Channel:
                //do the same for module3
            }
        }
    }

    func (this *RPCMaker) MakeRPCs(query *Query) *Result {

        moduleIDs := Filter(query)
        dataCh1 := make(ch *P1)
        dataCh2 := make(ch *P2)
        dataCh3 := make(ch *P3)
        
        for _, id := range moduleIDs {
            switch id {
                case "m1":
                    go func() {
                        handlerCh := make(chan *Module1Handler)
                        handler1Channel <- handlerCh
                        if handler1Channel != nil {
                            dataCh1 <- handler1Channel.Request(query)
                        } else {
                            dataCh1 <- nil
                        }
                    } ()
                    break
                case "m2":
                    //do the same for module2
                    break
                case "m3":
                    //do the same for module3
                    break
            }
        }

        timeout := time.After(time.Duration(500) * time.Millisecond)

        rst := new(Result)

        for range moduleIDs {
            select {
                case data := <- dataCh1:
                    rst.P1 = data
                case data := <- dataCh2:
                    rst.P2 = data
                case data := <- dataCh3:
                    rst.P3 = data
                case <- time:
                    break
            }
        }

        return rst
    }
{%endhighlight %}

The main technique employed here is before directly making request to remote modules, first check the status of each module and get the handler for each module is it is available. The channel of a channel passes a handler back to the MakeRPCs function. And the init function indefinitely checks availability and sends back the handlers to the MakeRPCs function.  

In sum, Go channel can deliver any type of data including channel itself. By making the use of the channel of a channel, the design provides somewhat a resource management module, controlling whether a request can be made or not. The model here can be upgraded into a resource pool by recording the used resource and keep the request waiting for the resource by channel blockage. No queue is needed here. Therefore this model can be easily transplanted into task manager which keep request in line to share resource.
