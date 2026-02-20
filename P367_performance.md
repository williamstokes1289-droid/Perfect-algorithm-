# P-367 Performance Characteristics

## Summary

P-367 improves sustained GPU utilization and throughput by enforcing shard-quantized,
legality-aware launch admissibility. Launches occur only when contiguous pre-gap payload
meets a 128 MiB lattice threshold, eliminating sub-batch launches and launch jitter.

---

## Launch Model

Q = 128 MiB  
lt_q ∈ { k·Q | k ≥ 1 }

ready_total = ready_legal + illegal  
ready_legal = contiguous pre-gap payload  

launch_allowed =
  (ready_legal ≥ lt_q) ∧
  cooldown_ok ∧
  (inflight < max_inflight) ∧
  ¬mutations_blocked

---

## Performance Effects

### 1. Shard-aligned batch sizing
Launches occur only at ≥128 MiB boundaries.

Effect:
- larger effective batches
- higher kernel occupancy
- improved memory throughput
- fewer launches per GB

---

### 2. Elimination of sub-shard launches
Partial batches (<128 MiB) cannot dispatch.

Effect:
- reduced launch overhead
- reduced scheduler churn
- improved SM fill consistency
- better PCIe/NVLink efficiency

---

### 3. Deterministic threshold crossings
Launch eligibility changes only when ready_legal crosses a 128 MiB boundary.

Effect:
- stable launch cadence
- reduced flip-flop behavior
- smoother GPU feed
- improved sustained utilization

---

### 4. Legality-aware dispatch
ready_legal excludes held or discontinuous payload.

Effect:
- no launches on gap-blocked regions
- no wasted compute on invalid payload
- consistent effective work per launch

---

### 5. Monotonic VRAM response vs lt_q
Shard quantization produces predictable VRAM scaling with threshold.

Effect:
- stable adaptive tuning
- safe threshold increases
- higher steady-state batch size
- improved throughput ceiling

---

## Example Transition

Gap held:

ready_total = 210  
ready_legal = 82  
illegal = 128  
lt_q = 128  

82 ≥ 128 → launch_allowed = False  

Gap resolved:

ready_legal = 210  
210 ≥ 128 → launch_allowed = True  

Result: shard-aligned launch fires immediately after legality restoration.

---

## Performance Interpretation

P-367 converts GPU feeding from irregular, sub-batch launches into
shard-aligned deterministic launches governed by legality thresholds.

Net effect:

- higher sustained utilization  
- lower launch overhead  
- smoother dispatch cadence  
- improved throughput stability  

---

## Bottom Line

P-367 increases GPU throughput and utilization by enforcing shard-quantized,
legality-aware launch thresholds that eliminate sub-batch overhead and launch jitter.
