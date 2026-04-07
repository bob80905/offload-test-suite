# Workflow Speed Optimization Plan — llvm/offload-test-suite

## Problem Statement

PR workflow wall-clock time from last commit to last workflow completion is ~5 hours (observed range: 0.8–7.6 hours across 20 recent PR runs). The root causes are:

1. **Single self-hosted runner per GPU SKU** — all jobs for a given SKU must run serially
2. **Redundant DXC + LLVM builds** — every single job builds both from source (~5–20 min each)
3. **Scheduled workflow contention** — scheduled runs compete for the same runners, causing massive queue waits (observed: up to 450 min)

---

## Evidence Files (CSVs)

Five CSV files have been generated in `.github/workflows/`:

1. **`scheduled-workflows.csv`** — All scheduled workflows with vendor, cron schedule, frequency, and exact file:line references
2. **`timing-test-all-runs.csv`** — Per-vendor wall-clock timing for 9 recent `test-all` PR runs
3. **`timing-default-runs.csv`** — Per-vendor wall-clock timing for 11 recent default (no `test-all`) PR runs
4. **`scenario-comparison.csv`** — Modeled impact of adding machines in different configurations
5. **`build-vs-test-timing.csv`** — Step-level build vs test profiling for 854 jobs across 59 PR runs (last 30 days)

---

## Build vs Test Time Split (from 854 jobs across 59 PR runs)

**Build time dominates test time on all vendors:**

| Vendor | Jobs | Avg Build | Avg Test | Build % | Test % |
|--------|------|-----------|----------|---------|--------|
| Qualcomm | 206 | **12.9 min** | 1.1 min | **81.9%** | 7.5% |
| AMD | 88 | **10.9 min** | 0.4 min | **86.0%** | 3.6% |
| Intel | 354 | **5.7 min** | 1.2 min | **68.2%** | 14.9% |
| NVIDIA | 88 | **5.3 min** | 0.9 min | **66.8%** | 13.3% |
| macOS | 118 | 1.1 min | 0.1 min | 41.8% | 3.5% |

**Key insight**: 67–86% of job time on Windows runners is spent building DXC + LLVM. Tests themselves take only 0.4–1.2 minutes on average. **Pre-building artifacts would eliminate the vast majority of per-job time.**

---

## Current Architecture

### PR Matrix (`pr-matrix.yaml`) — triggered on every PR

| Job Group | SKU(s) | Job Count | Condition |
|-----------|--------|-----------|-----------|
| Exec-Tests-Windows | `windows-intel` | 4 | Always |
| Exec-Tests-Windows-Warp | `windows-intel`, `windows-qc` | 4 (2 per SKU) | Always |
| Exec-Tests-Extra | `windows-nvidia`, `windows-amd`, `windows-qc` | 12 (4 per SKU) | Only with `test-all` label |
| Exec-Tests-MacOS | `macos` | 2 | Always |

**Default PR (no label): 10 jobs** — Intel(6), QC(2), macOS(2)
**test-all PR: 22 jobs** — Intel(6), QC(6), AMD(4), NVIDIA(4), macOS(2)

~33% of PRs get the `test-all` label (25 of 76 in last 30 days).

### Callable Workflow (`build-and-test-callable.yaml`)

Every job does ALL of these steps sequentially on the GPU runner:
1. Checkout DXC, LLVM, OffloadTest, Golden Images (4 repos)
2. Build DXC (CMake + Ninja)
3. Build LLVM (CMake + Ninja)
4. Run tests

### Scheduled Workflows (individual per-vendor files)

These run on a schedule AND compete for the same self-hosted runners. See `scheduled-workflows.csv` for full details with file:line references.

| SKU | Scheduled Workflows | Frequency | Est. Daily Runner Busy Time |
|-----|---------------------|-----------|-----------------------------|
| `hlsl-windows-intel` | 4 | 1x hourly, 3x every 2 hrs | ~8.2 hrs/day (34% busy) |
| `hlsl-windows-nvidia` | 4 | 1x hourly, 3x every 2 hrs | ~6.6 hrs/day (28% busy) |
| `hlsl-windows-qc` | 6 | Every 6 hours | ~6.4 hrs/day (27% busy) |
| `hlsl-windows-amd` | 6 + 1 GBV | 6x every 6 hours, 1x daily | ~5.0 hrs/day (21% busy) |

### Per-Job Execution Times (from recent runs)

