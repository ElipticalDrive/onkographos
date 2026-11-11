# Resumable Ray Marching Iterator

Also called: OptiX7 Adaptive Ray Marching, Resumable Ray Marching

## Overview
The Resumable Ray Marching Iterator is a GPU-accelerated ray marching algorithm that traverses a volume using a 3D Digital Differential Analyzer (DDA) over macrocells, adapting the sampling step size based on local opacity estimates to skip empty regions [S1, S2, unofficial]. It extends [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md) by introducing a resumable state mechanism that allows the traversal to be paused and resumed across multiple rendering passes, enabling support for out-of-core data streaming and dynamic memory management [S1, S2, S3, unofficial]. Unlike standard adaptive marchers that complete a ray in a single kernel launch, this method maintains traversal state in pixel-local payloads, allowing the CPU to intervene for data loading or the GPU to yield execution [S3, unofficial].

## How it works
The method operates in two modes: adaptive sampling (enabled by the `ADAPTIVE_SAMPLING` macro) and linear sampling. It relies on a macrocell-based DDA to traverse the volume hierarchy and a payload structure to store per-ray state.

### Inputs and Outputs
**Inputs:**
- Ray origin `org` and direction `dir`.
- Ray bounds `tMin` and `tMax`.
- Volume dimensions `dims`.
- Macrocell spacing parameters `macrocell_spacings_rcp` (reciprocal spacing).
- Base step size `step` (for non-adaptive mode).
- Adaptive sampling rate function `adaptiveSamplingRate`.
- Opacity upper bound estimation function `opacityUpperBound`.
- Body function `body(t)` which processes the segment `[t.x, t.y]`.

**Outputs:**
- Accumulated radiance and opacity stored in `SampleStreamingPayload` (or `RadiancePayload`/`ShadowPayload` for specific ray types).
- Updated traversal state for resumption.

### Data Structures
The payload structures store the ray state and traversal context.

**SampleStreamingPayload** (Adaptive Mode):
- `pixel_index`: Integer identifier for the pixel.
- `jitter`: Floating-point jitter value.
- `alpha`: Accumulated opacity (float).
- `color`: Accumulated color (vec3f).
- `iter.cell`: Current macrocell coordinates (vec3i).
- `iter.t_next`: Next traversal time coordinate (vec2f).
- `iter.next_cell_begin`: Offset from `tMin` to the next cell boundary (float).

**RadiancePayload** (Standard):
- `alpha`: Accumulated opacity, initialized to `0.f`.
- `color`: Accumulated color, initialized to `0.f`.
- `rng`: Pointer to random number generator state, initialized to `nullptr`.

**ShadowPayload**:
- `alpha`: Accumulated opacity, initialized to `0.f`.
- `rng`: Pointer to random number generator state, initialized to `nullptr`.

### Algorithm Steps

#### 1. Initialization
If `ADAPTIVE_SAMPLING` is defined:
1. Compute macrocell coordinates for the ray origin and direction:
   $$ m\_org = org \times macrocell\_spacings\_rcp $$
   $$ m\_dir = dir \times macrocell\_spacings\_rcp $$
2. Initialize the DDA iterator: `DDAIter::init(m_org, m_dir, tMin, tMax, dims)`.
3. Initialize `next_cell_begin` to `0.0` (or `tMin` offset).

If `ADAPTIVE_SAMPLING` is not defined:
1. Initialize `t.x = max(tMin + next_cell_begin, tMin)`.
2. Initialize `t.y = min(t.x + step, tMax)`.

#### 2. Adaptive Ray Marching Loop
The core loop uses a lambda function to process each macrocell interval `[t0, t1]` returned by the DDA.

1. **DDA Traversal**: Loop while `DDAIter::next(...)` returns true.
2. **Cell Processing**: For each cell interval `[t0, t1]`:
   a. Calculate the opacity upper bound `r = opacityUpperBound(self, cell)`.
   b. **Empty Space Skip**: If `fabsf(r) <= float_epsilon`, the cell is empty. Return `true` from the lambda to continue to the next DDA cell.
   c. **Step Size Calculation**: Calculate the adaptive step size `ss`.
      $$ ss = sample\_size\_scaler(adaptiveSamplingRate(step, r), t0, t1) $$
      The `sample_size_scaler` ensures an integer number of steps fit within `[t0, t1]`:
      $$ N = \lfloor (t1 - t0) / ss \rfloor + 1 $$
      $$ effective\_step = (t1 - t0) / N $$
   d. **Inner Loop**: Initialize `t.x = t0` and `t.y = min(t1, t0 + ss)`.
   e. While `t.y > t.x`:
      i. Set `next_cell_begin = t.y - tMin`. This value is critical for resumption.
      ii. Call `body(t)` to process the segment.
      iii. Update `t.x = t.y`.
      iv. Update `t.y = min(t.x + ss, t1)`.

