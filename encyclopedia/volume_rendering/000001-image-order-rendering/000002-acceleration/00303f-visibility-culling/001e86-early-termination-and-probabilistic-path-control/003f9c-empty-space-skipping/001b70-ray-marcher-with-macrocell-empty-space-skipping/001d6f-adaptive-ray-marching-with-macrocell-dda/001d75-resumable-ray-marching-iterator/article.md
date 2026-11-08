# Resumable Ray Marching Iterator

Also called: OptiX7 Adaptive Ray Marching, Resumable Ray Marching

## Overview
The Resumable Ray Marching Iterator is a GPU-accelerated ray marching algorithm that integrates adaptive step sizing with macrocell-based empty-space skipping to optimize direct volume rendering. It solves the problem of excessive sampling in sparse volumes by dynamically adjusting the step size based on an opacity upper bound estimate for the current macrocell, allowing the renderer to skip empty or low-opacity regions efficiently. It differs from its parent, [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md), by explicitly supporting interruption and resumption of the ray march, enabling robust handling of missing data or memory constraints through a resumable state mechanism.

## How it works
The iterator traverses a volume using a 3D Digital Differential Analyzer (DDA) over macrocells. It supports two modes: adaptive sampling (enabled by a preprocessor macro) and fixed-step linear sampling. The core innovation is the ability to pause the traversal, store the state, and resume later, which is critical for streaming architectures or handling out-of-core data.

### Inputs and Outputs
*   **Inputs**: Ray origin `org`, ray direction `dir`, near plane `tMin`, far plane `tMax`, volume dimensions `dims`, macrocell spacing parameters (`macrocell_spacings_rcp`), and a base `step` size.
*   **Outputs**: Accumulated radiance (color and alpha) via a callback `body(t)` for each segment, or a final transmittance value.
*   **State**: The iterator maintains internal state to allow resumption, including the current position in the DDA and the next cell boundary offset.

### Data Structures
The iterator relies on specific payload structures to communicate state between the ray generation, intersection, and miss programs (in an OptiX context) or between kernel launches.

**SampleStreamingPayload**
Used to store the state of a ray that may be interrupted.
*   `pixel_index`: Index of the pixel being rendered.
*   `jitter`: Jitter value for the current sample.
*   `alpha`: Accumulated alpha.
*   `color`: Accumulated color.
*   If `ADAPTIVE_SAMPLING` is defined:
    *   `iter.cell`: Current macrocell coordinates.
    *   `iter.t_next`: Next traversal time.
*   Always present:
    *   `iter.next_cell_begin`: Offset from `tMin` to the next cell boundary.

**RadiancePayload**
*   `alpha`: Float, initialized to `0.f`.
*   `color`: `vec3f`, initialized to `0.f`.
*   `rng`: Pointer to random number generator state, initialized to `nullptr`.

**ShadowPayload**
*   `alpha`: Float, initialized to `0.f`.
*   `rng`: Pointer to random number generator state, initialized to `nullptr`.

### Algorithm Steps

#### 1. Initialization
If `ADAPTIVE_SAMPLING` is defined:
1.  Initialize the DDA iterator with `DDAIter::init(m_org, m_dir, tMin, tMax, dims)`.
2.  Compute macrocell coordinates:
    $$ m\_org = org \times macrocell\_spacings\_rcp $$
    $$ m\_dir = dir \times macrocell\_spacings\_rcp $$

If `ADAPTIVE_SAMPLING` is not defined:
1.  Initialize `t.x = max(tMin + next_cell_begin, tMin)`.
2.  Initialize `t.y = min(t.x + step, tMax)`.

#### 2. Adaptive Ray Marching Loop
If `ADAPTIVE_SAMPLING` is defined, the iterator traverses macrocells using DDA. For each macrocell interval $[t_0, t_1]$:

1.  **Opacity Check**: Calculate the opacity upper bound $r$ for the current macrocell.
    $$ r = opacityUpperBound(self, cell) $$
