---
title: "Talk about Go: A simple implement of Circuit Breaker"
date: 2022-02-25
<!-- thumbnailImagePosition: left -->
<!-- thumbnailImage: /img/go-context.jpg -->
<!-- thumbnailImage: https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/cbk/cbk_front.png -->
categories:
- Go
- Middleware
tags:
- circuit breaker
- Talk about Go

metaAlignment: center
---

A simple implement of circuit breaker with Go, trigger for fail-fast and return quickly when your API comes to **high error rate durning a period of time**. 
It's a protection strategy that prevents the upstream from retrying repeatedly so that the downstream is overwhelmed.
<!--more-->

![熔断器](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/cbk/cbk_front.png)

## Foreword
Circuit Breaker can be said to be a high-frequency word that appears in microservice calls. It first came from the fuse of the old home circuit (power overload-tripping), and was later used by the stock market industry to describe the occurrence of an abnormal situation, temporarily suspending trading, and starting to protect the market scene. 

In a microservice link, a circuit breaker refers to **multiple exceptions/timeouts** on the API in a short period of time (**excessive error rate**), 
## Related concepts
- **dynamic adjustment**  
The core function of the fuse is "dynamic adjustment", that is to say, within a certain time window, when the number of errors on the interface, or the ratio reaches a dangerous value, the fuse must **hold** hold the request, return quickly (failure), and inform the upstream interface **Need to calm down* *, mainly has the following parameters:
  - **threshold**：Including two indicators: the minimum number of statistics and the proportion of errors. The minimum number of statistics is used to capture high-frequency requests, that is, the amount of requests must be at least greater than that before the circuit breaker starts to care about the error rate. Otherwise, a small number of failures on other low-frequency interfaces will trigger the circuit breaker.
  - **round interval**：the time window of the fuse scheduling, which can be analogous to the unit time in the current limiting algorithm, that is, how long it takes to reset the total number of times and the error count of the interface in the fuse period.
- **recover interval**：after breaker trigged, how long is the idle period to give the interface retry permission, and try to correct it and restore it to a normal state.

The main difference between **round interval** and the **recover interval** is that after the fuse resets the count, the interface must reach the error rate again to trigger the fuse, while the recovery interval is only to allow the current blown interface request, and the error rate is still very high. If you try again If the request is still not recovered, keep the blown state and wait for the next recovery cycle or reset cycle. So **recovery period** < **round interval**. 

Regarding the state transition, this example diagram is more vivid:

![CBK status change](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/cbk/en-CBK-state.png)
- **AOP Pattern**  
Being familiar with the basic purpose of the fuse, let's analyze the location of its components in the project.
I believe you will understand the concept of slice-oriented. Many common components, such as traffic restriction and circuit breaker protection, are slice-oriented modules, which are usually used to collect and report traffic conditions and protect downstream according to specific rules. The most common is to put them at the gateway layer.   

    **As shown in the figure below:**
    ![AOP location](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/cbk/en-CBK-AOP.png)