| SKU | DXC-target Jobs | Clang-target Jobs | Avg Per Job |
|-----|-----------------|-------------------|-------------|
| `windows-nvidia` | ~5–6 min | ~7–9 min | ~7 min |
| `windows-intel` | ~6–7 min | ~9–11 min | ~8 min |
| `windows-amd` | ~9–10 min | ~15–17 min | ~12.5 min |
| `windows-qc` | ~11–12 min | ~19–20 min | **~16 min (slowest)** |

---

## Analysis: Q1 — Which Vendor Should Get an Extra Machine?

### Answer: **Qualcomm is unequivocally the right choice** (given Intel is already in the works)

### Evidence Summary (from 20 PR runs, Apr 1–6, 2026)

**Test-all runs (9 runs):**
| Metric | Value |
|--------|-------|
| Avg wall time | **4.76 hours** |
| Max wall time | **7.62 hours** |
| Bottleneck = Qualcomm | **8 out of 9 runs (89%)** |
| Bottleneck = Intel | 1 out of 9 runs (11%) |

**Default runs (11 runs):**
| Metric | Value |
|--------|-------|
| Avg wall time | **3.24 hours** |
| Max wall time | **6.61 hours** |
| Bottleneck = Intel | **10 out of 11 runs (91%)** |
| Bottleneck = Qualcomm | 1 out of 11 runs (9%) |

### Scenario Modeling: 2 Intel vs 1 Intel + 1 Qualcomm

Given that a new Intel machine is already in the works (2 Intel total), the question is whether the next investment should be a 2nd Intel (3I total) or a Qualcomm (2I+2Q). The analysis uses actual wall-clock data from 20 PR runs to model each scenario.

| Scenario | Test-All Avg | Test-All Max | Default Avg | Default Max | Weighted Avg |
|----------|-------------|-------------|-------------|-------------|-------------|
| **Current** (1I, 1Q) | 4.76 hrs | 7.62 hrs | 3.24 hrs | 6.61 hrs | **3.74 hrs** |
| **+1 Intel** (2I, 1Q) | 4.66 hrs | 7.62 hrs | 2.01 hrs | 4.82 hrs | **2.88 hrs** |
| **+1I +1QC** (2I, 2Q) | **2.40 hrs** | **3.81 hrs** | **1.62 hrs** | **3.31 hrs** | **1.88 hrs** |
| **+2 Intel** (3I, 1Q) | 4.66 hrs | 7.62 hrs | 1.68 hrs | 4.82 hrs | **2.66 hrs** |

See `scenario-comparison.csv` for this data.

### Why 1I+1QC is definitively better than 2 extra Intel

The **2I+2Q scenario beats 3I+1Q in every single metric**:

- **Test-all avg**: 2.40 vs 4.66 hrs → QC wins by **2.26 hours** (49% better)
- **Test-all max**: 3.81 vs 7.62 hrs → QC wins by **3.81 hours** (50% better)
- **Default avg**: 1.62 vs 1.68 hrs → QC still slightly wins
- **Default max**: 3.31 vs 4.82 hrs → QC wins by **1.51 hours**
- **Weighted avg**: 1.88 vs 2.66 hrs → QC wins by **0.78 hours** (29% better)

**Key insight**: A 3rd Intel machine provides diminishing returns because:
1. With 2 Intel machines, Intel is no longer the bottleneck — **QC becomes the new bottleneck**
2. Adding a 3rd Intel doesn't help because QC (still with 1 machine) caps the overall time
3. QC jobs are 2x slower per job (~16 min vs ~8 min for Intel), so QC runner scarcity is the binding constraint
4. **Even for default runs** (no test-all, where QC only has 2 jobs), QC occasionally becomes the bottleneck once Intel is fast enough — which is why 2I+2Q edges out 3I+1Q even there

---

## Analysis: Q2 — Other Practical Optimizations

### Optimization 1: Pre-build LLVM/DXC Artifacts (User's Idea) ⭐ Highest Impact

**Current problem**: Every job independently checks out and builds DXC + LLVM. With 22 jobs in a test-all run, that's 22 redundant builds of the same code.

**Proposal**: Create a separate "builder" workflow that:
1. Runs on a schedule (e.g., hourly or on LLVM/DXC main branch commits)
2. Builds DXC and LLVM from source
3. Uploads build artifacts (or caches them on the runner)
4. PR test jobs download the pre-built artifacts instead of building

