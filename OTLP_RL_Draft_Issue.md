# Issue Title: Proposal: Reinforcement Learning (RL) Semantic Conventions (`rl.*` namespace)

**Labels:** `area:semantic-conventions`, `status:needs-sponsor`, `sig:genai`

## Context: Post-Training Observability

Reinforcement Learning (RL) has moved from research labs to large-scale production. It now plays a central role in robotics, autonomous agents, and notably, **Large Language Model (LLM) Post-Training** (e.g., RLHF, GRPO, PPO). 

Currently, OpenTelemetry has active discussions in the GenAI SIG for LLM inference, but there is **no standardized convention for LLM Post-Training or general RL workloads**. This leads to fragmented, ad-hoc metrics across different RL frameworks, making it impossible to build standardized observability tooling for GenAI scaling pipelines.

## Proposal

We propose establishing a new `rl.*` namespace to standardize metrics, spans, and attributes for Reinforcement Learning training systems. Modeled after modern RLHF lifecycles, we propose dividing the namespace into distinct execution phases:

### High-Level Summary of Proposed Namespace:

* **Resource Attributes:** `rl.system`, `rl.run.id`, `rl.algorithm`, `rl.environment.name`, `rl.model.name`, `rl.environment.type`, `rl.agent.count`
* **Metrics (Durations, Counters, Gauges):**
  * `rl.loop.*`: The outer holistic RL training iteration (e.g., `rl.loop.duration`).
  * `rl.sample.*`: Actor phase generating rollouts (e.g., `rl.sample.duration`, `rl.sample.samples`).
  * `rl.reward.*`: Reward model scoring phase (e.g., `rl.reward.duration`, `rl.reward.score`).
  * `rl.train.*`: Learner update phase (e.g., `rl.train.duration`, `rl.train.loss` (Gauge), `rl.train.mfu`).
  * `rl.environment.*`: Evaluation signals (e.g., `rl.environment.reward.mean`).
  * `rl.sync.*`: Weight synchronization (e.g., `rl.sync.duration`).
* **Spans:** `rl.loop`, `rl.sample`, `rl.reward`, `rl.train`, `rl.sync`
  * *Hierarchy:* We propose **`rl.loop`** as the Root Span. Standardizing this hierarchy facilitates cross-process trace propagation (e.g., passing the `traceparent` during remote rollout requests), allowing a single training iteration to be visualized as a unified trace across distributed actor-learner nodes.

### Why a top-level `rl.*` namespace?
The OTel community rightfully scrutinizes new top-level namespaces. We strongly advocate for `rl.*` rather than nesting under `ml.*` due to the fundamentally unique architecture of RL:
1. **The Synchronous Nature:** Unlike standard ML training (which is often a continuous asynchronous stream of data), RL is a tightly coupled loop where data generation (sampling) strictly depends on the latest model weights from the previous training step. Standard ML metrics don't capture this circular dependency well.
2. **Multi-Process Coordination:** RL involves distinct "Actors" and "Learners" on different compute nodes. Standardizing the `rl.run.id` and tracing the parameters across these boundaries is critical for joining spans across these highly distributed components.

## Viability & Prior Art

**Prior Art & Auto-Instrumentation:** Current RL frameworks (Ray RLlib, NeMo-RL, veRL) lack a unified telemetry interface, forcing users to build bespoke collectors for every new project. This proposal provides the Semantic Foundation required to develop an `opentelemetry-instrumentation-rl` package. Such a package would allow engineers to swap frameworks (e.g., moving a project from Ray to NeMo) without rewriting their Grafana dashboards or Prometheus alerts.

**Validation:** We have validated this schema by mapping it to the internal telemetry of several major open-source RL frameworks (including NeMo-RL, Ray RLlib, veRL, Tunix, and SKY RL). Specifically, we ensured that the `rl.sync` and `rl.sample` boundaries correctly capture the "dead time" often hidden in distributed RL training, and that hardware metrics like Model Flops Utilization (MFU) are natively supported.

## Next Steps & Sponsorship

We have already drafted the formal OpenTelemetry Schema YAML definitions for this namespace (`attributes.yaml`, `metrics.yaml`, `spans.yaml`). 

* **Sponsorship:** We are looking for sponsors within the **GenAI SIG** who would be interested in supporting this area, given its heavy intersection with LLM post-training and alignment workloads.
* **Review Request:** We'd love feedback from the community on the proposed subsystem boundaries (`sample`, `reward`, `train`, `sync`).

Once we establish interest, we will submit a Pull Request containing the formal YAML definitions to the `model/` directory.