2.  **Empty Space Skip**: If $|r| \le \epsilon$ (where $\epsilon$ is a small float epsilon), the macrocell is considered empty. Return `true` to continue the DDA to the next cell.
3.  **Step Size Calculation**: Calculate the adaptive step size $ss$.
    $$ ss = sample\_size\_scaler(adaptiveSamplingRate(step, r), t_0, t_1) $$
    The `sample_size_scaler` ensures the step size fits an integer number of steps within the interval:
    $$ N = \lfloor (t_1 - t_0) / ss \rfloor + 1 $$
    $$ ss_{final} = (t_1 - t_0) / N $$
4.  **Marching Loop**:
    *   Initialize $t.x = t_0$.
    *   Initialize $t.y = \min(t_1, t_0 + ss_{final})$.
    *   While $t.y > t.x$:
        *   Set `DDAIter::next_cell_begin = t.y - tMin`.
        *   Call `body(t)` to process the segment $[t.x, t.y]$.
        *   Update $t.x = t.y$.
        *   Update $t.y = \min(t.x + ss_{final}, t_1)$.

#### 3. Non-Adaptive Ray Marching Loop
If `ADAPTIVE_SAMPLING` is not defined, the iterator marches linearly from `tMin` to `tMax`.

1.  Initialize $t.x = \max(tMin + next\_cell\_begin, tMin)$.
2.  Initialize $t.y = \min(t.x + step, tMax)$.
3.  While $t.y > t.x$:
    *   Set `next_cell_begin = t.y - tMin`.
    *   Call `body(t)`.
    *   Update $t.x = t.y$.
    *   Update $t.y = \min(t.x + step, tMax)$.
4.  Set `next_cell_begin = float_large` and return.

#### 4. Resumability Check
To determine if the ray march can be resumed in a subsequent pass:
*   If `ADAPTIVE_SAMPLING` is defined: Return `DDAIter::resumable(m_dir, tMin, tMax, dims)`.
*   If `ADAPTIVE_SAMPLING` is not defined: Return `tMin + next_cell_begin < tMax`.

#### 5. Shadow Ray Transmittance
For shadow rays, the iterator creates a `ShadowPayload` with a random number generator. It packs the pointer to the shadow payload into `u0` and `u1` using `packPointer`. It then calls `optixTrace` with the following parameters:
*   Traversable: From `optixLaunchParams`.
*   Origin: `org`.
*   Direction: `dir`.
*   `tmin`: `0.f`.
*   `tmax`: `float_large`.
*   `time`: `0.0f`.
*   `mask`: `255`.
*   `flag`: `OPTIX_RAY_FLAG_DISABLE_ANYHIT`.
*   `rayType`: `SHADOW_RAY_TYPE`.
*   `sbtStride`: `RAY_TYPE_COUNT`.
*   `missIndex`: `SHADOW_RAY_TYPE`.
*   `u0`, `u1`: Packed payload pointer.
The function returns $1.0 - shadow.alpha$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raymarching_iterator(org, dir, tMin, tMax, dims, macrocell_spacings_rcp, step, body, adaptive_sampling=True):
    """
    Resumable Ray Marching Iterator.
    Returns True if the ray is complete, False if it needs to be resumed.
    """
    next_cell_begin = 0.0
    
    if adaptive_sampling:
        # Initialize DDA
        dda_iter = DDAIter()
        dda_iter.init(org * macrocell_spacings_rcp, dir * macrocell_spacings_rcp, tMin, tMax, dims)
        
        while True:
            # Get next macrocell interval [t0, t1]
            t0, t1, cell = dda_iter.next()
            if t0 is None:
                break # DDA finished
            
            # Calculate opacity upper bound
            r = opacity_upper_bound(cell)
            
            # Skip empty cells
            if abs(r) <= float_epsilon:
                continue
            
            # Calculate adaptive step size
            raw_ss = adaptive_sampling_rate(step, r)
            ss = sample_size_scaler(raw_ss, t0, t1)
            
            # March within the cell
            t_x = t0
            t_y = min(t1, t0 + ss)
            
            while t_y > t_x:
                next_cell_begin = t_y - tMin
                body(t_x, t_y)
                t_x = t_y
                t_y = min(t_x + ss, t1)
            
            # Check resumability
            if not dda_iter.resumable(dir, tMin, tMax, dims):
                return True
    else:
        # Non-adaptive linear marching
        t_x = max(tMin + next_cell_begin, tMin)
        t_y = min(t_x + step, tMax)
        
        while t_y > t_x:
            next_cell_begin = t_y - tMin
            body(t_x, t_y)
            t_x = t_y
            t_y = min(t_x + step, tMax)
        
        next_cell_begin = float_large
        return True

