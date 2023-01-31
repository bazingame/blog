---
title: "保护你的程序 sony/gobreaker"
summary: 保护你的程序 sony/gobreaker
date: 2022-08-22
weight: 1
tags: ["golang"]
---

微服务中常常有着复杂的依赖关系，在调用链路上如果某个微服务的响应时间过长或者服务不可用，那么上游服务的系统资源占用就会越来越多，进而引起系统崩溃，发生雪崩效应。熔断机制是解决以上问题的一个方式。

## 断路器模式

在家庭电路中有一个叫断路器的安全设备，当出现电路过载、短路、漏电等情况时，就会发生跳闸，防止出现安全事故。类比到上面描述的业务问题场景，我们需要在系统中实现一个类似断路器功能的组件，用于阻止重复尝试很可能失败的调用。

在断路器模式中，断路器组件需要监测到最近失败的调用，并且利用这些信息去决定新的调用是否执行，还是立即抛出异常。当断路器组件“跳闸”之后，还需要能探测被调用服务是否恢复正常，

断路器模式的代码实现，使用了有限状态机的思想。最基本的实现有三种状态：

- 关闭（Closed)：调用正常执行。断路器组件对最近失败的调用进行计数，当达到阈值时，则断路器组件“跳闸”，进入“打开”状态。

- 打开（Open）：调用请求会立即失败，断路器组件抛出异常。

- 半打开（Half-Open）：当处于“打开”状态时，会启动一个超时定时器，当超时后，断路器组件会进入“半打开”状态，此时允许执行部分调用，断路器会对成功执行的调用进行计数，达到阈值后，会认为被调用服务恢复正常，断路器状态回到“关闭”状态，如果有请求出现失败，则回到“打开”状态。

  ![circuit-breaker-diagram.png](https://cdn.gocn.vip/forum-user-images/20220821/08ee5bcf876c45baab4e39320d1e93eb.jpg)

## Golang中断路器的实现：sony/gobreaker

gobreaker 是在上述状态机的基础上，实现的一个熔断器。

### 熔断器的定义

```go
type CircuitBreaker struct {
  name          string
  maxRequests   uint32  // 最大请求数 （半开启状态会限流）
  interval      time.Duration   // 统计周期
  timeout       time.Duration   // 进入熔断后的超时时间
  readyToTrip   func(counts Counts) bool // 通过Counts 判断是否开启熔断。需要自定义
  onStateChange func(name string, from State, to State) // 状态修改时的钩子函数

  mutex      sync.Mutex // 互斥锁，下面数据的更新都需要加锁
  state      State  // 记录了当前的状态
  generation uint64 // 标记属于哪个周期
  counts     Counts // 计数器，统计了 成功、失败、连续成功、连续失败等，用于决策是否进入熔断
  expiry     time.Time // 进入下个周期的时间
}
```

其中，如下参数是我们可以自定义的：

- MaxRequests：最大请求数。当在最大请求数下，均请求正常的情况下，会关闭熔断器
- interval：一个正常的统计周期。如果为0，那每次都会将计数清零
- timeout: 进入熔断后，可以再次请求的时间
- readyToTrip：判断熔断生效的钩子函数
- onStateChagne：状态变更的钩子函数

### 请求的执行

熔断器的执行操作，主要包括三个阶段；

- 请求之前的判定
- 服务的请求执行
- 请求后的状态和计数的更新

```go
// 熔断器的调用
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {

  // ①请求之前的判断
  generation, err := cb.beforeRequest()
  if err != nil {
    return nil, err
  }

  defer func() {
    e := recover()
    if e != nil {
      // ③ panic 的捕获
      cb.afterRequest(generation, false)
      panic(e)
    }
  }()

  // ② 请求和执行
  result, err := req()

  // ③ 更新计数
  cb.afterRequest(generation, err == nil)
  return result, err
}
```

### 请求之前的判定操作

请求之前，会判断当前熔断器的状态。如果熔断器以开启，则不会继续请求。如果熔断器半开，并且已达到最大请求阈值，也不会继续请求。

```go
func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
  cb.mutex.Lock()
  defer cb.mutex.Unlock()

  now := time.Now()
  state, generation := cb.currentState(now)

  if state == StateOpen { // 熔断器开启，直接返回
    return generation, ErrOpenState
  } else if state == StateHalfOpen && cb.counts.Requests >= cb.maxRequests { // 如果是半打开的状态，并且请求次数过多了，则直接返回
    return generation, ErrTooManyRequests
  }

  cb.counts.onRequest()
  return generation, nil
}
```

其中当前状态的计算，是依据当前状态来的。如果当前状态为已开启，则判断是否已经超时，超时就可以**变更状态到半开**；如果当前状态为关闭状态，则通过周期判断是否进入下一个周期。

```go
func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
  switch cb.state {
  case StateClosed:
    if !cb.expiry.IsZero() && cb.expiry.Before(now) { // 是否需要进入下一个计数周期
      cb.toNewGeneration(now)
    }
  case StateOpen:
    if cb.expiry.Before(now) {
      // 熔断器由开启变更为半开
      cb.setState(StateHalfOpen, now)
    }
  }
  return cb.state, cb.generation
}
```

周期长度的设定，也是以据当前状态来的。如果当前正常（熔断器关闭），则设置为一个interval 的周期；如果当前熔断器是开启状态，则设置为超时时间（超时后，才能变更为半开状态）。

### 请求之后的处理操作

每次请求之后，会通过请求结果是否成功，对熔断器做计数。

```go
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
  cb.mutex.Lock()
  defer cb.mutex.Unlock()

  now := time.Now()

  // 如果不在一个周期，就不再计数
  state, generation := cb.currentState(now)
  if generation != before {
    return
  }

  if success {
    cb.onSuccess(state, now)
  } else {
    cb.onFailure(state, now)
  }
}
```

如果在半开的状态下：

- 如果请求成功，则会判断当前连续成功的请求数 大于等于 maxRequests， 则可以把状态由**半开状态转移为关闭状态**
- 如果在半开状态下，请求失败，则会直接将**半开状态转移为开启状态**

如果在关闭状态下：

- 如果请求成功，则计数更新
- 如果请求失败，则调用readyToTrip 判断是否需要将状态**关闭状态转移为开启状态**

## 小结

对于频繁请求一些远程或者第三方的不可靠的服务，存在失败的概率还是非常大的。使用断路器的好处就是可以使我们自身的服务不被这些不可靠的服务拖垮，一方面可以减少依赖服务对自身的依赖，防止出现雪崩效应；另一方面降低请求频率以方便上游尽快恢复服务。

## References

[Golang中使用断路器 (yangxikun.com)](https://yangxikun.com/golang/2019/08/10/golang-circuit.html)

[Golang 熔断器 · 搬砖程序员带你飞 (lpflpf.cn)](https://blog.lpflpf.cn/passages/circuit-breaker/)

[Circuit Breaker pattern - Azure Architecture Center | Microsoft Docs](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker#context-and-problem)