#### 3. Non-Adaptive Ray Marching Loop
If `ADAPTIVE_SAMPLING` is not defined:
1. Loop while `t.y > t.x`:
   a. Set `next_cell_begin = t.y - tMin`.
   b. Call `body(t)`.
   c. Update `t.x = t.y`.
   d. Update `t.y = min(t.x + step, tMax)`.
2. Set `next_cell_begin = float_large` (indicating completion or pause point).
3. Return.

#### 4. Resumability Check
To determine if the ray can be resumed in a subsequent pass:
- If `ADAPTIVE_SAMPLING`: Return `DDAIter::resumable(m_dir, tMin, tMax, dims)`.
- Else: Return `tMin + next_cell_begin < tMax`.

#### 5. Shadow Ray Execution (Optional Variant)
For shadow rays, the iterator may construct a `ShadowPayload` and invoke `optixTrace`:
1. Pack pointer to shadow data into `u0`, `u1` using `packPointer`.
2. Call `optixTrace` with:
   - Traversable from `optixLaunchParams`.
   - Ray origin `org`, direction `dir`.
   - `tmin = 0.f`, `tmax = float_large`.
   - `time = 0.0f`, `mask = 255`.
   - `flag = OPTIX_RAY_FLAG_DISABLE_ANYHIT`.
   - `rayType = SHADOW_RAY_TYPE`.
   - `sbtStride = RAY_TYPE_COUNT`, `missIndex = SHADOW_RAY_TYPE`.
3. Return `1.f - shadow.alpha`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raymarching_iterator(org, dir, tMin, tMax, dims, macrocell_spacings_rcp, step, 
                         adaptiveSamplingRate, opacityUpperBound, body, 
                         ADAPTIVE_SAMPLING):
    """
    Resumable ray marching iterator.
    Returns: (payload, next_cell_begin, is_resumable)
    """
    # Initialize payload
    payload = {
        'alpha': 0.0,
        'color': vec3f(0, 0, 0),
        'iter': {
            'cell': vec3i(0, 0, 0),
            't_next': vec2f(0, 0),
            'next_cell_begin': 0.0
        }
    }

    if ADAPTIVE_SAMPLING:
        # 1. Initialize DDA
        m_org = org * macrocell_spacings_rcp
        m_dir = dir * macrocell_spacings_rcp
        dda_iter = DDAIter()
        dda_iter.init(m_org, m_dir, tMin, tMax, dims)
        
        # 2. Adaptive Loop
        while dda_iter.next():
            t0, t1 = dda_iter.get_interval()
            cell = dda_iter.get_cell()
            
            # 3. Opacity Check
            r = opacityUpperBound(cell)
            if abs(r) <= float_epsilon:
                continue  # Skip empty cell
            
            # 4. Step Size
            raw_ss = adaptiveSamplingRate(step, r)
            ss = sample_size_scaler(raw_ss, t0, t1)
            
            # 5. Inner Loop
            t_x = t0
            t_y = min(t1, t0 + ss)
            
            while t_y > t_x:
                payload['iter']['next_cell_begin'] = t_y - tMin
                body(t_x, t_y, payload)
                t_x = t_y
                t_y = min(t_x + ss, t1)
        
        is_resumable = dda_iter.resumable(dir, tMin, tMax, dims)
        
    else:
        # Non-Adaptive Loop
        t_x = max(tMin + payload['iter']['next_cell_begin'], tMin)
        t_y = min(t_x + step, tMax)
        
        while t_y > t_x:
            payload['iter']['next_cell_begin'] = t_y - tMin
            body(t_x, t_y, payload)
            t_x = t_y
            t_y = min(t_x + step, tMax)
            
        payload['iter']['next_cell_begin'] = float_large
        is_resumable = (tMin + payload['iter']['next_cell_begin'] < tMax)

    return payload, payload['iter']['next_cell_begin'], is_resumable

def sample_size_scaler(ss, t0, t1):
    """
    Normalizes step size to fit integer number of steps in interval.
    """
    interval_len = t1 - t0
    if interval_len <= 0:
        return 0.0
    N = floor(interval_len / ss) + 1
    return interval_len / N