def sample_size_scaler(raw_ss, t0, t1):
    """
    Normalizes step size to fit an integer number of steps within [t0, t1].
    """
    interval = t1 - t0
    if interval <= 0:
        return 0.0
    n = floor(interval / raw_ss) + 1
    return interval / n

def compute_transmittance(org, dir):
    """
    Computes transmittance using a shadow ray.
    """
    shadow_payload = ShadowPayload()
    shadow_payload.alpha = 0.0
    shadow_payload.rng = create_rng()
    
    u0, u1 = pack_pointer(shadow_payload)
    
    # Call OptiX trace
    optix_trace(
        traversable=optix_launch_params.traversable,
        org=org,
        dir=dir,
        tmin=0.0,
        tmax=float_large,
        time=0.0,
        mask=255,
        flag=OPTIX_RAY_FLAG_DISABLE_ANYHIT,
        ray_type=SHADOW_RAY_TYPE,
        sbt_stride=RAY_TYPE_COUNT,
        miss_index=SHADOW_RAY_TYPE,
        u0=u0,
        u1=u1
    )
    
    return 1.0 - shadow_payload.alpha
```

### Parameters
*   `next_cell_begin`: A member variable tracking the offset from `tMin` to the next cell boundary, used for resuming the march.
*   `ADAPTIVE_SAMPLING`: A preprocessor macro that must be defined to enable adaptive sampling logic. If not defined, the code compiles in fixed-step mode. [S2, unofficial]

### Edge Cases and Pitfalls
*   **Precision**: The opacity upper bound check uses a float epsilon. If the epsilon is too large, valid data may be skipped; if too small, empty space may not be skipped efficiently.
*   **Boundary Conditions**: The `sample_size_scaler` function ensures that the step size divides the interval evenly, preventing overshoot at the end of a macrocell.
*   **Resumability**: The resumability check depends on the DDA state. If the DDA cannot be resumed (e.g., due to hardware limitations or specific traversal states), the iterator must handle this gracefully, potentially falling back to a non-resumable mode or erroring out. [S1, unofficial]

## Subtopics

### [Woodcock Collision Estimation (Standard)](3331-woodcock-collision-estimation-standard.md)
A ray marching technique for volume rendering that uses Woodcock collision estimation (delta tracking) to skip empty space, avoiding the need for spatial acceleration structures for sampling but requiring a global maximum density estimate. It differs from sibling methods like [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md) by avoiding the need for spatial acceleration structures for sampling, instead relying on probabilistic step sizing based on a global maximum density estimate.

### [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md)
Adaptive Ray Marching with Voxel Skipping is a visibility culling technique for direct volume rendering that accelerates ray marching by dynamically adjusting the step size based on local density. It solves the problem of excessive sampling in large empty volumes by using a low-precision traversal algorithm for empty regions and switching to a high-precision algorithm when approaching data. It differs from standard empty-space skipping by dynamically adjusting precision based on proximity to data, rather than using a static skip distance or hierarchical structure like [Adaptive Ray Marching with Macrocell DDA](3869-adaptive-ray-marching-with-macrocell-dda.md).

### [Voxel Bricks](5794-voxel-bricks.md)
Voxel Bricks is a hierarchical culling method for volume rendering that optimizes ray marching by grouping voxels into fixed-size $8 \times 8 \times 8$ blocks called "bricks". It solves the problem of excessive traversal steps in sparse voxel grids by maintaining a secondary "brickgrid" that references only non-empty bricks, allowing the renderer to skip entire regions of empty space. It differs from [Adaptive Ray Marching with Voxel Skipping](3870-adaptive-ray-marching-with-voxel-skipping.md) by using a fixed hierarchical depth (two levels) and from [Block Predicate Filtering](9106-block-predicate-filtering.md) by explicitly storing empty bricks as null references in the grid rather than using bit-packed predicates.

### [Block Predicate Filtering](9106-block-predicate-filtering.md)
A culling technique that evaluates a user-defined predicate on each grid block using affine arithmetic to determine if the block's domain can possibly contain a zero of the scalar field, skipping blocks that cannot before scalar evaluation.

### Ray-Guided Streaming with Interrupt/Resume
A method for handling missing GPU memory by stopping ray tracing when data is absent, communicating this to the CPU for loading, and resuming the ray at the last position in subsequent passes. [S3, unofficial]

**Definition**: This variant extends the resumable iterator to support out-of-core rendering. When a ray encounters missing data, the traversal is halted, and the state is communicated to the CPU. The CPU loads the required data, and the ray tracing resumes from the last known position. [S3, unofficial]

**Steps**:
1.  Start ray tracing. If data is missing, stop the ray and communicate this to the CPU. [S3, unofficial]
2.  The CPU initiates loading of the missing data. [S3, unofficial]
3.  The algorithm resumes ray tracing at the last ray position. [S3, unofficial]
4.  Block terminated pixels with the Z-Buffer to treat only active rays, relying on z-cull and early-Z GPU features to prevent their execution. [S3, unofficial]
5.  Maintain ray status information in two Multiple Render Targets (MRTs): preliminary output color accumulation, missing node ID, and current marching position. [S3, unofficial]
6.  Download missing IDs to the CPU so it can adapt the data structure. [S3, unofficial]
7.  In the next rendering pass, initiate with a proxy surface and use the MRTs as input textures; earlyZ ensures only active pixels execute the fragment shader. [S3, unofficial]

**Properties**: Loaded data is directly traversed by the ray, and no data lying outside the frustum will ever be loaded, even though this was not explicitly tested for it. Furthermore, the CPU does not need to track the needed LODs; the rays themselves indicate through their traversal which precision is needed in each area of the screen. [S3, unofficial]

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of samples. Adaptive sampling reduces $N$ significantly in sparse volumes by skipping empty macrocells.
*   **Memory Complexity**: $O(1)$ per ray for the iterator state, plus the memory for the volume data and acceleration structures (macrocell grid).
*   **Measured Performance**: Performance gains depend on the sparsity of the volume. In dense volumes, the overhead of macrocell traversal and opacity estimation may slightly reduce performance compared to fixed-step marching. In sparse volumes, significant speedups are achieved by skipping empty regions. [S2, unofficial]
*   **Strengths**: Efficiently handles sparse volumes; supports resumption for streaming and out-of-core rendering; integrates well with GPU ray tracing APIs like OptiX.
*   **Limitations**: Requires a macrocell grid structure; adaptive step sizing adds computational overhead for opacity estimation; resumability adds complexity to the rendering pipeline.
*   **When to Use**: Ideal for large, sparse volumes where empty space skipping is beneficial, and when out-of-core rendering or memory constraints require interruptible ray marching.
*   **When Not to Use**: For dense volumes where the overhead of macrocell traversal outweighs the benefits of skipping; when simple fixed-step marching is sufficient and resumability is not needed.

## Background
The Resumable Ray Marching Iterator is part of the Open Volume Renderer project by VIDILabs. It is implemented in the `open-volume-renderer` repository and is designed to work with the OptiX API. The method builds on the principles of adaptive ray marching and macrocell-based empty-space skipping, adding support for resumption to enable streaming and out-of-core rendering scenarios. [S1, unofficial][S2, unofficial]

The concept of resumable ray marching for handling missing data was introduced in "Ray-Guided Streaming for Efficient and Detailed Voxel Rendering" (CNLE09), which proposed a method to stop ray tracing when data is missing, load the data on the CPU, and resume the ray in subsequent passes. [S3, unofficial]

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S3] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
