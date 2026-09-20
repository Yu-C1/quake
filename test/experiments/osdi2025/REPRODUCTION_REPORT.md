# Quake OSDI 2025 Artifact Reproduction Report

2026-09-19 · Yutao

## Summary

This reports an attempt to reproduce the experimental results of *Quake: Adaptive Indexing for Vector Search* (OSDI 2025) from its published artifact, the `osdi2025` branch of `marius-team/quake`. All work ran on CloudLab hardware over 18–19 September 2026.

Seven experiments were executed: the six the artifact's README documents, plus `vary_levels`. The headline outcome is that **the hardware-independent quantities reproduce essentially exactly, while every latency figure is uniformly 3–4x slower than published** — a consistent offset attributable to the test machine rather than to the system.

| Claim | Target | Outcome |
| --- | --- | --- |
| Early termination comparison | Table 5 | Reproduced — all six methods match on recall and nprobe |
| Adaptive partition scanning | Figure 6 | Reproduced — including the zero-tuning-cost claim |
| Maintenance ablation | Table 6 | Partial — conclusions hold, two quantitative discrepancies unresolved |
| NUMA thread scaling | Figure 5 | Not reproduced — wrong dataset available, and a bug prevents NUMA placement |

Alongside the measurements, **eight defects in the artifact were identified and fixed** — six of them dependency drift that rendered the install script inoperable, plus a blocking terms-of-service prompt and a stale internal API call. A ninth, a memory-safety bug in the NUMA allocation path, was diagnosed but not fixed, and is the reason Figure 5 could not be attempted honestly.

The practical conclusion: Quake's central algorithmic claims hold up under independent measurement. The artifact itself, eighteen months after publication, does not run without repair.

## Environment and methodology

### Hardware

All measurements ran on a single **CloudLab `c240g5`** node at the Wisconsin cluster — bare metal, exclusively allocated, no virtualization:

|  |  |
| --- | --- |
| CPU | 2x Intel Xeon Silver 4114, 10 cores @ 2.20 GHz (40 hardware threads) |
| Memory | 192 GB across 2 NUMA nodes (95 GB + 96 GB) |
| NUMA distances | local 10, remote 21 |
| OS | Ubuntu 22.04.2 LTS (`UBUNTU22-64-X86`) |

The dual-socket configuration matters: it is the reason `c240g5` was chosen over single-socket alternatives, since the NUMA experiments require more than one memory domain. Node type was held constant across both sessions, so all results are mutually comparable.

### Software

The artifact's `install.sh` specifies most dependencies without version constraints. Eighteen months of upstream releases had made those unresolvable, so the following were pinned to restore a working build:

| Package | Pinned | Reason |
| --- | --- | --- |
| torch | 2.5.1 | Newer headers require C++20; `CMakeLists.txt` builds C++17 |
| numpy | 1.26.4 | Last 1.x release — satisfies pandas' `>=1.26` floor while keeping the 1.x C ABI diskannpy was built against |
| faiss-cpu | 1.9.0 | Current builds force numpy `>=2`, and 1.15.1's Python wrapper is broken |
| matplotlib | 3.8.4 | `matplotlib.cm.get_cmap` was removed in 3.9 |
| tabulate | any | Imported but never installed |

For reference, unpinned resolution on a fresh node in September 2026 selected **torch 2.14.0** — six minor versions beyond what the artifact was written against.

### Execution

Every run used `OMP_NUM_THREADS=1`, as the README directs: Quake manages its own thread pool, and OpenMP spawning threads underneath it would oversubscribe the cores and distort latency. Experiments were launched through `experiment_runner.py` from the repository root, inside `tmux` so that browser-shell disconnections could not interrupt long runs.

All configurations, fixes and results are committed to the `osdi2025` branch of the fork `Yu-C1/quake`.

## Early termination — Table 5

**Reproduced.** This experiment compares six strategies for deciding, per query, when to stop scanning partitions. It is the paper's central comparison: Quake's Adaptive Partition Scanning (APS) against learned, geometric, fixed and oracle alternatives. Run on SIFT1M with a 1,000-partition index, 10,000 queries, k=100 — matching the published configuration exactly.

