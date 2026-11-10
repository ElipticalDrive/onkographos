# Window/Level Volume Raycasting with Bricking

Also called: LQ/HQ Volume Rendering, Volume Raycasting with Window/Level and Bricking, Window/Level Volume Rendering

## Overview
Window/Level Volume Raycasting with Bricking is a direct volume rendering technique that combines spatial decomposition into fixed-size bricks with user-defined intensity windowing to accelerate the rendering of large medical datasets on standard PC hardware [S2, unofficial]. It addresses the performance bottleneck of accessing large, contiguous volume data by storing it in a bricked layout and traversing the bricks during ray marching, while allowing users to adjust tissue visibility via window/level parameters [S1, unofficial][S2, unofficial]. It differs from sibling methods like [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) by using a spatial brick layout rather than frequency-domain transforms [S2, unofficial].

## How it works
The method operates in two phases: data preparation (bricking and metadata extraction) and runtime raycasting (brick traversal, window/level mapping, and accumulation) [G].

### Inputs and Outputs
**Inputs:**
*   Raw volume data decomposed into bricks [S1, unofficial].
*   Window center ($gLevel$) and window width ($gWindow$) for intensity mapping [S1, unofficial].
*   Voxel spacing ($voxelSpacingBuffer$) for geometric accuracy [S1, unofficial].
*   Volume dimensions ($volumeWidth, volumeHeight, volumeDepth$) [S1, unofficial].
*   Bricking parameters: chunk size ($chunkSize$) and number of chunks ($numChunksX, numChunksY, numChunksZ$) [S1, unofficial].
*   Rendering mode flag ($usingHQ$): selects between Low Quality (LQ) or High Quality (HQ) parameter sets [S1, unofficial].
*   Cutting cube bounds ($xmin, xmax, ymin, ymax, zmin, zmax$) for clipping, defaulting to $-1.0$ and $1.0$ if null [S1, unofficial].

**Outputs:**
*   A rendered image buffer with accumulated color and opacity [S1, unofficial].

### Data Structures
1.  **Brick Layout:** The volume is decomposed into 3D bricks of size $chunkSize \times chunkSize \times chunkSize$ [S1, unofficial].
2.  **Metadata Buffer:** Contains voxel spacing and volume dimensions [S1, unofficial].
3.  **Parameter Sets:** Two sets of thresholds and constants are maintained, one for LQ and one for HQ, selected by the $usingHQ$ flag [S1, unofficial]:
    *   $gDensityForMarchSpaceSkipping$: Threshold to skip marching steps.
    *   $skipMultiplier$: Multiplier for step skipping.
    *   $subtleSurfaceThreshold$: Threshold for subtle surface detection.
    *   $surfaceThreshold$: Threshold for surface rendering.
    *   $maxSteps$: Maximum ray marching steps.
    *   $minGradientMagnitude$: Minimum gradient for calculation.
    *   $accumulatedThreshold$: Threshold for accumulated opacity to terminate early.
    *   $transmittanceThreshold$: Threshold for transmittance to terminate early.
    *   $gUseGradientLQ/HQ$: Boolean flag for gradient calculation.
    *   $gAmbient$: Ambient lighting intensity [S1, unofficial].

### Numbered Steps
1.  **Data Loading and Bricking:** Read the raw volume data and decompose it into bricks of size $chunkSize$, storing them indexed by chunk coordinates [S1, unofficial][S2, unofficial].
2.  **Ray Initialization:** For each pixel, compute the ray origin and direction in world space, then transform it into volume space [G]. Initialize ray parameters $t_{min}$ and $t_{max}$ based on the intersection with the volume bounding box and the cutting cube bounds ($xmin, xmax, \dots$) [S1, unofficial].
3.  **Brick Traversal:** Traverse the ray through the brick hierarchy. For each intersected brick, load the brick data into memory and compute the entry and exit points of the ray within the current brick [S2, unofficial].
4.  **Ray Marching and Window/Level Mapping:** For each step along the ray within the brick:
    *   **Addressing:** Compute the voxel coordinates and resolve the brick and local element offsets [S2, unofficial].
    *   **Sampling:** Interpolate the voxel value $V$ at the current position [S2, unofficial].
    *   **Window/Level Transformation:** Map the raw voxel value $V$ to an opacity $\alpha$ and color $C$ using the window/level parameters:
        $$
        \alpha = \begin{cases} 
        0 & \text{if } V \leq gLevel - \frac{gWindow}{2} \\
        \frac{V - (gLevel - \frac{gWindow}{2})}{gWindow} & \text{if } gLevel - \frac{gWindow}{2} < V < gLevel + \frac{gWindow}{2} \\
        1 & \text{if } V \geq gLevel + \frac{gWindow}{2}
        \end{cases}
        $$
        [G]
    *   **Gradient Calculation (Optional):** If $gUseGradient$ is true, compute the gradient $\nabla V$ using a 26-connected neighborhood [S2, unofficial].
    *   **Accumulation:** Update the accumulated color $C_{acc}$ and opacity $A_{acc}$ using front-to-back blending:
        $$
        C_{acc} = C_{acc} + C \cdot \alpha \cdot (1 - A_{acc})
        $$
        $$
        A_{acc} = A_{acc} + \alpha \cdot (1 - A_{acc})
        $$
    *   **Early Termination:** If $A_{acc} > accumulatedThreshold$ or $1 - A_{acc} < transmittanceThreshold$, terminate the ray [S1, unofficial].
