# Polly Circuit Breaker

## Configuration

```csharp
CircuitBreakerStrategyOptions
{
    FailureRatio = 0.5,
    SamplingDuration = TimeSpan.FromSeconds(10),
    MinimumThroughput = 8,
    BreakDuration = TimeSpan.FromSeconds(30),
    ShouldHandle = new PredicateBuilder().Handle<SomeExceptionType>()
};
```

## Purpose

- Prevent cascading failures.
- Stop traffic to unhealthy dependencies.
- Fail fast during outages or throttling.
- Reduce retry amplification.
- Reduce wasted I/O.
- Improve dependency recovery stability.

## States

### Closed

- Normal operation.
- Requests allowed.
- Failures monitored.

### Open

- Requests rejected immediately.
- No dependency call executed.
- Duration controlled by `BreakDuration`.

### Half-Open

- Limited probe requests allowed.
- Only one request is sent as a probe while other requests are rejected.
- Successful probe closes the circuit.
- Failed probe reopens the circuit.

## Options

### `FailureRatio = 0.5`

- Opens the circuit when failed requests are at least 50% of total requests.
- Evaluated only when `MinimumThroughput` is reached.

Example:

- 10 requests, 5 failures: may open.
- 10 requests, 4 failures: stays closed.

### `SamplingDuration = TimeSpan.FromSeconds(10)`

- Sliding evaluation window.
- Only requests from the last 10 seconds are counted.
- Older failures are ignored.

### `MinimumThroughput = 8`

- Minimum request count required before evaluation.
- Prevents opening from low-traffic noise.

Example:

- 3 requests, 3 failures: ignored.
- 8 requests, 5 failures: evaluated.

### `BreakDuration = TimeSpan.FromSeconds(30)`

- Time the circuit stays open.
- Matching requests fail fast during this period.
- After this period, the circuit moves to half-open probing.

### `ShouldHandle`

```csharp
ShouldHandle = new PredicateBuilder().Handle<SomeExceptionType>()
```

- Defines which failures are counted by the breaker.
- Only matching exceptions or results affect breaker state.

Typical handled failures:

- Timeouts.
- HTTP 429.
- HTTP 5xx.
- Network failures.

Usually ignored:

- Validation errors.
- Business rule exceptions.
- Client misuse.

## Evaluation Logic

The circuit opens when:

```text
Failures / TotalRequests >= FailureRatio
AND
TotalRequests >= MinimumThroughput
WITHIN SamplingDuration
```

Example:

```text
SamplingDuration: 10s
MinimumThroughput: 8
FailureRatio: 0.5
Requests: 10
Failures: 6
Result: circuit opens
```

## Circuit breaker Block mode (keep operation alive)

Recommended order:

```text
Retry -> Circuit Breaker -> Dependency
```

- Retry handles transient failures.
- Circuit breaker handles sustained failures.
- Open circuit prevents retry storms.
- Backoff and jitter reduce synchronized retry spikes.

## Key Metrics

Track:

- Open count.
- Failure ratio.
- Rejected requests.
- Half-open probe success rate.
- Half-open probe failure rate.
- Retry count.
- Dependency latency.
- HTTP 429 rate.
- HTTP 5xx rate.

## Notes

- Circuit breaker limits failure impact.
- Circuit breaker does not fix the dependency.
- Shared breaker is recommended for parallel workloads.
- Combine with timeout, retry, backoff, jitter, telemetry, and backpressure.