| Method | Target | Recall (paper → measured) | nprobe (paper → measured) | Latency (paper → measured) | Tuning (paper → measured) |
| --- | --- | --- | --- | --- | --- |
| APS | 0.80 | 0.821 → 0.821 | 11.8 → 11.79 | 0.34 → 1.16 ms | 0 → **0** |
| APS | 0.90 | 0.912 → 0.912 | 20.2 → 20.15 | 0.48 → 1.74 ms | 0 → **0** |
| APS | 0.99 | 0.989 → 0.989 | 50.1 → 50.12 | 0.96 → 3.78 ms | 0 → **0** |
| LAET | 0.80 | 0.813 → 0.813 | 10.5 → 10.52 | 0.29 → 1.06 ms | 81 → 334 s |
| LAET | 0.90 | 0.905 → 0.905 | 18.2 → 18.28 | 0.42 → 1.62 ms | 104 → 430 s |
| LAET | 0.99 | 0.990 → 0.990 | 58.3 → 58.34 | 1.03 → 4.39 ms | 232 → 974 s |
| Auncel | 0.80 | 0.857 → 0.855 | 16.4 → 16.39 | 0.41 → 1.52 ms | 66 → 304 s |
| Auncel | 0.90 | 0.981 → 0.981 | **73.8 → 51.44** | 1.29 → 4.00 ms | 74 → 343 s |
| Auncel | 0.99 | 0.997 → 0.997 | 95.9 → 98.21 | 1.61 → 7.16 ms | 83 → 515 s |
| SPANN | 0.80 | 0.816 → 0.816 | 11 → 11 | 0.31 → 1.08 ms | 173 → 374 s |
| SPANN | 0.90 | 0.902 → 0.901 | 19 → 19 | 0.43 → 1.58 ms | 183 → 400 s |
| SPANN | 0.99 | 0.990 → 0.990 | 70 → 70 | 1.07 → 4.23 ms | 259 → 567 s |
| FixedNProbe | 0.80 | 0.817 → 0.817 | 11 → 11 | 0.33 → 1.06 ms | 318 → 705 s |
| FixedNProbe | 0.90 | 0.903 → 0.903 | 19 → 19 | 0.44 → 1.63 ms | 330 → 734 s |
| FixedNProbe | 0.99 | 0.990 → 0.990 | 65 → 65 | 1.16 → 4.74 ms | 424 → 941 s |
| Oracle | 0.80 | 0.833 → 0.833 | 11.5 → 11.46 | 0.29 → 0.87 ms | 320 → 728 s |
| Oracle | 0.90 | 0.924 → 0.924 | 19.3 → 19.31 | 0.41 → 1.38 ms | 331 → 753 s |
| Oracle | 0.99 | 0.992 → 0.992 | 42.0 → 41.99 | 0.74 → 2.93 ms | 368 → 835 s |

### What reproduced

**Recall matches on all eighteen rows**, in most cases to three decimal places. **nprobe matches on seventeen of eighteen.** These are the quantities that depend on the algorithm rather than the machine, and their agreement is strong evidence that the build is correct and the methods behave as described.

The paper's qualitative claims hold as stated. Auncel overshoots its recall target substantially — asked for 0.90, it delivers 0.98 while scanning 2.8x the partitions LAET needs for the same nominal target. APS lands closest to each requested target. Every competing method pays a calibration cost between 304 and 974 seconds; APS pays none.

### The one mismatch

**Auncel at target 0.90 scanned 51.4 partitions against the published 73.8**, despite achieving identical recall (0.981). Auncel's geometric parameter `a` is selected by binary search over `[1e-5, 0.5]`, accepting any value whose mean recall clears the target. A search that converges on a different acceptable value would produce exactly this: same recall, different work. The measured value is *better* than published, so this is not a failure to reproduce so much as an unexplained difference in calibration.

## Maintenance ablation — Table 6

**Partially reproduced.** This experiment disables individual components of Quake's maintenance policy — the cost model, the rejection mechanism, partition refinement — and measures the damage. Seven variants, 1,000 operations each on a dynamic SIFT1M workload (30% inserts, 20% deletes, 50% queries).

| Variant | Recall (paper → measured) | Search (paper → measured) | Maintenance (paper → measured) |
| --- | --- | --- | --- |
| Quake (full) | 0.905 → 0.908 | 86.3 → 473.8 s | 21.4 → 1132.9 s |
| NoCost | 0.901 → 0.900 | 93.5 → 571.6 s | 20.4 → 1127.8 s |
| LIRE | 0.900 → 0.902 | 100.5 → 598.6 s | 11.9 → 418.2 s |
| NoCost+NoRef | 0.879 → 0.881 | 100.7 → 602.4 s | 0.8 → 3.7 s |
| NoRef | 0.881 → 0.873 | 101.7 → 495.2 s | 5.2 → 72.3 s |
| NoRef+NoRej | 0.730 → **0.780** | 85.5 → 473.3 s | 1.0 → 6.8 s |
| NoRej | 0.662 → **0.775** | 84.2 → 444.4 s | 18.5 → 1093.4 s |