## Code implement
Let's look at the code implementation part.
### Interface definition
Based on the characteristics of the circuit breaker, first define the circuit breaker interface ``CircuitBreaker```, where the key refers to the API collected by the circuit breaker, which can be the request identifier on the link to distinguish different requests.
```go
type CircuitBreaker interface {
    // check if your api(key) allow to access
    CanAccess(key string) bool
    // mark failed api result
    Failed(key string)
    // mark succeed api result
    Succeed(key string)
}
```
### Interface implement
Based on several functional characteristics agreed upon above, create ``CircuitBreakerImp``` as the implementation body
```go
type CircuitBreakerImp struct {
    lock            sync.RWMutex
    apiMap          map[string]*apiSnapShop 
    minCheck        int64                   
    cbkErrRate      float64                 
    recoverInterval time.Duration           
    roundInterval   time.Duration           
}
```
The ``apiMap`` of the circuit breaker implementation body mainly stores the list of collection APIs, which is used to represent the overall snapshot of each API collected at the current stage.
```go
type apiSnapShop struct {
    isPaused   bool 
    errCount   int64
    totalCount int64

    accessLast int64 // The last access time of the api
    roundLast  int64 // fuse cycle time
}
```
----
**Circuit breaker status**, based on each failed request, the request snapshot will be stored in the apiMap of the current ```CircuitBreakerImp``` instance, so the implementation of the ```IsBreak()``` interface is relatively Simple, just return its status directly:
```go
func (c *CircuitBreakerImp) IsBreak(key string) bool {
    c.lock.RLock()
    defer c.lock.RUnlock()

    // Find out whether the current key (API) has reached a fuse
    if api, ok := c.apiMap[key]; ok {
        return api.isPaused
    }
    return false
}
```
----
**Access record**，Every time the gateway receives a request, it will report the number of interface accesses and the information on whether the return is successful or not to the circuit breaker at the origin of the request and the return of the request, as shown in the AOP picture above, the ```accessed()``` function is used to report each time the API is requested.  
In addition, as mentioned earlier, the circuit breaker has a **reset period**, that is, after how long the interface will be accessed again, its snapshot count will be reset. Therefore, at each ```access```, an additional judgment is required. Used to reset API counts.  
```go
// accessed record
func (c *CircuitBreakerImp) accessed(api *apiSnapShop) {
    /*
        whether now it is older than the cycle time?
        - yes: reset counter
        - no: update counter
    */
    now := time.Now().UnixNano()
    if util.Abs64(now-api.roundLast) > int64(c.roundInterval) {
        if api.roundLast != 0 {            
            log.Warnf("# Trigger circuit breaker windows，reset API counter!")
        }
        api.errCount = 0
        api.totalCount = 0
        api.roundLast = now
    }
    api.totalCount++
    api.accessLast = now
}
```
----
**report success**，After the gateway receives the successful response code, it will record its success. Another detail is why the ```accessed()``` function above does not need to add a mutex, it is actually needed, but in ```accessed()``` function is added at the upper level of the call, that is, in the subsequent ```Successd()``` and ```Failed()``` function bodies.
Let's first look at the implementation of ```Successd()```, **If the current api is recorded in the circuit breaker (it has failed), then close the circuit breaker after success and give the request**, even if the api fails again, it will be based on Request error rate redistribution circuit breaker.
```go
/*
    Succeed record
    Only collect which api in global map,
    check whether is's paused:
    - yes, cancel the paused breaker state.
*/
func (c *CircuitBreakerImp) Succeed(key string) {
    c.lock.Lock()
    c.lock.Unlock()

    if api, ok := c.apiMap[key]; ok {
        c.accessed(api)
        if api.isPaused {
            log.Warnf("# Trigger API: %v access succeed.", key)
            api.isPaused = false
        }
    }
}
```
----
**Reporting failed access**, the logic of counting failures is relatively simple. For the interface that fails for the first time, add it to the health check map. For the interface that has been recorded, it is necessary to judge the error rate and update it in time.   
In addition, the premise of triggering the circuit breaker is the request The amount has to reach a certain threshold.
```go
/*
    Failed record
    whether exists in api map,
        - yes:
            - record access and error count
            - the percentage of failures reaches the threshold? yes, mark as paused
        - no:
            update in global map for access and error count
*/
func (c *CircuitBreakerImp) Failed(key string) {
    c.lock.Lock()
    defer c.lock.Unlock()

    if api, ok := c.apiMap[key]; ok {
        c.accessed(api)
        api.errCount++

        errRate := float64(api.errCount) / float64(api.totalCount)
        // access count reaches the threshold && failures rate above the fuse limit
        if api.totalCount > c.minCheck && errRate > c.cbkErrRate {
            log.Warnf("# Trigger CBK！: %v, total: %v, "+
                "errRate: %.3f.", key, api.totalCount, errRate)
            api.isPaused = true
        }
    } else {
        api := &apiSnapShop{}
        c.accessed(api)
        api.errCount++
        c.apiMap[key] = api
    }
}
```
----
**Access permission query**, Based on the core function of the function circuit breaker implemented above, it is necessary to provide the gateway layer with a circuit breaker status to obtain the current interface.   
The following is the **access query** provided for the caller.  

One question here, isn't the ```IsBreak()``` function already implemented above, why do we need the ```CanAccess()``` function?  
Well, because ```IsBreak()``` returns only the blown state of the interface, but don't forget that there is a "half-closed state" in the fuse phase, that is, if the fuse time passes the recovery period (cooling off period), then access rights can be released, so this part of the logic is processed in ```CanAccess()```, let's take a look at the code.  
Among them, the input key represents the identifier of the interface to be accessed by the gateway layer, such as ```/get-hello```, and the return value of the function indicates the accessible status of the interface.    
> During the fuse cycle, if the recovery period (cooling-off period) has passed, you can release access, or call the ```Successd()``` fuse state to temporarily recover, otherwise the fuse state will remain, and the interface will be restricted and fast The core code that fails is here.