**Impact estimate**: Could reduce per-job time by 60–80%. For example:
- QC Clang jobs: ~20 min → ~5–8 min (mostly test execution time)
- Intel DXC jobs: ~7 min → ~2–3 min
- Total QC serial (6 jobs): ~96 min → ~30–48 min

**Considerations**:
- The builder needs to build for each target architecture (x64, ARM64 for QC)
- Builds need to include all cmake option variants (BuildType, clang-tidy, GBV flags, etc.)
- Need to handle artifact staleness (what if LLVM changes mid-PR?)
- Could use GitHub Actions artifact cache or a shared network path on self-hosted runners
- sccache already exists but appears insufficient — full builds still take 5–20 min

### Optimization 2: Reduce Scheduled Workflow Frequency

**Current problem**: Scheduled workflows occupy runners 21–34% of each day, causing queue contention for PR jobs.

**Proposals** (see `scheduled-workflows.csv` for exact file:line references):
- `windows-intel-dxc-d3d12.yaml:9` — runs every **hour** → reduce to every 2 hours
- `windows-nvidia-dxc-d3d12.yaml:9` — runs every **hour** → reduce to every 2 hours
- This alone could meaningfully reduce Intel and NVIDIA queue contention

### Optimization 3: Give PR Jobs Priority Over Scheduled Jobs

**Proposal**: Add concurrency groups that cancel in-progress scheduled runs when PR jobs need the runner:
```yaml
concurrency:
  group: hlsl-${{ inputs.SKU }}
  cancel-in-progress: ${{ github.event_name == 'schedule' }}
```
This would allow PR jobs to preempt scheduled runs rather than waiting behind them.

### Optimization 4: Split Build from Test in the Callable Workflow

**Current problem**: The build (no GPU needed) and test (GPU needed) run in a single job, holding the GPU runner for the entire duration including build time.

**Proposal**: Split into a 2-job pipeline:
1. **Build job**: Runs on a generic (non-GPU) runner, builds DXC + LLVM
2. **Test job**: Runs on the GPU runner, downloads build artifacts and runs tests only

This frees GPU runners faster and allows builds to run in parallel on cheaper hardware.

### Optimization 5: Cache DXC/LLVM Builds on Self-Hosted Runners

**Current problem**: Each job does a fresh checkout and build. Self-hosted runners persist between runs, so build artifacts from previous runs may still exist on disk.

**Proposal**: Modify the callable workflow to:
1. Check if a cached build exists for the current LLVM/DXC commit hashes
2. Skip the build step if artifacts are up-to-date
3. Only rebuild when source has changed

This is simpler than Optimization 1 but less reliable (depends on runner state).

### Optimization 6: Reduce Job Count in PR Matrix

**Proposal options**:
- Make some test targets non-blocking (run as informational, don't block merge)
- Merge related test targets (e.g., run d3d12+vk in a single job)
- Move WARP tests to a separate, non-blocking workflow

---

## Recommended Action Plan (Ordered by Impact × Feasibility)

### Phase 1: Quick Wins (workflow config changes only)

1. **Reduce scheduled workflow frequency** — Change Intel/NVIDIA DXC-D3D12 from hourly to every 2 hours
2. **Add concurrency groups** for scheduled workflows to allow PR jobs to preempt them
3. **Evaluate which scheduled workflows are essential** — can any be reduced further or removed?

### Phase 2: Architecture Changes (medium effort)

4. **Pre-build LLVM/DXC artifacts** — Create a builder workflow that produces cached artifacts for PR jobs
5. **Split build from test** in the callable workflow (build on generic runner, test on GPU runner)

### Phase 3: Hardware Investment

6. **Add a 2nd Intel machine** (already in the works) — reduces default run avg from 3.24 to 2.01 hrs
7. **Add a 2nd Qualcomm machine** — with 2 Intel already, this is the highest-impact next step. Reduces weighted avg from 2.88 to 1.88 hrs, test-all max from 7.62 to 3.81 hrs

---

## Data Collection Tasks (for further analysis)

- Measure sccache hit rates: Add `sccache --show-stats` step after "Build LLVM" in `build-and-test-callable.yaml` (~line 167) to capture hit/miss rates in job logs
- Check if multiple concurrent PRs are common (compounds the queuing problem)
- Investigate if DXC and LLVM builds can be separated (DXC changes less frequently)