### What reproduced

The paper's conclusions hold. Among variants that actually meet the recall target, **full Quake has the lowest search cost** (473.8 s, against NoCost at 571.6 s and LIRE at 598.6 s) — the same ordering as published, supporting the claim that the cost model and refinement together minimise search latency. Removing refinement raises search time; removing rejection collapses recall. Four of seven variants match published recall within 0.003.

### Discrepancy 1: the rejection variants

The two variants with rejection disabled came out **substantially better than published** — NoRej at 0.775 against 0.662, NoRef+NoRej at 0.780 against 0.730. The qualitative finding survives (recall does collapse relative to the 0.905 baseline, and rejection is clearly load-bearing), but the magnitude does not. No explanation identified.

### Discrepancy 2: maintenance time, and a likely cause

Maintenance time runs far above the published figures — full Quake at 1132.9 s against 21.4 s, a factor of 53. That is an order of magnitude beyond the 3–6x hardware offset seen everywhere else, so it is not simply a slower machine.

Sorting the variants by that ratio reveals the pattern:

| Variant | Sets `refinement_radius: 0`? | Maintenance ratio |
| --- | --- | --- |
| NoCost+NoRef | yes | 4.6x |
| NoRef+NoRej | yes | 6.8x |
| NoRef | yes | 13.9x |
| LIRE | no | 35x |
| Quake (full) | no | 53x |
| NoCost | no | 55x |
| NoRej | no | 59x |

The split is clean: **every variant that disables refinement is within the normal hardware offset; every variant that performs refinement is 35–59x over.** The cost is concentrated entirely in refinement.

The paper states: *"For configurations with refinement we use a refinement radius of rₓ = 50."* The artifact's `maintenance_ablation/configs/sift1m.yaml` **never sets `refinement_radius`** for the refining variants — it only sets it to `0` on the NoRefine ones. Those variants therefore ran at whatever the code's built-in default is, which appears to be considerably more expensive than 50.

This is a config gap rather than a reproduction failure, but it means **the maintenance-time column should not be cited** until the experiment is re-run with `refinement_radius: 50` set explicitly.

## Adaptive partition scanning — Figure 6

**Reproduced.** This experiment sweeps recall targets from 0.2 to 0.999, comparing APS against an Oracle that determines each query's minimum necessary nprobe by brute-force search against ground truth. The Oracle is not deployable — it consults the answer it is meant to find — and exists to define the lower bound.

| Target | APS recall | Oracle recall | APS nprobe | Oracle nprobe | APS overhead |
| --- | --- | --- | --- | --- | --- |
| 0.20 | 0.310 | 0.352 | 1.04 | 1.53 | −32% |
| 0.40 | 0.458 | 0.520 | 2.05 | 2.87 | −29% |
| 0.60 | 0.654 | 0.677 | 4.63 | 4.98 | −7% |
| 0.80 | 0.837 | 0.841 | 10.35 | 9.66 | +7% |
| 0.90 | 0.917 | 0.925 | 17.38 | 16.03 | +8% |
| 0.95 | 0.954 | 0.967 | 25.30 | 24.11 | +5% |
| 0.99 | 0.990 | 0.992 | 45.43 | 36.90 | +23% |
| 0.999 | 0.997 | 1.000 | 67.04 | 55.84 | +20% |

The paper claims APS "matches the nprobe of an oracle across recall targets on Sift1M, with only a 20% increase in latency relative to the oracle." The measurement supports this: across the operating range that matters (0.80 and above), APS tracks the requested target closely and stays within 5–23% of optimal nprobe, with the gap widening only at the extreme 0.99–0.999 targets.

The corresponding rows in the Table 5 run make the same point from a different angle, and reproduce the published values to three decimals:

| Target | Recall | nprobe | Tuning cost |
| --- | --- | --- | --- |
| 0.80 | 0.821 | 11.79 | **0 s** |
| 0.90 | 0.912 | 20.15 | **0 s** |
| 0.99 | 0.989 | 50.12 | **0 s** |

