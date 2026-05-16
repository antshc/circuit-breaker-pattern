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
- Duration controlled by `BreakDuration` or `BreakDurationGenerator`.

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

- Fixed time the circuit stays open.
- Matching requests fail fast during this period.
- After this period, the circuit moves to half-open probing.

### `BreakDurationGenerator`

- Generates the open duration dynamically.
- Can be used for fixed or exponential break duration.
- Useful when the break duration must depend on failure context.
- Prefer fixed duration for predictable recovery probes.
- Use exponential duration when repeated breaks should wait longer before the next probe.

Fixed break duration:

```csharp
BreakDurationGenerator = static _ =>
    new ValueTask<TimeSpan>(TimeSpan.FromSeconds(30));
```

Exponential break duration:

```csharp
BreakDurationGenerator = static args =>
{
    var seconds = Math.Min(60, Math.Pow(2, args.FailureCount));
    return new ValueTask<TimeSpan>(TimeSpan.FromSeconds(seconds));
};
```

Example exponential sequence:

```text
Break #1: 2s
Break #2: 4s
Break #3: 8s
Break #4: 16s
Break #5+: capped at 60s
```

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

Recommended strategy:

```text
Retry -> Circuit Breaker -> Dependency
```

- Use fixed `BreakDuration` on the circuit breaker.
- Use exponential backoff on retries.
- Fixed break duration provides predictable recovery probes.
- Circuit breaker quickly checks external system availability after a known delay.
- Exponential retry backoff prevents retry storms and excessive traffic spikes during dependency recovery.
- Reduces wasted requests during outages.
- Keeps operation alive while protecting the dependency.

## Thundering herd prevention

- Without exponential retry:
  - Probe succeeds.
  - Thundering herd floods the dependency.
  - Azure re-throttles requests.
  - Circuit oscillates between open and half-open states.

- With exponential retry:
  - Probe succeeds.
  - Thundering herd is spread over time.
  - Traffic ramps gradually:
    - 50 → 120 → 200 → 300
  - Dependency recovers without re-throttling.
  - Circuit stability improves under concurrency.

## Notes

- Circuit breaker limits failure impact.
- Circuit breaker does not fix the dependency.
- Shared breaker is recommended for parallel workloads.
- Combine with timeout, retry, backoff, jitter, telemetry, and backpressure.