5.  **Rendering:** Pass the final pixel color to the GPU for display via the render pipeline [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_ray_casting_bricks(volume_data, brick_layout, params, window_center, window_width):
    """
    Main rendering loop for Window/Level Volume Raycasting with Bricking.
    All coordinates are in voxel space. volume_data is a list of bricks,
    where each brick is a flat array of size chunkSize^3.
    """
    width = params['volumeWidth']
    height = params['volumeHeight']
    depth = params['volumeDepth']
    chunk_size = params['chunkSize']
    num_chunks_x = params['numChunksX']
    num_chunks_y = params['numChunksY']
    num_chunks_z = params['numChunksZ']
    is_hq = params['usingHQ']

    # LQ/HQ parameter selection
    density_skip = params['densityForMarchSpaceSkippingHQ'] if is_hq else params['densityForMarchSpaceSkippingLQ']
    skip_mult = params['skipMultiplierHQ'] if is_hq else params['skipMultiplierLQ']
    max_steps = params['maxStepsHQ'] if is_hq else params['maxStepsLQ']
    acc_thresh = params['accumulatedThresholdHQ'] if is_hq else params['accumulatedThresholdLQ']
    trans_thresh = params['transmittanceThresholdHQ'] if is_hq else params['transmittanceThresholdLQ']
    use_gradient = params['useGradientHQ'] if is_hq else params['useGradientLQ']
    min_grad_mag = params['minGradientMagnitudeHQ'] if is_hq else params['minGradientMagnitudeLQ']
    surface_thresh = params['surfaceThresholdHQ'] if is_hq else params['surfaceThresholdLQ']
    subtle_thresh = params['subtleSurfaceThresholdHQ'] if is_hq else params['subtleSurfaceThresholdLQ']
    ambient = params['gAmbient']

    # Cutting cube bounds
    xmin, xmax = params['xmin'], params['xmax']
    ymin, ymax = params['ymin'], params['ymax']
    zmin, zmax = params['zmin'], params['zmax']

    # Window/level bounds
    lower_bound = window_center - window_width / 2.0
    upper_bound = window_center + window_width / 2.0

    # Initialize output buffer
    image = [[[0.0, 0.0, 0.0] for _ in range(width)] for _ in range(height)]

    # Step size in voxel units
    step_size = 1.0

    for y in range(height):
        for x in range(width):
            # Standard orthographic ray generation [G]
            ray_origin = [0.0, 0.0, 0.0]
            ray_direction = [0.0, 0.0, -1.0]

            t = 0.0
            accumulated_color = [0.0, 0.0, 0.0]
            accumulated_alpha = 0.0
            steps_taken = 0

            while t < depth and accumulated_alpha < acc_thresh and (1.0 - accumulated_alpha) > trans_thresh and steps_taken < max_steps:
                # Current position in voxel space
                pos = [ray_origin[i] + ray_direction[i] * t for i in range(3)]

                # Check cutting cube bounds
                if not (xmin <= pos[0] <= xmax and ymin <= pos[1] <= ymax and zmin <= pos[2] <= zmax):
                    t += step_size
                    steps_taken += 1
                    continue

                # Address brick and voxel
                gx, gy, gz = [int(p) for p in pos]
                bx, by, bz = gx // chunk_size, gy // chunk_size, gz // chunk_size
                lx, ly, lz = gx % chunk_size, gy % chunk_size, gz % chunk_size

                # Compute brick index (row-major)
                brick_idx = bx * num_chunks_y * num_chunks_z + by * num_chunks_z + bz
                # Compute local index
                local_idx = lx + ly * chunk_size + lz * chunk_size * chunk_size

                # Sample voxel value
                voxel_value = volume_data[brick_idx][local_idx]

                # Window/Level mapping
                if voxel_value <= lower_bound:
                    alpha = 0.0
                    color = [0.0, 0.0, 0.0]
                elif voxel_value >= upper_bound:
                    alpha = 1.0
                    color = [1.0, 1.0, 1.0]
                else:
                    alpha = (voxel_value - lower_bound) / window_width
                    color = [alpha, alpha, alpha]

                # Gradient calculation (26-connected neighborhood) [G]
                gradient = [0.0, 0.0, 0.0]
                grad_mag = 0.0
                if use_gradient:
                    # Central difference approximation
                    gradient[0] = volume_data[brick_idx][local_idx + 1] - volume_data[brick_idx][local_idx - 1]
                    gradient[1] = volume_data[brick_idx][local_idx + chunk_size] - volume_data[brick_idx][local_idx - chunk_size]
                    gradient[2] = volume_data[brick_idx][local_idx + chunk_size * chunk_size] - volume_data[brick_idx][local_idx - chunk_size * chunk_size]
                    grad_mag = (gradient[0]**2 + gradient[1]**2 + gradient[2]**2)**0.5
                    if grad_mag < min_grad_mag:
                        gradient = [0.0, 0.0, 0.0]
                        grad_mag = 0.0

                # Simple shading based on gradient and ambient [G]
                if use_gradient and grad_mag >= min_grad_mag:
                    diffuse = min(1.0, grad_mag * 0.1)
                    color = [c * (ambient + diffuse) for c in color]

                # Accumulation (front-to-back)
                weight = alpha * (1.0 - accumulated_alpha)
                accumulated_color = [accumulated_color[i] + color[i] * weight for i in range(3)]
                accumulated_alpha += weight

                t += step_size
                steps_taken += 1

            image[y][x] = accumulated_color

    return image
```

### Parameters with Typical Values
*   $chunkSize$: Not given in the sources.
*   $gLevel$ (Window Center): Depends on the dataset (e.g., Hounsfield units for CT).
*   $gWindow$ (Window Width): Depends on the dataset.
*   $gDensityForMarchSpaceSkippingLQ/HQ$: Not given in the sources.
*   $skipMultiplierLQ/HQ$: Not given in the sources.
*   $maxStepsLQ/HQ$: Not given in the sources.
*   $accumulatedThresholdLQ/HQ$: Not given in the sources.
*   $transmittanceThresholdLQ/HQ$: Not given in the sources.

### Edge Cases and Pitfalls
*   **Pipeline Flushes:** Using `if-else` statements to check brick boundaries during addressing can cause CPU pipeline flushes on architectures with long instruction pipelines, significantly degrading performance [S2, unofficial].
*   **Boundary Conditions:** When a ray crosses a brick boundary, the addressing logic must correctly handle the transition between bricks to avoid accessing invalid memory [S2, unofficial].
*   **Precision:** Floating-point precision in ray marching and coordinate transformation can lead to artifacts, especially at large scales [G].

## Subtopics

### Refined Data Addressing for Bricked Volumes
An addressing scheme designed to reduce the performance penalty of computing two offsets (brick offset + element offset) in a bricked volume layout. It works by testing if needed data elements can be addressed within one brick; if true, it uses a single offset like a linear volume; if false, it performs the costly two-offset calculation [S2, unofficial]. The pitfall is that a naive if-else check for brick boundaries incurs CPU pipeline flushes on architectures with long instruction pipelines due to branch prediction failures [S2, unofficial].

## Operating characteristics
*   **Measured Performance:** Tested on a Pentium 4 Xeon with 512KB Level-2 cache [S2, unofficial].
*   **Strengths:** Efficient handling of large datasets via bricking; user-friendly window/level adjustment; CPU-side optimization for addressing [S1, unofficial][S2, unofficial].
*   **Limitations:** Performance sensitive to CPU pipeline behavior; requires careful tuning of LQ/HQ parameters [S1, unofficial][S2, unofficial].
*   **When to use:** Real-time rendering of large medical datasets on standard PC hardware [S2, unofficial].

## Background
The method originates from research on real-time mono- and multi-volume rendering of large medical datasets on standard PC hardware [S2, unofficial]. It builds on the principles of multiresolution and level of detail, specifically using bricking for spatial decomposition [S2, unofficial]. The implementation details are also found in a WebGPU volume renderer demo, which integrates window/level mapping and brick-based traversal [S1, unofficial].

## Sources
[S1] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S2] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