The zero in that last column is the paper's actual argument. APS is not dramatically more accurate than LAET or Auncel — it is comparably accurate *without an offline calibration phase*. In this run the competing methods spent between 304 and 974 seconds tuning per recall target, a cost that would have to be repaid every time the index or data distribution shifts. For a system whose premise is continuously updated indexes, that distinction is the point.

## NUMA scaling — Figure 5

**Not reproduced.** Two independent obstacles prevented an honest attempt, and a third makes the measurement that *was* obtained uninterpretable as a NUMA result.

First, the dataset. Figure 5's caption reads *"MSTuring100M: Scaling the number of threads with and without NUMA."* The artifact ships only a `msturing10m` config — already a tenfold reduction from what was published — and `ann_datasets.py` cannot download either, raising *"Please download it manually using big-ann-benchmarks."* MSTuring100M is roughly 40 GB, exceeding the 38 GB of free disk on a default node. Measurements therefore ran on SIFT1M, at ~512 MB.

### Thread scaling (the half that worked)

Single-query search time per query, in milliseconds:

| Workers | non-NUMA | NUMA | non-NUMA σ | NUMA σ |
| --- | --- | --- | --- | --- |
| 1 | 2.369 | 2.149 | 0.031 | 0.275 |
| 2 | 2.204 | 1.568 | **1.698** | 0.223 |
| 4 | 0.854 | 0.836 | 0.002 | 0.003 |
| 8 | 0.754 | 0.828 | 0.005 | 0.004 |
| 16 | 0.813 | 0.734 | 0.009 | 0.002 |
| 32 | 1.108 | 0.612 | **0.593** | 0.001 |
| 64 | 198.877 | 183.328 | 9.133 | 31.698 |

Scaling is clean and plausible up to eight workers — 2.37 ms down to 0.75 ms, a 3.2x speedup — then plateaus at 16 and regresses at 32. At 64 workers performance collapses to ~190 ms, roughly 250x worse than the optimum, which is expected: the node has 40 hardware threads, so 64 workers oversubscribe it badly.

### Why the NUMA comparison is void

The NUMA and non-NUMA columns differ by small, inconsistent amounts — NUMA appears faster at 1, 2, 16 and 32 workers, slower at 8, identical at 4. The two apparently large gaps are exactly the rows where the non-NUMA standard deviation is enormous (σ = 1.698 against a mean of 2.204; σ = 0.593 against 1.108). These are not effects; they are noise.

There is a concrete explanation. Throughout every run, including runs where **every configuration set `use_numa: false`**, stderr filled with `mbind: Invalid argument`. Quake allocates partition memory via `numa_alloc_onnode()`, which mmaps and then calls `mbind()` to pin pages to a node; when `mbind` fails, libnuma prints that message but **still returns a valid pointer**. Allocation succeeds and NUMA placement is silently skipped. The "NUMA" runs were therefore almost certainly not using NUMA-placed memory at all.

Separately, SIFT1M at ~512 MB against 192 GB across two sockets is unlikely to stress the interconnect even with working placement — which is presumably why the paper chose a 100M-vector dataset for this figure.

### Multi-query

Batch latency, 1,000 queries at k=100:

| Workers | non-NUMA | NUMA |
| --- | --- | --- |
| 0 | 3629.6 ms | 377.8 ms |
| 1 | 1521.1 ms | 1769.0 ms |
| 4 | 418.2 ms | 623.5 ms |
| 16 | 246.2 ms | 332.0 ms |

These numbers should not be read as a NUMA comparison either, for a separate reason: the paired configurations differ in more than the NUMA flag. Every non-NUMA config sets `batched_scan: true` while every NUMA config sets `batched_scan: false`, and `Quake_0_numa` additionally sets `n_threads: 16`. Two variables change at once, so the contrast is not controlled.

## Hierarchy depth — vary_levels

**Not a reproduction** — included as a supporting measurement. The artifact ships only a SIFT10M config for this experiment, and SIFT10M cannot be downloaded (`Sift10m.download()` is a no-op). A scaled-down SIFT1M config was written locally, preserving the paper's partition density of ~250 vectors per partition. There is no published counterpart to compare against.

The experiment measures what a multi-level index buys. `ChildOnly_L0` is a flat index over 4,000 partitions; the others add one or two parent levels above it.

Latency at matched recall ≈ 0.90 (L0 nprobe = 16):

| Configuration | Latency | vs flat |
| --- | --- | --- |
| ChildOnly_L0 | 54.6 ms | — |
| L0 + L1 (100 centroids) | 14.0 ms | 3.9x faster |
| L0 + L1 + L2 | 11.7 ms | 4.7x faster |
| L0 + L1 (50 centroids) | 10.0 ms | 5.5x faster |

