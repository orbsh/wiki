# Concurrency Control and Backpressure Design

> **Languages:** [English](congestion-control-design-en.md) (primary) · [中文](congestion-control-design.md)

Concurrency control design for the Embedding generation scenario (SurrealDB making pass-through calls to Ollama). Core principle: concurrency control is the **caller's responsibility** (end-to-end principle), not the DB layer's.

## Signal Chain Architecture

```
                    ┌──────────────────────────────────────────────────────────────────┐
                    │                       Ollama (bottleneck)                        │
                    └──────────┬───────────────────────────────────────────────────────┘
                               │ overload → rising latency / timeout
                    ┌──────────▼───────────────────────────────────────────────────────┐
                    │                  SurrealDB (pass-through layer)                  │
                    │                http::post() returns result/error                 │
                    └──────────┬───────────────────────────────────────────────────────┘
                               │ latency + error code
                    ┌──────────▼───────────────────────────────────────────────────────┐
                    │                          Python caller                           │
                    │                                                                  │
                    │ Signal collection ──→ Decision engine ──→ Concurrency adjustment │
                    │        (backpressure-aware)   (AIMD)    (execution layer)        │
                    └──────────────────────────────────────────────────────────────────┘
```

## Signal Layer (Backpressure-Aware)

| Signal | Meaning | Action |
|:---|:---|:---|
| Response latency < target threshold T | Ollama idle | Allow additive increase |
| Response latency >= target threshold T | Ollama near saturation | Pause growth, hold current concurrency (soft backpressure) |
| Timeout / 5xx | Ollama overloaded | Trigger multiplicative decrease (hard backpressure) |
| K consecutive timeouts | Ollama down | Enter cooldown period (circuit breaker open) |

**Soft backpressure vs hard backpressure**: latency exceeding the threshold without timing out = a soft signal, telling the caller "don't add more" without punishing it. This is smoother than waiting for a timeout to react, and avoids oscillation.

## Decision Layer (AIMD)

Concurrency degree C, target latency T (e.g. 200ms):

```
On each request return:
  if response success && latency < T:
      C += 1                    # additive increase: slowly probe the ceiling
  elif response success && latency >= T:
      pass                     # soft backpressure: stop growing, but do not punish
  elif timeout or 5xx:
      C = max(C * 0.5, 1)     # multiplicative decrease: back off quickly
  elif K consecutive failures:
      C = 1; enter cooldown period T_cool  # Circuit breaker open
```

**Why the combination beats a single layer**:

| Approach | Flaw |
|:---|:---|
| Congestion control only, no backpressure | Overload is learned only at timeout; slow to react |
| Backpressure only, no congestion control | Overload is known, but not by how much to back off; either backs off too much (waste) or too little (oscillation) |
| Combination | Smooth adjustment via soft signals (latency) + fast back-off via hard signals (errors) + cooldown period to prevent oscillation |

## Pseudocode

```python
class EmbeddingCongestionController:
    def __init__(self, initial=5, target_latency_ms=200, cooldown_sec=10, failure_threshold=3):
        self.concurrency = initial
        self.target = target_latency_ms
        self.cooldown = cooldown_sec
        self.failure_threshold = failure_threshold
        self.consecutive_failures = 0
        self.next_probe_time = 0

    def acquire(self) -> bool:
        """Whether a new request may be sent"""
        if time.now() < self.next_probe_time:
            return False  # reject during the cooldown period
        return True

    def on_response(self, latency_ms, success):
        """Called after each request returns"""
        if not success or latency_ms is None:
            self.consecutive_failures += 1
            self.concurrency = max(self.concurrency * 0.5, 1)
            if self.consecutive_failures >= self.failure_threshold:
                self.next_probe_time = time.now() + self.cooldown
                self.consecutive_failures = 0
        else:
            self.consecutive_failures = 0
            if latency_ms < self.target:
                self.concurrency += 1
            # latency >= target: neither increase nor decrease (soft backpressure signal)
```

## SurrealDB's Monitoring Interface

The monitoring capabilities SurrealDB exposes are more direct than instrumenting Python yourself:

| Interface | Purpose | Advantage |
|:---|:---|:---|
| `/metrics` | Prometheus-format metrics (query count, latency, memory) | Standard protocol, consumed directly by Grafana |
| `info()` system table | Real-time query count, memory usage, uptime | No extra dependency, queryable directly via SQL |

**Key insight**: Python instrumentation observes the latency of the "Python → SurrealDB" segment, but the bottleneck is at "SurrealDB → Ollama". Reading SurrealDB's `/metrics` directly observes end-to-end latency, giving a more accurate signal.

## The Essence of Backpressure

Let the overload signal of the bottleneck node (Ollama) propagate backward along the call chain. The signal chain is `Ollama overloaded → SurrealDB's http::post() times out → Python receives an error → backs off`. Although Python does not communicate with Ollama directly, because `fn::ollama::embed()` is a pass-through call (no extra processing), SurrealDB's response latency and errors directly reflect Ollama's state, with minimal signal distortion.

Doing backpressure at the DB layer instead introduces oscillation: `Ollama overloaded → SurrealDB detects → SurrealDB rejects → Python retries → SurrealDB calls Ollama again → blows up`. That is one extra round trip, and the retry request hits the bottleneck node again. Doing backpressure on the Python side completes the back-off in a single round trip.

## Cross-References

- **[Unified Data Layer](unified-data-layer-en.md)**: the SurrealDB compute pushdown architecture and the Embedding generation case; this document expands on its concurrency control details.
