# System Designs

Six system designs by [Gautam Patil](https://github.com/Gautam-R-Patil): requirements and architectures for AI products. Each folder holds one design, imported with its original commit history.

These are **design specifications, not implementations.**

| Design | Folder | What it is |
|---|---|---|
| **Veltrix** | [`Veltrix/`](Veltrix/) | Energy-fingerprint security: detecting model theft, crypto-mining and unauthorised GPU/CPU workloads from power signatures |
| **Watson** | [`Watson/`](Watson/) | Deterministic neuro-symbolic forensic reconstruction with physics-based ballistics, evidence hashing, replay and immutable audit trails |
| **NeuroPilot** | [`NeuroPilot/`](NeuroPilot/) | Cross-platform developer-workflow copilot: planner → validator → guardrail → executor, with explicit confirmation for risky actions |
| **Vacay** | [`Vacay/`](Vacay/) | Autonomous travel concierge that derives taste profiles from photos, composes itineraries and plans agent-driven booking |
| **Synaptek** | [`Synaptek/`](Synaptek/) | Adaptive ICU early warning: multimodal time series, LSTMs, risk scoring, alert-fatigue controls and explainability |
| **Axon** | [`Axon/`](Axon/) | Neuro-symbolic structural inspection: drone RGB and LiDAR perception fused with deterministic engineering rules and digital twins |

## Common threads

- **Safety by construction:** guardrails, confirmation gates and deterministic checks around learned components.
- **Evidence and auditability:** hashing, replay and immutable trails, the same ideas behind [SOVA OSS](https://github.com/TRUSCOR/SOVA-OSS).
- **Neuro-symbolic design:** learned perception combined with explicit rules where correctness matters.