```

### Edge Cases and Pitfalls
- **Precision**: The check `fabsf(r) <= float_epsilon` relies on a small epsilon value. If `r` is very close to zero but not exactly zero, the cell might be skipped incorrectly or processed unnecessarily. [S2, unofficial]
- **Resumption State**: The `next_cell_begin` variable must be correctly saved and restored between passes. Failure to do so results in re-sampling or skipping segments. [S1, unofficial]
- **Macrocell Boundaries**: The DDA must correctly handle ray exits from the volume bounds (`tMax`). The `sample_size_scaler` ensures steps do not overshoot `t1`. [S2, unofficial]
- **ADAPTIVE_SAMPLING Macro**: If `ADAPTIVE_SAMPLING` is not defined, the code path for adaptive sampling is excluded, and the compiler may error if dependent symbols are used. [S2, unofficial]

## Subtopics

### [Woodcock Collision Estimation (Standard)](3331-woodcock-collision-estimation-standard.md)
A ray marching technique for volume rendering that uses Woodcock collision estimation (delta tracking) to skip empty space, avoiding the need for spatial acceleration structures for sampling but requiring a global maximum density estimate. It differs from sibling methods like [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md) by avoiding the need for spatial acceleration structures for sampling, instead relying on probabilistic step sizing based on a global maximum density estimate.

### [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md)
Adaptive Ray Marching with Voxel Skipping is a visibility culling technique for direct volume rendering that accelerates ray marching by dynamically adjusting the step size based on local density. It solves the problem of excessive sampling in large empty volumes by using a low-precision traversal algorithm for empty regions and switching to a high-precision algorithm when approaching data. It differs from standard empty-space skipping by dynamically adjusting precision based on proximity to data, rather than using a static skip distance or hierarchical structure like [Adaptive Ray Marching with Macrocell DDA](3869-adaptive-ray-marching-with-macrocell-dda.md).

### [Voxel Bricks](5794-voxel-bricks.md)
Voxel Bricks is a hierarchical culling method for volume rendering that optimizes ray marching by grouping voxels into fixed-size $8 \times 8 \times 8$ blocks called "bricks". It solves the problem of excessive traversal steps in sparse voxel grids by maintaining a secondary "brickgrid" that references only non-empty bricks, allowing the renderer to skip entire regions of empty space. It differs from [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md) by using a fixed hierarchical depth (two levels) and from [Block Predicate Filtering](9106-block-predicate-filtering.md) by explicitly storing empty bricks as null references in the grid rather than using bit-packed predicates.

### [Block Predicate Filtering](9106-block-predicate-filtering.md)
Block Predicate Filtering is a culling technique that evaluates a user-defined predicate on each grid block using affine arithmetic to determine if the block's domain can possibly contain a zero of the scalar field, skipping blocks that cannot before scalar evaluation. It builds on [Resumable Ray Marching Iterator](1505-resumable-ray-marching-iterator.md) by providing a coarse-grained filtering mechanism that reduces the number of expensive scalar field evaluations required during ray marching. Compared to naive block mask extraction, it filters blocks before scalar evaluation, reducing computation.

### Ray-Guided Streaming with Interrupt/Resume
A method for handling missing GPU memory by stopping ray tracing when data is absent, communicating this to the CPU for loading, and resuming the ray at the last position in subsequent passes [S3, unofficial].

**Definition:**
A technique where ray tracing is paused when GPU memory is insufficient, allowing the CPU to load missing data blocks before resuming the ray from its last position [S3, unofficial].

**Steps:**
1. Start ray tracing; if data is missing, stop the ray and communicate this to the CPU [S3, unofficial].
2. CPU initiates loading of missing data [S3, unofficial].
3. Algorithm resumes ray tracing at the last ray position [S3, unofficial].
4. Block terminated pixels with the Z-Buffer to treat only active rays, relying on z-cull and early-Z GPU features to prevent their execution [S3, unofficial].
5. Maintain ray status information in two Multiple Render Targets (MRTs): preliminary output color accumulation, missing node ID, and current marching position [S3, unofficial].
6. Download missing IDs to CPU so it can adapt the data structure [S3, unofficial].
7. In the next rendering pass, initiate with a proxy surface and use the MRTs as input textures; earlyZ ensures only active pixels execute the fragment shader [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the volume density and the effectiveness of the opacity upper bound estimation. Adaptive sampling reduces the number of `body` calls in empty regions, improving performance. The DDA traversal is $O(N)$ where $N$ is the number of macrocells intersected.
- **Memory Complexity**: $O(1)$ per ray for the iterator state, plus $O(P)$ for the payload storage where $P$ is the number of active pixels. The macrocell structure requires additional memory proportional to the volume dimensions.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales well with resolution due to the parallel nature of ray tracing. The resumable nature allows for handling larger-than-memory volumes by streaming data, at the cost of increased latency due to CPU-GPU communication and multiple passes.
- **Strengths**: Efficient empty space skipping via macrocell DDA; support for out-of-core rendering via resumption; flexible step sizing.
- **Limitations**: Requires careful management of resumption state; overhead of DDA traversal; potential for precision issues in opacity estimation.
- **When to use**: When rendering large volumes with significant empty space, or when GPU memory is insufficient to hold the entire volume.
- **When not to use**: For small volumes that fit entirely in GPU memory, simpler linear or adaptive marchers may be more efficient due to lower overhead.

## Background
The Resumable Ray Marching Iterator is part of the OptiX7 Adaptive Ray Marching implementation, as seen in the Open Volume Renderer project [S2, unofficial]. It builds upon the principles of adaptive ray marching and macrocell-based traversal [S1, S2, unofficial]. The concept of resumable ray tracing for out-of-core rendering was introduced in "Ray-Guided Streaming for Efficient and Detailed Voxel Rendering" [S3, unofficial], which demonstrated the feasibility of pausing and resuming rays to handle memory constraints. The specific implementation details, including the payload structures and DDA integration, are derived from the instantvnr and open-volume-renderer repositories [S1, S2, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S3] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
