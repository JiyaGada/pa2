# Task 2: I-RRIP and Dynamic I-RRIP

## Task 2A: I-RRIP

I-RRIP gives instruction lines a longer expected lifetime in L2. New instruction lines enter at RRPV 1, and data lines enter at RRPV 2. On a hit, either kind of line is promoted to RRPV 0. When choosing a victim, the policy first evicts a data line at the maximum RRPV. If none is available, it evicts an instruction line at the maximum RRPV. If no line has the maximum RRPV, the RRPVs age until a victim is available.

With maximum RRPV 3, instruction insertion at 1 gives an instruction line one extra reuse opportunity compared with the usual RRIP insertion at 2. Data stays at 2 so it remains easier to evict when both classes are old. A hit promotes either class to 0 because recent use is evidence that either line may be reused. At eviction, preferring data at maximum RRPV protects an equally old instruction line when possible. On Trace2, this lowers instruction MPKI from 18.7638 to 15.0475, while data MPKI rises from 8.0330 to 8.9334; the other traces show smaller IPC changes. This is the intended tradeoff: preserve useful code without assuming that data has no value.

The table shows RRIP, I-RRIP, and the selected DIRRIP configuration. Speedup is IPC divided by RRIP IPC for the same trace. L2 load MPKI is also split into instruction and data loads. Complete measurements, including the sensitivity sweep and insertion-position comparisons, are in `data.csv`.

| Trace | Policy | IPC | Speedup | L2 load MPKI | L2 instruction MPKI | L2 data MPKI |
|---|---|---:|---:|---:|---:|---:|
| Trace1 | RRIP | 0.233047 | 1.0000 | 17.1922 | 10.6175 | 6.5747 |
| Trace1 | I-RRIP | 0.233096 | 1.0002 | 17.0757 | 10.0885 | 6.9872 |
| Trace1 | DIRRIP | 0.233217 | 1.0007 | 17.0432 | 10.0666 | 6.9766 |
| Trace2 | RRIP | 0.250908 | 1.0000 | 26.7968 | 18.7638 | 8.0330 |
| Trace2 | I-RRIP | 0.255003 | 1.0163 | 23.9808 | 15.0475 | 8.9334 |
| Trace2 | DIRRIP | 0.254821 | 1.0156 | 24.0985 | 15.2079 | 8.8906 |
| Trace3 | RRIP | 0.382430 | 1.0000 | 3.8467 | 1.2097 | 2.6370 |
| Trace3 | I-RRIP | 0.382608 | 1.0005 | 3.7527 | 1.0845 | 2.6682 |
| Trace3 | DIRRIP | 0.382580 | 1.0004 | 3.7563 | 1.0895 | 2.6668 |

The results show that I-RRIP lowers instruction-load MPKI on all three traces. The data-load MPKI rises slightly, and IPC gains vary by trace. Thus the policy helps most when the trace benefits from retaining instruction lines.

## Task 2B: Dynamic I-RRIP

DIRRIP uses the first 32 L2 sets as RRIP leader sets. They use RRIP for victim selection and insertion, including inserting instruction lines at RRPV 2. The policy counts instruction and data accesses and misses in those sets. This uses a small sample to guide the other sets, though using only the first 32 assumes their traffic represents the cache. For the other sets, it calculates:

`(instruction accesses + instruction misses) / (all accesses + all misses)`

If this value is greater than 0.25, it uses I-RRIP; otherwise, it uses RRIP. This follows the assignment's suggested decision rule. The 32-set choice gives the selector a sample of accesses while limiting the cache sets reserved for the RRIP leaders.

I tested thresholds 0.15, 0.20, 0.25, 0.35, and 0.55 with 8, 32, and 64 leader sets. The table gives the geometric-mean speedup across the three traces for each setting.

| Threshold | 8 leaders | 32 leaders | 64 leaders |
|---:|---:|---:|---:|
| 0.15 | 1.00549x | 1.00548x | 1.00541x |
| 0.20 | 1.00544x | 1.00555x | 1.00547x |
| 0.25 | 1.00553x | 1.00555x | 1.00543x |
| 0.35 | 1.00501x | 1.00529x | 1.00492x |
| 0.55 | 1.00012x | 1.00021x | 0.99996x |

The 0.25 threshold and 32 leaders gave the highest average among these settings. The 0.20/32 result was nearly identical, so this small difference should not be treated as a universal optimum. I use 0.25/32 for DIRRIP in the results table and figures. `data.csv` contains the per-trace measurements for every sweep setting and the additional insertion-at-RRPV-0 comparisons.

Instruction insertion at RRPV 0 reduces instruction MPKI, but increases data MPKI on Trace1 and Trace2. Insertion at RRPV 1 gives a more balanced result across these traces, so it is retained.

## Figures and measurements

The measurements used a single core, with 25 million warmup instructions and 25 million measured instructions per trace. All policies used the same simulator configuration and `ip_stride` L1D prefetcher. The figures compare RRIP, I-RRIP, and the selected DIRRIP setting (threshold 0.25, 32 leaders). The full sensitivity sweep is reported in the table above and in `data.csv`.

- `ipc.png`: IPC comparison. The y-axis is zoomed for each trace, and each value is labeled.
- `speedup.png`: IPC gain over RRIP, shown as a percentage so small gains are visible.
- `load_mpki.png`: absolute L2 total load, instruction-load, and data-load MPKI. Axes are zoomed by trace and metric, and values are labeled.
- `data.csv`: all plotted and tabulated measurements
