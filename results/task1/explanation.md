# Task 1: Stride Pattern Prefetcher (SPP)

## 1) IPC, Speedup, L1D Load MPKI, Prefetch Accuracy, and Coverage

The table below reports the values obtained for the three traces.

| Trace | IPC with Prefetcher | Speedup | L1D Load MPKI | Prefetch Accuracy | Coverage |
|---|---:|---:|---:|---:|---:|
| Trace1.gz | 0.233905 | 1.00772x | 9.67296 | 62.0624% | 9.42% |
| Trace2.gz | 0.252287 | 1.00922x | 13.0896 | 56.0276% | 3.31% |
| Trace3.gz | 0.384573 | 1.00708x | 3.108 | 34.234% | 8.68% |

The corresponding plots are stored in [results/task1/plots](plots):

- speedup.png
- l1d_mpki.png
- accuracy.png
- coverage.png

## 2) Storage overhead of the prefetcher

The prefetcher uses two main tables:

### IP tracker table
Each entry of `IP_Tracker` contains:

- `ip` = 64 bits
- `last_addr` = 64 bits
- `last_stride` = 64 bits
- `uniform_conf` = 32 bits
- `signature` = 32 bits
- `lru_cycle` = 64 bits

Per-entry size:

- 64 + 64 + 64 + 32 + 32 + 64 = 320 bits

There are 128 entries:

- 128 × 320 = 40,960 bits

### Pattern table
Each entry of `Pattern_Entry` contains:

- `signature` = 32 bits
- `next_stride` = 64 bits
- `lru_cycle` = 64 bits

Per-entry size:

- 32 + 64 + 64 = 160 bits

There are 256 entries:

- 256 × 160 = 40,960 bits

### Total storage overhead

- IP tracker table: 40,960 bits
- Pattern table: 40,960 bits
- Prefetch tracking array: 64 × 64 = 4,096 bits

Total per core:

- 40,960 + 40,960 + 4,096 = 86,016 bits

This is the storage used by the prefetcher metadata tables.

## 3) Threshold for accuracy throttling

We chose the accuracy thresholds to be 0.75 and 0.40. The prefetcher increases its degree only when more than 75% of recent prefetches are useful, and it backs off when usefulness drops below 40%. Keeping a gap between these two points makes the control more stable, so the prefetcher does not keep flipping between aggressive and conservative behavior.

These values were picked after trying different cutoffs and comparing IPC, L1D MPKI, and prefetch usefulness across the traces. The final pair gave the best balance between better coverage and lower cache pollution. It lets the prefetcher stay aggressive when the pattern is working but back off quickly when it starts generating useless fills.

## 4) Why these prefetch degrees were chosen

The prefetcher starts with a degree of 2 and then adjusts itself by one step at a time based on how useful the recent prefetches actually are.

- If accuracy is above 75%, the degree is increased by 1 to improve coverage.
- If accuracy falls below 40%, the degree is decreased by 1 to avoid useless prefetches.

This starting value was chosen because it is a good middle ground: it is high enough to help when the access pattern is regular, but not so aggressive that it pollutes the cache on noisy or irregular streams. Uniform stride patterns can benefit from a slightly higher degree, while non-uniform or unstable patterns are better handled with a smaller degree. So the degree of 2 keeps the design conservative at first, and the accuracy-based throttling then decides whether to increase or decrease the aggressiveness by exactly 1 step at a time, with the degree bounded between 1 and 4.
