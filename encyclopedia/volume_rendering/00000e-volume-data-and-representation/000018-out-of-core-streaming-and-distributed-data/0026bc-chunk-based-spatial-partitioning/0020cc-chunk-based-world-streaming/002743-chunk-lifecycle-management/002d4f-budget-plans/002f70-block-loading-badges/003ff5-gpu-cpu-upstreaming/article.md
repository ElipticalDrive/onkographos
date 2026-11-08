# GPU-CPU Upstreaming

Also called: Subvolume Priority Loading

## Overview
GPU-CPU Upstreaming is a chunk management technique that prioritizes the transfer of volume subvolumes from CPU to GPU memory based on their immediate visibility during ray-casting [S1, unofficial]. It solves the problem of inefficient memory usage by ensuring that only the most actively sampled subvolumes reside in VRAM, thereby reducing runtime memory requirements and enabling higher resolution rendering [S1, unofficial]. It differs from its parent [Block Loading Badges](6011-block-loading-badges.md) by using ray-casting step counters for priority rather than resident count movement, and from [GPU-CPU Data Synchronization](6012-gpu-cpu-data-synchronization.md) by focusing on initial loading priority rather than bidirectional data consistency [S1, unofficial][S2, unofficial].

## How it works
The method determines subvolume priority by tracking ray-casting activity and uploads data accordingly.

**Inputs:**
- Ray-casting step data (current subvolume hit, accumulated step size $a$).
- Subvolume counter map (associating subvolumes to priority scores).

**Outputs:**
- Updated subvolume priority list.
- GPU memory containing loaded subvolumes.

**Data Structures:**
- **Counter Map:** A hash map or array mapping subvolume IDs to floating-point priority scores.
- **Priority List:** A sorted list of subvolume IDs ordered by score descending.
- **Rendering Target:** A small 2D texture (e.g., $24 \times 16$ pixels) used to accumulate counter updates via interlocked memory operations [S1, unofficial].

**Steps:**
1. **Identify Needed Subvolumes:** During ray-casting, for each step, determine which subvolume is currently hit by the ray [S1, unofficial].
2. **Update Counters:** Increase the counter associated with the hit subvolume by the value $\frac{1}{a + \epsilon}$, where $a$ is the accumulated step size along the ray and $\epsilon$ is a small constant to prevent division by zero [S1, unofficial].
3. **Minimize Contention:** To avoid interlocked global memory access conflicts during counter updates, run a separate rendering process for this task. Use a small texture (e.g., $24 \times 16$ pixels) as the rendering target to accumulate these updates efficiently [S1, unofficial].
4. **Sort Priorities:** Order the subvolume counters decreasingly to create a priority list [S1, unofficial].
5. **Load Subvolumes:** Load the associated subvolumes into GPU memory according to the priority list [S1, unofficial].

**Parameters:**
- $\epsilon$: Small constant for numerical stability. Value not given in the sources.
- Texture Size: $24 \times 16$ pixels in the implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $a=0$, the term $\frac{1}{a+\epsilon}$ becomes undefined. The parameter $\epsilon$ must be chosen to handle this [S1, unofficial].
- **Memory Contention:** Without the separate rendering process for counter updates, interlocked global memory accesses may cause consistency issues or performance degradation [S1, unofficial].

## Subtopics

### GPU-CPU Data Synchronization
Synchronizes CPU-side volume data with GPU-side buffers, called every frame or every few frames [S2, unofficial]. It uses an operation enum `op` (`DNmemOp`) to control direction: `DN_READ` queries GPU for visible chunks, `DN_WRITE` uploads modified voxel data, and `DN_READ_WRITE` performs both [S2, unofficial]. It supports `lightingSplit`, an integer specifying the number of frames to split lighting calculations over (e.g., 5), updating $1/N$ of chunks per frame to increase performance [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Sorting counters takes $O(N \log N)$ where $N$ is the number of subvolumes. Loading is proportional to the number of loaded subvolumes.
- **Memory Complexity:** Reduces GPU runtime memory requirements by only loading active subvolumes [S1, unofficial].
- **Performance:** Enables significantly finer resolution rendering due to reduced memory footprint [S1, unofficial].
- **Strengths:** Efficient memory usage; prioritizes actively used data.
- **Limitations:** Requires additional rendering pass for counter updates; sorting overhead.

## Background
Originated in the context of vector-to-closest-point octree surface ray-casting [S1, unofficial]. The technique was developed to address memory constraints in high-resolution volume rendering.

## Sources
[S1] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
[S2] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
