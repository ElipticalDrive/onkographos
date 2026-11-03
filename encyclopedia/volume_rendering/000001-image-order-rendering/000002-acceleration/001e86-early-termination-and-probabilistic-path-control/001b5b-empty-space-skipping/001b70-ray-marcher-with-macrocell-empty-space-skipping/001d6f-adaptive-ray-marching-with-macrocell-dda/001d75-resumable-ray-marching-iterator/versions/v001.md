# Resumable Ray Marching Iterator

Also called: OptiX7 Adaptive Ray Marching, Resumable Ray Marching

## Overview
The Resumable Ray Marching Iterator is a GPU-based ray traversal method designed to handle out-of-core volume rendering by pausing ray execution when data is missing from GPU memory and resuming it in subsequent passes [S2]. It builds on [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md) by adding state persistence via Multiple Render Targets (MRTs) and Z-buffer culling to ensure only active rays are processed during data loading [S2]. Unlike [Network Ray Marching Iterator](1510-network-ray-marching-iterator.md) which may rely on different streaming heuristics, or [Macrocell Grid Traversal with Majorant Densities](1528-macrocell-grid-traversal-with-majorant-densities.md) which focuses on density bounds, this method explicitly manages the CPU-GPU synchronization required for dynamic data loading [S2].

## How it works
### Inputs and Outputs
**Inputs:**
- Ray origin `org` and direction `dir`.
- Volume bounds `tMin`, `tMax`.
- Volume dimensions `dims`.
- Macrocell spacing (via `macrocell_spacings_rcp`).
- Base sample step `step`.
- Adaptive sampling rate parameter `adaptiveSamplingRate`.
- Missing data identifiers (node IDs) communicated from GPU to CPU.

**Outputs:**
- Accumulated color and alpha for the pixel.
- Updated ray state (position, accumulated color) for resumption.
- Missing node IDs for CPU-side data loading.

### Data Structures
The method relies on specific payload structures to store ray state and traversal context.

**SampleStreamingPayload**
Stores the state of a ray for resumption and output.
- `pixel_index`: Integer identifying the pixel.
- `jitter`: Floating point jitter value.
- `alpha`: Accumulated alpha (float).
- `color`: Accumulated color (vec3f).
- `iter.cell`: Current macrocell coordinates (vec3i, present if `ADAPTIVE_SAMPLING` is defined) [S1].
- `iter.t_next`: Next traversal position (vec2f, present if `ADAPTIVE_SAMPLING` is defined) [S1].
- `iter.next_cell_begin`: Offset from `tMin` to the next cell boundary (float, always present) [S1].

**RadiancePayload**
Used for primary ray tracing.
- `alpha`: Initialized to `0.f`.
- `color`: Initialized to `0.f`.
- `rng`: Pointer to random number generator state, initialized to `nullptr` [S3].

**ShadowPayload**
Used for shadow ray tracing.
- `alpha`: Initialized to `0.f`.
- `rng`: Pointer to random number generator state, initialized to `nullptr` [S3].

### Numbered Steps

1.  **Initialization**:
    - If `ADAPTIVE_SAMPLING` is defined, initialize the Digital Differential Analyzer (DDA) iterator: `DDAIter::init(m_org, m_dir, tMin, tMax, dims)` [S1].
    - If `ADAPTIVE_SAMPLING` is not defined, initialize `t.x = max(tMin + next_cell_begin, tMin)` and `t.y = min(t.x + step, tMax)` [S1].

2.  **Ray Marching Loop**:
    - **Adaptive Mode** (`ADAPTIVE_SAMPLING` defined):
        1.  Compute macrocell coordinates: `m_org = org * macrocell_spacings_rcp`, `m_dir = dir * macrocell_spacings_rcp` [S1].
        2.  Perform DDA traversal over macrocells using `dda::dda3` [S1].
        3.  For each macrocell interval `[t0, t1]`:
            - Calculate opacity upper bound `r = opacityUpperBound(self, cell)` [S1].
            - If `fabsf(r) <= float_epsilon`, the cell is empty; return `true` to continue DDA to the next cell [S1].
            - Calculate adaptive step size `ss` using `sample_size_scaler(adaptiveSamplingRate(step, r), t0, t1)` [S1].
            - Initialize `t = vec2f(t0, min(t1, t0 + ss))` [S1].
            - Loop while `t.y > t.x`:
                - Set `DDAIter::next_cell_begin = t.y - tMin` [S1].
                - Call `body(t)` to accumulate color/alpha for the segment `[t.x, t.y]` [S1].
                - Update `t.x = t.y` and `t.y = min(t.x + ss, t1)` [S1].
    - **Non-Adaptive Mode** (`ADAPTIVE_SAMPLING` not defined):
        1.  Loop while `t.y > t.x`:
            - Set `next_cell_begin = t.y - tMin` [S1].
            - Call `body(t)` for the segment `[t.x, t.y]` [S1].
            - Update `t.x = t.y` and `t.y = min(t.x + step, tMax)` [S1].
        2.  Set `next_cell_begin = float_large` and return [S1].

