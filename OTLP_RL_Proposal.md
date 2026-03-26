# OpenTelemetry Semantic Conventions for Reinforcement Learning (RL)

**Target SIG:** SIG GenAI
**Status:** Draft / Experimental

---

## 1. Motivation

Reinforcement Learning (RL) is rapidly becoming a core component of production software systems, scaling across robotics, autonomous systems, and notably, large language model (LLM) post-training (e.g., RLHF, GRPO, PPO). These systems are highly distributed, multi-stage pipelines that exhibit unique failure modes—such as policy collapse, reward model hacking, and synchronization lag—that possess no equivalent in traditional request-response services.

While OpenTelemetry is actively developing conventions for generative AI (GenAI SIG), there are currently **no standard conventions for RL workloads.** Without standardized conventions, telemetry cannot be federated or compared across frameworks like NeMo-RL, veRL, and Ray RLlib.

This proposal seeks to establish a new `rl.*` namespace to standardize telemetry for RL training systems. We advocate for a top-level `rl.*` namespace rather than `ml.*` because RL demands a fundamentally unique architecture:
1. **The Synchronous Nature:** Unlike standard ML training (which is often a continuous asynchronous stream of data), RL is a tightly coupled loop where data generation (sampling) strictly depends on the latest model weights from the previous training step. Standard ML metrics don't capture this circular dependency well.
2. **Multi-Process Coordination:** RL involves distinct "Actors" and "Learners" on different compute nodes. Standardizing the `rl.run.id` and tracing parameters across these boundaries is critical for joining spans.

## 2. Evidence of Viability

A core requirement for standardizing OpenTelemetry semantic conventions is proving that they are viable in operational code. 

**Prior Art & Auto-Instrumentation:** Current RL frameworks (Ray RLlib, NeMo-RL, veRL) lack a unified telemetry interface, forcing users to build bespoke collectors for every new project. This proposal provides the Semantic Foundation required to develop an `opentelemetry-instrumentation-rl` package. Such a package would allow engineers to swap frameworks without rewriting their Grafana dashboards or Prometheus alerts.

**Validation:** We have successfully validated these proposed conventions internally by prototyping the OTel instrumentation across several major RL frameworks (including NeMo-RL, Ray RLlib, veRL, Tunix, and SKY RL). We verified that standard Cloud Monitoring and Grafana backends can seamlessly consume these metrics to reliably track golden signals. 

## 3. Scope and Subsystems

Based on the architecture of modern distributed RL, the `rl.*` namespace is divided into distinct execution phases:
* `rl.loop`: The outer holistic RL training iteration.
* `rl.sample`: The actor phase (generating rollouts/responses).
* `rl.reward`: The environment / reward model scoring phase.
* `rl.train`: The learner phase (policy gradient updates).
* `rl.sync`: The parameter/weight synchronization phase between actors and learners.

## 4. Proposed Namespace & Attributes

*(The exact technical specifications for attributes, metrics, and spans are defined in the accompanying OpenTelemetry Weaver YAML files in `model/rl/`)*. 

### 4.1 Resource Attributes

* `rl.system`, `rl.system.version`
* `rl.run.id`, `rl.algorithm`, `rl.model.name`
* `rl.environment.name`, `rl.environment.type`
* `rl.agent.count`

### 4.2 Signal-Scoped Context Attributes (Span / Event Attributes)

* `rl.loop.iteration`
* `rl.sample.batch_size`, `rl.train.batch_size`
* `rl.reward.sandbox`
* `rl.sync.source`, `rl.sync.destination`, `rl.sync.bytes`

## 5. Metric Boundaries 

Following OTel conventions, time is tracked as Histograms with the `.duration` suffix.

### 5.1 Durations & Latencies (Histograms)
* `rl.loop.duration`: Total time for one complete RL iteration.
* `rl.sample.duration`: Time spent generating rollouts.
* `rl.reward.duration`: Time spent running the reward model or environment step.
* `rl.train.duration`: Time spent executing learner updates.
* `rl.sync.duration`: Time spent synchronizing weights across the cluster.
* `rl.step.duration`: Time spent in an individual generic step.

### 5.2 Throughput & Traffic (Counters & Gauges)
* `rl.sample.samples`, `rl.sample.episodes`, `rl.train.steps`, `rl.train.tokens`

### 5.3 Health & Efficiency (Gauges)
* `rl.train.loss`: Training loss value (Gauge). Differentiated by the `rl.loss.type` attribute (e.g., `policy`, `value`, `kl_divergence`).
* `rl.train.mfu`: Model Flops Utilization.
* `rl.environment.reward.mean`: Moving average of evaluation rewards.
* `rl.environment.episode.length.mean`: Average steps per episode during evaluation.

## 6. Alignment with OTel Guidelines

1. **Duration Naming:** We strictly use the `.duration` suffix for histograms instead of `.latency`.
2. **Subsystem Granularity:** The `sample`, `reward`, `train`, and `sync` abstractions perfectly map to distributed tracing spans. Specifically, formalizing the `rl.sync` boundary facilitates **cross-process trace propagation** (passing the `traceparent` payload during weight broadcast), addressing one of the hardest observability pain points in distributed actor-learner architectures.
3. **Alignment with GenAI SIG:** Tracking token throughput, MFU, and distinct reward modeling phases is essential to modern GenAI post-training (RLHF).

## 7. Distributed Trace Hierarchy

To ensure coherent observability in distributed RL training, the following span hierarchy is recommended. The `rl.loop` span serves as the root boundary for a single iteration, ensuring that all sub-operations can be visualized as part of a single coherent trace.

### 7.1 Span Relationships

| Span | Parent | Location | Context Strategy |
| :--- | :--- | :--- | :--- |
| `rl.loop` | Root / None | Learner (Orchestrator) | Root span start. |
| `rl.sample` | `rl.loop` | Actors (Distributed) | **`traceparent`** propagated via RPC headers. |
| `rl.reward` | `rl.loop` | Actors / Reward Node | Child of loop (batch scored or per rollout). |
| `rl.train` | `rl.loop` | Learner (Learner) | Child of loop (master node). |
| `rl.sync` | `rl.loop` | Learner -> Actors | Child of loop (weight broadcast). |

### 7.2 Context Propagation Diagram

```mermaid
sequenceDiagram
    participant Learner
    participant Actor
    Note over Learner: Start rl.loop
    Learner->>Actor: Request Rollout (Injected traceparent)
    Note over Actor: Start rl.sample (Parent: rl.loop)
    Actor->>Actor: Start rl.reward (Parent: rl.loop)
    Actor-->>Learner: Return samples/tokens
    Note over Actor: End rl.reward
    Note over Actor: End rl.sample
    Note over Learner: Start rl.train (Parent: rl.loop)
    Note over Learner: Start rl.sync (Parent: rl.loop)
    Note over Learner: End rl.sync
    Note over Learner: End rl.train
    Note over Learner: End rl.loop
```

Validating this hierarchy ensures that engineers can identify exactly where "dead time" exists in the loop (e.g., if `rl.loop` is active but neither `rl.sample` nor `rl.train` are running, the system is likely stalled on networking or I/O).
