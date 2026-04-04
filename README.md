# Opportunistic Instruction Replication (OIR) for Fault Tolerance
**ENGG 4540 — Advanced Computer Architecture**
**Group 19, Project 21**
Omar Aslam (1213198) | Abdallah Al Hussami (1230248)

Based on: Waser et al., "FAULTLESS," DIMVA 2025
DOI: https://doi.org/10.1007/978-3-031-97623-0_18

---

## What This Project Does

This project implements Opportunistic Instruction Replication (OIR)
inside the gem5 out-of-order CPU simulator. When a 4-wide processor
issues fewer than 4 instructions in a cycle, the spare slots are used
to schedule replicas of recently issued ALU instructions. At the commit
stage, each primary result is compared against a baseline copy saved at
issue time. A mismatch indicates a transient fault.

---

## Repository Branches

| Branch | Purpose |
|--------|---------|
| `oir-implementation` | Clean OIR with zero overhead — use this for clean runs |
| `fault-injection` | Adds a single bit-flip at instruction #500000 for detection testing |
| `submission` | This branch — clean code + full README for submission |

---

## Files Modified from gem5 Baseline

All modifications are confined to three source files inside
`src/cpu/o3/`. No other files were changed.

### 1. `src/cpu/o3/dyn_inst.hh`
**What it does:** Defines the instruction object used throughout the pipeline.

**Our additions:** Five new fields added to the `DynInst` class:
- `bool isReplica` — marks this instruction as a replica copy
- `bool hasReplica` — marks that a replica has been assigned to this instruction
- `uint64_t oirResult` — stores the execution result captured at issue time
- `uint64_t oirResultCopy` — baseline copy saved before any fault can occur; compared at commit
- `bool oirCompared` — prevents the same instruction from being compared twice

### 2. `src/cpu/o3/inst_queue.hh` and `inst_queue.cc`
**What it does:** Manages the instruction queue and dispatch logic.

**Our additions:**
- Three new statistics counters declared in `inst_queue.hh`:
  - `oir_emptySlotsTotal` — total empty issue slots observed across all cycles
  - `oir_replicasInserted` — how many instructions were marked for replication
  - `oir_candidatesSkipped` — how many eligible candidates were found but had no spare slot
- Post-selection scan added to `scheduleReadyInsts()` in `inst_queue.cc`:
  - Runs every cycle after normal instruction dispatch
  - Counts empty slots: `emptySlots = issueWidth - total_issued`
  - Scans a window of recently issued instructions for ALU candidates
  - Excludes memory operations and branches (non-idempotent side effects)
  - For each eligible candidate, sets `hasReplica = true` and saves `oirResultCopy`

### 3. `src/cpu/o3/commit.hh` and `commit.cc`
**What it does:** Manages the commit stage where instructions retire.

**Our additions:**
- Three new statistics counters declared in `commit.hh`:
  - `oir_matches` — primary and replica results agreed; clean commit
  - `oir_mismatches` — results differed; transient fault detected
  - `oir_skipped` — instruction had no replica; committed normally
- Comparator logic added to `commitHead()` in `commit.cc`:
  - Runs before every instruction retires
  - If `hasReplica` is false: commit normally, increment `oir_skipped`
  - If `hasReplica` is true and `oirResult == oirResultCopy`: commit, increment `oir_matches`
  - If `hasReplica` is true and `oirResult != oirResultCopy`: fault detected, increment `oir_mismatches`

---

## Dependencies

These are the packages required to build gem5 on Ubuntu 22.04 or later.
Run the following command before building:
```bash
sudo apt update && sudo apt install -y \
    build-essential git m4 scons zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev \
    libgoogle-perftools-dev python3-dev python3-six \
    python3-pydot libboost-all-dev pkg-config
```

**Python version:** 3.8 or later
**SCons version:** 4.0 or later
**GCC version:** 9 or later

---

## Build Instructions
```bash
# Clone this repository
git clone https://github.com/Omar-aslam/gem5.git gem5-oir
cd gem5-oir

# Switch to the submission branch
git checkout submission

# Build gem5 for X86 (takes 30-60 minutes on first build)
scons build/X86/gem5.opt -j$(nproc)
```