3.  **Resumability Check**:
    - If `ADAPTIVE_SAMPLING` is defined, the ray is resumable if `DDAIter::resumable(m_dir, tMin, tMax, dims)` returns true [S1].
    - If `ADAPTIVE_SAMPLING` is not defined, the ray is resumable if `tMin + next_cell_begin < tMax` [S1].

4.  **Handling Missing Data**:
    - If data is missing during traversal, stop the ray and communicate the missing node ID to the CPU [S2].
    - Maintain ray status (preliminary color, missing node ID, current position) in two MRTs [S2].
    - Download missing IDs to the CPU to initiate data loading [S2].

5.  **Resumption**:
    - In the next rendering pass, use a proxy surface to initiate rendering [S2].
    - Use the MRTs as input textures [S2].
    - Use Z-buffer culling (early-Z) to ensure only active pixels (where rays were paused) execute the fragment shader [S2].
    - Resume ray tracing at the last stored position [S2].

6.  **Shadow Ray Tracing**:
    - Create `ShadowPayload` with a new RNG.
    - Pack pointer to shadow into `u0`, `u1` using `packPointer` [S3].
    - Call `optixTrace` with:
        - Traversable from `optixLaunchParams`.
        - Origin `org`, direction `dir`.
        - `tmin=0.f`, `tmax=float_large`.
        - `time=0.0f`, `mask=255`.
        - `flag=OPTIX_RAY_FLAG_DISABLE_ANYHIT`.
        - `rayType=SHADOW_RAY_TYPE`.
        - `sbtStride=RAY_TYPE_COUNT`.
        - `missIndex=SHADOW_RAY_TYPE`.
        - `u0`, `u1` [S3].
    - Return `1.f - shadow.alpha` [S3].

### Formulas

**Sample Size Scaler**
Calculates the effective step size to fit an integer number of steps within an interval `[t0, t1]`:
$$ N = \lfloor (t1 - t0) / ss \rfloor + 1 $$
$$ \text{effective\_step} = (t1 - t0) / N $$
where `ss` is the initial adaptive step size estimate [S3].

**Resumability Condition (Non-Adaptive)**
$$ \text{resumable} = (tMin + next\_cell\_begin < tMax) $$
[S1]

### Parameters
- `ADAPTIVE_SAMPLING`: A preprocessor macro. Must be defined to enable adaptive sampling logic. If not defined, compilation fails with error "ADAPTIVE_SAMPLING is not defined" [S3].
- `next_cell_begin`: Member variable tracking the offset from `tMin` to the next cell boundary, used for resuming the march [S1].
- `adaptiveSamplingRate`: Function/parameter controlling the base adaptive step size based on opacity [S1].
- `step`: Base sample step size for non-adaptive mode [S1].

### Edge Cases and Pitfalls
- **Precision**: The use of `float_epsilon` for empty cell detection (`fabsf(r) <= float_epsilon`) requires careful consideration of floating-point precision in opacity bounds [S1].
- **Boundary Conditions**: The loop condition `t.y > t.x` and the clamping `min(t.y, t1)` or `min(t.y, tMax)` must handle cases where the interval is smaller than the step size [S1].
- **Data Loading Latency**: The method assumes the CPU can load data fast enough to keep up with rendering; otherwise, frames may be delayed or require multiple passes per frame [S2].
- **Z-Buffer Culling**: Relies on early-Z features; if early-Z is disabled or fails, inactive pixels may still execute, wasting resources [S2].

## Operating characteristics
- **Time Complexity**: Depends on the number of macrocells traversed and the adaptive step size. Adaptive sampling reduces steps in empty regions, improving performance [S1].
- **Memory Complexity**: Requires additional memory for MRTs to store ray state (color, alpha, position, missing ID) for all active pixels [S2].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with the number of active rays and the complexity of the volume data. More missing data leads to more passes, increasing total time [S2].
- **Strengths**: Handles out-of-core data efficiently; only loads data within the frustum; adaptive sampling reduces computation in empty regions [S2].
- **Limitations**: Requires multiple rendering passes for complete frames if data is missing; relies on CPU-GPU synchronization; complexity in managing ray state [S2].
- **When to use**: When volume data exceeds GPU memory capacity and dynamic loading is required [S2].
- **When not to use**: When all data fits in GPU memory and low latency is critical, as the overhead of state management and multiple passes may not be justified [S2].

## Background
The method originates from research on ray-guided streaming for efficient and detailed voxel rendering [S2]. It was implemented in the context of OptiX7-based volume renderers, as indicated by the alias "OptiX7 Adaptive Ray Marching" and the use of OptiX-specific features like `optixTrace` and SBT (Shader Binding Table) configuration [S3]. The approach addresses the challenge of rendering large volumetric datasets that cannot fit entirely in GPU memory by streaming data on-demand [S2].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26
[S3] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