**Hierarchy is a large win.** The flat index costs 33.9 ms even at `nprobe = 1`, because every query must compare against all 4,000 centroids before scanning anything. A parent level replaces that with 50–100 comparisons. That single fact accounts for most of the difference.

**The cost is a recall ceiling.** The flat index reaches 1.00 recall by nprobe 128; the hierarchical variants plateau at 0.98 (100-centroid parents) or 0.96 (50-centroid). Parent-level pruning can discard a partition containing a true neighbour, and no amount of additional L0 scanning recovers it. The ordering is instructive: the most aggressive pruning is simultaneously the fastest and the least accurate.

**A third level does not pay at this scale.** Comparing two levels against three: 13.95 ms vs 11.72 ms at nprobe 16, but 224.7 ms vs 244.0 ms at nprobe 512, with identical recall plateaus. Roughly a wash. Scanning 100 L1 centroids is already cheap enough that summarising them with an L2 saves little. At SIFT1M scale, two levels is enough — though this says nothing about the 10M scale the paper actually studied.

## Defects found in the artifact

The artifact does not build as shipped. Eight defects were identified and fixed; a ninth was diagnosed but not repaired. All fixes are committed to the fork.

### Dependency drift (six)

The install script leaves most versions unconstrained, so a fresh install in September 2026 resolves packages roughly eighteen months newer than those the artifact was written against.

| # | Defect | Effect | Fix |
| --- | --- | --- | --- |
| 1 | `pip install . --no-use-pep517` | Flag removed from modern pip; install aborts immediately | `--no-build-isolation` (isolation must stay off, since `setup.py` imports torch at build time) |
| 2 | torch unpinned → resolves to 2.14.0 | Its headers require C++20 while `CMakeLists.txt:20` sets C++17; compilation fails on `std::strong_ordering` | Pin 2.5.1 |
| 3 | faiss-cpu unpinned → 1.15.1 | Requires numpy ≥ 2, silently upgrading numpy and breaking diskannpy's 1.x ABI. The wheel is also broken in itself: `NameError: SuperKMeans` on import | Pin 1.9.0 |
| 4 | numpy pinned to 1.25.0 | The environment's pandas requires ≥ 1.26, so pandas cannot import | Pin 1.26.4 — the last 1.x release, satisfying both |
| 5 | matplotlib unpinned | `matplotlib.cm.get_cmap`, used at `maintenance_ablation/run.py:23`, was removed in 3.9 | Pin 3.8.4 |
| 6 | `tabulate` never installed | Imported by `maintenance_ablation/run.py`. Since `experiment_runner.py` imports every experiment module at load time, this broke *all* experiments, not just that one | Add to the install list |

### Environment (one)

**7.** Recent conda refuses non-interactive environment creation until the default channels' terms are accepted, aborting the script with `CondaToSNonInteractiveError`. Fixed by accepting them in `install.sh` before the environment is built.

### Stale internal API (one)

**8.** `numa_multi_query/run.py:101` calls `common_utils.prepare_index()`, which does not exist — `experiment_utils.py` provides `prepare_quake_index()` with a different signature. The experiment aborted at startup with `AttributeError`. `numa_single_query` calls the current function correctly, so this appears to be an un-migrated caller. Fixed by mirroring the working call site, including the `nc` → `nlist` key translation that `prepare_quake_index` requires.

### Diagnosed, not fixed: memory safety in the NUMA path

**9.** Two linked symptoms, reproducible on correctly configured dual-socket hardware:

- `mbind: Invalid argument` floods stderr on every run — **including runs where every configuration sets `use_numa: false`**, so the call is evidently unconditional. Since `numa_alloc_onnode()` returns a valid pointer regardless, the failure is silent to the program and NUMA placement simply does not occur.
- Running NUMA and non-NUMA configurations in one process **aborts at the transition between them**, with `free(): invalid next size` or `double free or corruption`. Each mode completes cleanly in isolation; only crossing between them fails. Even a single-mode run aborts at process teardown, after results are written.

The obvious culprits were checked and are correctly written: `free_memory()` (`index_partition.cpp:195-214`) pairs `std::free` with malloc'd memory and `numa_free` with NUMA-allocated, and `reallocate_memory()` (`216-237`) frees before updating `buffer_size_`, so `numa_free` receives the right size. The corruption originates elsewhere — `set_numa_node()`, the worker pool in `parallel.h`, or the batched-scan buffers are the remaining candidates.