Type `y` if prompted about git hooks. Build is complete when you see:
y
---

## Running the Benchmarks

### Get MiBench
```bash
cd ~
git clone https://github.com/embecosm/mibench.git
cd mibench/automotive/basicmath && make
cd ../qsort && make
cd ../susan && make
cd ~/mibench/network/dijkstra && make
```

### Run Each Benchmark

Replace `/home/$USER` with your actual home directory path.

**basicmath:**
```bash
cd ~/gem5-oir
./build/X86/gem5.opt -d m5out_basicmath \
    configs/deprecated/example/se.py \
    --cmd=/home/$USER/mibench/automotive/basicmath/basicmath_small \
    --cpu-type=O3CPU --caches
```

**qsort:**
```bash
./build/X86/gem5.opt -d m5out_qsort \
    configs/deprecated/example/se.py \
    --cmd=/home/$USER/mibench/automotive/qsort/qsort_small \
    --options="/home/$USER/mibench/automotive/qsort/input_small.dat" \
    --cpu-type=O3CPU --caches
```

**susan:**
```bash
./build/X86/gem5.opt -d m5out_susan \
    configs/deprecated/example/se.py \
    --cmd=/home/$USER/mibench/automotive/susan/susan \
    --options="/home/$USER/mibench/automotive/susan/input_small.pgm /tmp/out.pgm -s" \
    --cpu-type=O3CPU --caches
```

**dijkstra:**
```bash
./build/X86/gem5.opt -d m5out_dijkstra \
    configs/deprecated/example/se.py \
    --cmd=/home/$USER/mibench/network/dijkstra/dijkstra_small \
    --options="/home/$USER/mibench/network/dijkstra/input.dat" \
    --cpu-type=O3CPU --caches
```

---

## Reading the OIR Statistics

After each run, extract the OIR stats from the output directory:
```bash
grep "oir_\|system.cpu.ipc\|simInsts" m5out_basicmath/stats.txt
```

Expected output format:
simInsts                              48,814,702
system.cpu.ipc                         1.161533
system.cpu.oir_emptySlotsTotal       225,581,080
system.cpu.oir_replicasInserted       43,113,908
system.cpu.oir_candidatesSkipped      22,264,699
system.cpu.commit.oir_matches         42,077,217
system.cpu.commit.oir_mismatches               0
system.cpu.commit.oir_skipped         51,965,886
`oir_mismatches` should be **0** on all clean runs.

---

## Running Fault Injection

Switch to the fault-injection branch to test detection:
```bash
git checkout fault-injection
scons build/X86/gem5.opt -j$(nproc)
```

Then run any benchmark as above. The simulation will print:
warn: OIR FAULT INJECTED at instruction #500000 — bit-flip applied
warn: OIR: FAULT DETECTED [sn:XXXXXXX] original=0x0 flipped=0x1
And `oir_mismatches` will equal **1** in the stats file.

---

## Expected Results Summary

| Benchmark | IPC | Overhead | Replicas | Coverage | Mismatches |
|-----------|-----|----------|----------|----------|------------|
| basicmath | 1.1615 | 0% | 43.1M | 88% | 0 |
| qsort | 0.8089 | 0% | 19.7M | 57% | 0 |
| susan | 2.0246 | 0% | 20.5M | 85% | 0 |
| dijkstra | 1.5486 | 0% | 36.7M | 86% | 0 |

Fault injection: 4/4 detected, 100% detection rate, 0% IPC impact.

---

## Results Files

Pre-computed results are saved in `oir_results/`:
- `oir_results/all_results.txt` — clean run results for all 4 benchmarks
- `oir_results/fault_injection_results.txt` — fault injection results for all 4 benchmarks

---

## References

- Waser et al., "FAULTLESS," DIMVA 2025. https://doi.org/10.1007/978-3-031-97623-0_18
- gem5 Simulator: https://www.gem5.org
- MiBench: https://github.com/embecosm/mibench
