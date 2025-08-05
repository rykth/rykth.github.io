---
title: "Microservices Resilience Patterns"
date: 2025-08-05T12:19:58+05:30
description: "Microservices are small, separate parts of a bigger application that work on their own. But if one part stops working, it can cause problems for the whole system. That's where resilience patterns come in. These are smart ways to keep things running smoothly, even when something goes wrong. For example, a retry pattern tries again if a service doesn't respond at first. A circuit breaker stops sending requests to a broken service so the system doesn't get overloaded. Timeouts stop the system from waiting too long for a reply. These patterns help keep the app steady and fast, giving users a better experience even if some parts fail."
tags: [patterns, golang, microservices, resilience]
---

Microservices are small, separate parts of a bigger application that work on
their own. But if one part stops working, it can cause problems for the whole
system. That's where resilience patterns come in. These are smart ways to keep
things running smoothly, even when something goes wrong. For example, a retry
pattern tries again if a service doesnt respond at first. A circuit breaker
stops sending requests to a broken service so the system doesn't get overloaded.
Timeouts stop the system from waiting too long for a reply. These patterns help
keep the app steady and fast, giving users a better experience even if some
parts fail.

## Using Resilience Patterns to Keep Microservices Running Smoothly

Resilience patterns are smart techniques used to keep microservices working
well, even when something goes wrong or when there's a lot of traffic. In
systems where many services talk to each other over a network, one small failure
can cause big problems if not handled properly. These patterns help prevent that
by making sure the whole system doesn't break when one part fails. For example,
we can use Go (Golang) to build these patterns: the [context package](https://pkg.go.dev/context) 
helps set time limits (timeouts), the [gobreaker library](https://github.com/sony/gobreaker)  
adds a circuit breaker to stop sending requests to broken services, and the
[backoff library](https://github.com/cenkalti/backoff) lets us try failed 
requests again in a smart way (retry). We'll start with the policy and policy
provider first.

You can check the full implementation in the [github repo](https://github.com/rickKoch/go-resilience)

## Policy and Policy Provider

The `Provider` and `Policy` work together to implement a __Factory Pattern__
with __Named Configuration__ for resilience patterns. 

The Provider acts as a centralized configuration factory that:
1. Stores Named Components
2. Configuration Management
3. Target-Based Policy Assembly

This pattern enables declarative resilience configuration where you define
"what" resilience patterns to apply rather than "how" to implement them, making
the system more maintainable and configurable.

```go
// The Policy struct is a central component that implements the Resilience Policy
// Pattern for executing operations with multiple fault tolerance mechanisms.
type Policy struct {
    timeout        time.Duration     // Timeout pattern
    retry          *retry            // Retry pattern  
    circuitBreaker *circuitBreaker   // Circuit Breaker pattern
}

// Centralized configuration factory
type Provider struct {
    timeouts        map[string]time.Duration     // Named timeout configurations
    retries         map[string]*retry            // Named retry strategies  
    circuitBreakers map[string]*circuitBreaker   // Named circuit breakers
    targets         map[string]target            // Named target configurations
}
```

### Execution Flow

The NewExecutor function creates an executor that applies resilience patterns in
a specific order:

1. Timeout (outermost layer)
2. Circuit Breaker (middle layer)
3. Retry (innermost layer)

```go
func NewExecutor(ctx context.Context, policy *Policy) Executor {
	if policy == nil {
		return func(oper Operation) (any, error) {
			return oper(ctx)
		}
	}

	return func(oper Operation) (any, error) {
		operation := oper

		if policy.timeout > 0 {
			operation = policy.withTimeout(operation)
		}

		if policy.circuitBreaker != nil {
			operation = policy.withCircuitBreaker(operation)
		}

		if policy.retry == nil {
			return operation(ctx)
		}

		return policy.withRetry(ctx, operation)
	}
}
```

This layering ensures that:
- Timeouts prevent operations from running indefinitely
- Circuit breakers can fail fast when services are unhealthy
- Retries happen within the circuit breaker's execution context

### Timeout pattern

The timeout pattern is used to stop a service from waiting too long for a
response from another service. If the reply takes too much time, the system
cancels the request and moves on. This helps the application fail quickly
instead of getting stuck, which keeps things running smoothly and prevents slow
services from affecting the whole system.

`withTimeout` applies the timeout pattern to an operation by using Go's
context.WithTimeout. It wraps a given operation so that if it takes longer than
a specified time limit (p.timeout), the operation is canceled.  It runs the
operation in a separate goroutine and waits for either the result or the timeout
to occur. If the operation finishes in time, the result is returned; if not, the
context times out and returns an error. It also safely handles panics inside the
operation to avoid crashing the program. This approach helps prevent slow or
stuck services from blocking the system, making it more responsive and
resilient.

```go
func (p *Policy) withTimeout(oper Operation) Operation {
	return func(ctx context.Context) (any, error) {
		// Context-Based Timeout Setup
		timeoutCtx, cancel := context.WithTimeout(ctx, p.timeout)
		defer cancel()

		resultCh := make(chan operationResult, 1)

		go func() {
			defer func() {
				if r := recover(); r != nil {
					select {
					case resultCh <- operationResult{nil, fmt.Errorf("operation panicked: %v", r)}:
					default:
					}
				}
			}()

			value, err := oper(timeoutCtx)

			select {
			case resultCh <- operationResult{value, err}:
			case <-timeoutCtx.Done():
			}
		}()

		select {
		case result := <-resultCh:
			return result.value, result.err
		case <-timeoutCtx.Done():
			return nil, timeoutCtx.Err()
		}
	}
}
```

The timeout pattern wraps an operation to ensure it doesn't run longer than a
specified duration, preventing resource exhaustion and improving system
responsiveness.

This `context.WithTimeout` usage creates a robust, standards-compliant timeout
mechanism that integrates seamlessly with Go's ecosystem and provides both
automatic timeout enforcement and cooperative cancellation opportunities.

### Retry Pattern

The retry pattern is useful when something fails for a short time, like a
network issue or a service being briefly unavailable. Instead of giving up right
away, the system waits a little and then tries again. This helps the application
recover from quick, temporary problems without causing bigger issues for users.

```go
func (r *retry) backoff(ctx context.Context) backoff.BackOff {
	var b backoff.BackOff = backoff.NewConstantBackOff(r.duration)

	if r.maxRetries >= 0 {
		b = backoff.WithMaxRetries(b, uint64(r.maxRetries))
	}

	return backoff.WithContext(b, ctx)
}

func OperationRetry(operation backoff.OperationWithData[any], b backoff.BackOff) (any, error) {
	return backoff.RetryWithData(func() (any, error) {
		return operation()
	}, b)
}
```

The OperationRetry function uses the __github.com/cenkalti/backoff__ library to
retry a failing operation according to a backoff strategy (e.g., constant
backoff). You give it an operation (one that can return a value and an error)
and a BackOff policy b. It calls backoff.RetryWithData, which will repeatedly
invoke your operation, waiting between attempts as defined by b, until it either
succeeds or the backoff policy gives up. The function then returns the last
result and error, so callers can see what eventually happened. This makes it
easy to wrap transient failures (like brief network glitches) in automatic,
controlled retries.

### Circuit Breaker

The circuit breaker pattern helps stop a service from constantly trying to reach
another service that isn't working. If something keeps failing, trying again and
again wastes resources and slows everything down. Like a real circuit breaker
that cuts off electricity to prevent damage, this pattern "opens" the connection
after a few failures, blocking more tries for a while. Once the problem is fixed
and the service is working again, the connection "closes" and normal
communication continues. This keeps the system running more smoothly and avoids
making things worse during failures.

This code builds and applies a circuit breaker using the sony/gobreaker library
to stop calling a failing service too often. The newCircuitBreaker function
reads settings (like how long to keep counts before resetting (Interval), how
long to stay open after tripping (Timeout), how many allowed trial requests in
half-open (MaxRequests), and how many consecutive failures trigger a trip) and
creates a breaker with a custom ReadyToTrip that opens when the consecutive
failure count reaches the configured threshold.
```go
func newCircuitBreaker(name string, config CircuitBreaker) (*circuitBreaker, error) {
	...
	cb.breaker = gobreaker.NewCircuitBreaker(gobreaker.Settings{
		Name:        name,
		MaxRequests: maxRequest,
		Interval:    interval,
		Timeout:     timeout,
		ReadyToTrip: tripFn,
	})

	return cb, nil
}
```

`withCircuitBreaker` wrapper then runs an operation through that breaker: if the
breaker is open or the call fails, the failure is returned; if retries are also
configured and the error is considered permanent, it's marked so the retry logic
won't keep retrying. This setup prevents repeated calls to an unhealthy
downstream service while allowing controlled recovery checks and integrating
with retry policies.
```go
func (p *Policy) withCircuitBreaker(oper Operation) Operation {
	return func(ctx context.Context) (any, error) {
		res, err := p.circuitBreaker.breaker.Execute(func() (any, error) {
			return oper(ctx)
		})

		if p.retry != nil && IsErrorPermanent(err) {
			err = backoff.Permanent(err)
		}

		return res, err
	}
}
```

### Conclusion

Resilience patterns are strategies used in microservices to keep systems stable
and reliable, even when some parts fail. Three common patterns are timeout,
retry, and circuit breaker. The timeout pattern sets a limit on how long a
service waits for a response, helping the system avoid getting stuck. The retry
pattern handles temporary failures by trying the operation again after a short
pause. The circuit breaker pattern protects the system by stopping calls to a
service that keeps failing and only allowing requests again once it recovers.
Together, these patterns help applications stay responsive and fault-tolerant in
the face of errors or high demand.

### Resources

* ["gobreaker library"](https://github.com/sony/gobreaker)    
* ["backoff library"](https://github.com/cenkalti/backoff)    
* ["Context package"](https://pkg.go.dev/context)