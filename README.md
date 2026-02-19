Got it. Based on your recent work (P-361 two-bound controller, VRAM-stable pipeline, commit-true ingestion, adaptive FEED control), here’s a concise, professional README you can drop into a repo or attach to a LinkedIn/GitHub post.
VRAM-Stable Adaptive GPU Feed Controller (P-361)
Overview
P-361 is a two-bound adaptive GPU launch controller designed to stabilize VRAM utilization and eliminate starvation/overflow oscillations in high-throughput GPU pipelines. It integrates starvation feedback and VRAM occupancy into a hysteresis-controlled launch threshold that maintains continuous compute while preserving exactly-once ingestion guarantees.
This work builds on a commit-true, manifest-driven pipeline architecture with staged lanes, pinned queues, and event-gated transfers.
Key Contributions
Two-bound control loop: Joint regulation of starvation (s) and VRAM fraction (vram_frac)
Quantized launch threshold: Stable, monotonic VRAM response to threshold tuning
Hysteresis + cooldown: Prevents flip-flop retuning and launch jitter
Commit-true mutation boundary: Controller/state updates only on safe rewrite windows
Exactly-once preservation: Compatible with manifest-only ingestion and atomic claim
VRAM stabilization prerequisite: Enabled by prior memory monotonicity fixes
Control Model
Inputs per window:
s — starvation metric (idle ratio / feed deficit)
vram_frac — allocated VRAM / device VRAM
cd — cooldown counter
lt — launch threshold (bytes)
Decision:
If s high → FEED_GPU (decrease lt)
If vram_frac high → DRAIN (increase lt)
Else → HOLD
Retune occurs only when hysteresis bands are crossed and cooldown expired.
Telemetry
Canonical retune record:
Copy code

[P361Δ] w=<window> dir=<FEED|DRAIN|HOLD> retuned=<0|1> 
cd=<cooldown> s=<starvation> vram=<fraction> lt=<old->new>
This enables offline stability analysis (flip-rate, dwell, boundary error).
Integration Context
Validated within a VRAM-stable GPU pipeline featuring:
Manifest-only ingestion + atomic claim
Pinned host staging queues (2/4/6 ladder)
Event-gated GPU→CPU transfers
Commit-true writes (tmp→fsync→rename→fsync dir)
Rewrite boundary at window-7 (coprime with probe cadences 3/6)
These invariants ensure VRAM usage is monotonic in lt, making feedback control well-posed.
Results (Pipeline-level)
Near-critical damping of feed oscillations
~50% reduction in FEED↔HOLD transitions
Stable VRAM operating band
Sustained GPU occupancy without spill or starvation
Status
Controller stabilized and validated in production-like workload replay.
Ready for pilot integration in high-throughput GPU ingestion or training pipelines.