**Workaround used:** the NUMA and non-NUMA configurations were split into separate config files and run as separate processes, which completes both. This is a workaround, not a fix — the underlying memory-safety bug remains, and it is why no NUMA result can be trusted from these runs.

## Limitations

**Latency is not comparable to the paper.** Every measured latency runs 3–6x the published figure, consistently across unrelated experiments and methods. The uniformity points at the machine rather than the build — the `c240g5`'s Xeon Silver 4114 at 2.20 GHz is a modest CPU, and the paper does not state what it measured on. Recall, nprobe and partitions-scanned, which do not depend on the machine, reproduced exactly. **Any latency figure in this report should be read as relative, not absolute.**

**Most datasets are out of reach.** Only SIFT1M downloads automatically. SIFT10M's `download()` is a no-op; MSTuring10M and 100M raise an instruction to fetch them by hand from big-ann-benchmarks. Everything here therefore ran on a single 1M-vector dataset — in the NUMA case, one hundred times smaller than what was published.

**Wikipedia-12M is absent entirely.** It carries Figure 1, Table 3 and half the APS claim in the paper, and is described there as a contribution to be released. The artifact has no loader for it, and `measure_wiki_skew_and_perf/run.py` and `workload_comparison/run.py` are both 2-byte empty stubs. The README's final section is headed "Full Reproduction (In progress)", so this appears to be known. OpenImages-13M is likewise unavailable, so Table 3's end-to-end comparison cannot be attempted at all.

**Three discrepancies remain unexplained:**

1. Maintenance times 35–59x published, for refining variants only. A probable cause is identified (`refinement_radius` unset) but untested.
2. The NoRej variants achieve 8–11 points higher recall than published.
3. Auncel at target 0.90 scans 51.4 partitions against a published 73.8, at identical recall.

**Two experiments are measurements rather than reproductions.** `vary_levels` ran on a locally written config with no published counterpart; `numa_multi_query` is labelled a "Revision Experiment" with no figure cited.

**Single-run results.** Each experiment ran once. Within-experiment repetition exists (three trials in the NUMA and hierarchy experiments, 10,000 queries in the termination comparison), but no experiment was repeated end to end, so run-to-run variance across full executions is unmeasured.

## Next steps

Ranked by value against effort.

**1. Re-run the maintenance ablation with `refinement_radius: 50`.** The cheapest open item and the one that would close the largest quantitative gap. The paper specifies that value; the config omits it. One config line, one experiment run (~90 minutes), and either the 35–59x maintenance discrepancy resolves or a genuine difference is confirmed. Until then the maintenance-time column should not be cited.

**2. Report the NUMA memory-safety bug upstream.** The evidence is strong and reproducible: `mbind` called unconditionally regardless of `use_numa`, a deterministic abort at mode transitions, and a teardown abort even in single-mode runs, all on a node with two healthy NUMA domains. A maintainer who knows the allocator will find this faster than an outside reader. This also gates any future Figure 5 attempt.

**3. Obtain MSTuring10M.** ~4 GB via big-ann-benchmarks, and the config already exists. It would not reproduce Figure 5 — that used MSTuring100M — but a 10M-vector working set is large enough to exercise memory bandwidth, unlike SIFT1M. Worth doing only after item 2, since without working `mbind` the comparison remains void.

**4. Fix the multi-query configuration so NUMA is the only variable.** As shipped, the NUMA configs also flip `batched_scan` and one sets `n_threads`, so the comparison confounds two or three changes. A corrected config is a few lines and makes the experiment interpretable.

**5. Create a CloudLab disk image.** Currently blocked: the DSDM project's image storage is over quota and the snapshot failed. Installing from scratch costs ~3 hours per node, which consumed most of one session. Resolving the quota with the project administrator would make every subsequent run cheap, and would also freeze the pinned dependency set against further upstream drift — a reproducibility gain in its own right.

**6. Ask the authors about Wikipedia-12M.** The paper commits to releasing it. Without it, Figure 1 and Table 3 are permanently out of reach, and those carry a substantial share of the paper's evidence.

**Also worth considering:** repeating at least one experiment end to end to establish run-to-run variance, and writing a scaled-down SIFT1M config for `multi_level` — the one remaining experiment that has never run — though it overlaps `vary_levels` and adds mainly a Faiss-IVF comparison.