```go
func (c *CircuitBreakerImp) CanAccess(key string) bool {
    /*
        return the api's status of isPaused
        - not paused, return true
        - paused, return whether current time oldder then recovery period
    */
    c.lock.RLock()
    defer c.lock.RUnlock()
    log.Debugf("# Cbk check accessable for api id-%v key", reqType)

    if api, ok := c.apiMap[key]; ok {
        log.Debugf("# Cbk detail for api id-%v key, total: %v, "+
            "errCount: %v, paused: %v", reqType, api.totalCount,
            api.errCount, api.isPaused)
            
        if api.isPaused {            
            latency := util.Abs64(time.Now().UnixNano() - api.accessLast)
            if latency < int64(c.recoverInterval) {
                return false
            }
            log.Warnf("# Trigger: The CBK has passed the recovery period: %v, key: %v!", c.recoverInterval, key)
        }
    }    
    return true
}
```

## Unit Test
Based on the above implementation, next write test code to cover the case, demonstrate and keep these two processes in loop scrolling:
- **Fail -> Keep failed -> Fuse on -> Cooldown time elapsed**  
- **Enter Recovery Period -> Continue Access**  

### Mock API Access
```go
const API_PREFIX = "/fake-api"
var (
    HasCbk = false
)

func StartJob(cbk *CircuitBreakerImp) {
    for {
        // 1 failure per second, parameter 0 means failed, 1 means success
        ReqForTest(cbk, 0)
        time.Sleep(time.Second * 1)
    }
}

// Build request scheduling, fuse to restore it and let it succeed 1 time
func ReqForTest(cbk *CircuitBreakerImp, req int) {
    // mock failed case
    mockAPI := API_PREFIX //+ strconv.Itoa(req)
    //log.Infof("Ready to reqForTest: %s, req-id-%v", HOST_PREFIX+mockAPI, req)

    if !cbk.CanAccess(mockAPI, req) {
        log.Warnf("Api: %v is break, req-id-%v, wait for next round or success for one...", mockAPI, req)
        HasCbk = true
        return
    } else {
        log.Infof("Request can access: %s, req-id-%v", HOST_PREFIX+mockAPI, req)
        // After the recovery period, after the circuit breaker is recovered, skip the error and let it succeed
        if HasCbk && req == 0 {
            HasCbk = false
            req = 1
            log.Warnf("Transfer fail to success: %s, req-id-%v", HOST_PREFIX+mockAPI, req)
        }
    }

    if req == 0 {
        log.Errorf("# Meet failed ReqForTest: %s", HOST_PREFIX+mockAPI)
        cbk.Failed(mockAPI)
    } else {
        log.Infof("# Meet success ReqForTest: %s", HOST_PREFIX+mockAPI)
        cbk.Succeed(mockAPI)
    }
}
```
### Initialize the fuse
Initialize the circuit breaker and assign relevant parameters
```go
cbk := &CircuitBreakerImp{}
cbk.apiMap = make(map[string]*apiSnapShop)
// 15s per round to reset the err rate
cbk.roundInterval = util.ToDuration(15 * time.Second)
// when breaker is triggered, recover for next try
cbk.recoverInterval = util.ToDuration(5 * time.Second)
// consider the api at least request for 5 time durning the round interval
cbk.minCheck = 5
// when error rate comes to 50%, circuit breaker triggered
cbk.cbkErrRate = 0.5
```

### Output explanation
![CBK status](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/cbk/en-cbk-output.png)

As we expected, although the failure rate of the first 5 requests on the interface is 100%, the request volume has not increased, so the circuit breaker is not triggered until continued fusing condition is met, breaker trigged.  
After the breaker trigged lasts for 5 seconds, it turns to the idle period, the API becomes accessible, and the interface is accessed again. If the error rate is still satisfied, the fuse state will continue to be restored.  

And after the reset cycle, the API count is reset, and it returns to the state where the program was started, and the interface resumes **continuous accessible** until the error rate is reached and the fuse is turned on.

## Summary
At this point, a simple circuit breaker implementation has been completed. At the beginning, many of the concepts are easy to get around. Later, combined with scene simulation and code debugging, as well as the state transition picture at the beginning, it is easy to understand. Although many off-the-shelf components are available out of the box, their internal implementation is not so unfathomable as long as you put some thought into it.

## Project link
https://github.com/pixeldin/cbk-s1mpl3
## References
**Circuit Breaker Pattern**  
https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn589784(v=pandp.10)  
**Sony's implement of gobreaker**  
https://github.com/sony/gobreaker  
**微服务架构中的熔断器设计与实现**  
https://mp.weixin.qq.com/s/DGRnUhyv6SS_E36ZQKGpPA