# Window/Level Volume Raycasting with Bricking

Also called: LQ/HQ Volume Rendering, Volume Raycasting with Window/Level and Bricking, Window/Level Volume Rendering

## Overview
Window/Level Volume Raycasting with Bricking is a direct volume rendering technique that combines spatial data decomposition (bricking) with intensity-based windowing to accelerate the rendering of large medical datasets on standard hardware [S1, S2, unofficial]. It solves the memory bandwidth and cache inefficiency problems associated with linear volume layouts by storing data in fixed-size bricks, while using a window/level transfer function to map scalar values to optical properties [S1, S2, unofficial]. It differs from standard raycasting by explicitly managing brick boundaries and utilizing density thresholds for step skipping, and from pure bricking methods by integrating specific window/level parameters and low/high-quality (LQ/HQ) rendering modes [S1, unofficial].

## How it works
The method implements the emission-absorption optical model as defined in [Direct volume ray casting](19-direct-volume-ray-casting.md), but optimizes data access and ray marching through bricking and window/level mapping [S1, S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** Stored in a bricked layout. The volume is decomposed into chunks (bricks) of size `chunkSize` along each dimension [S1, unofficial].
*   **Window/Level Parameters:** `gLevel` (window center) and `gWindow` (window width) define the linear mapping from voxel intensity to opacity/color [S1, unofficial].
*   **Geometry:** Camera origin $\mathbf{o}$, direction $\mathbf{d}$, and volume bounds $[xmin, xmax, ymin, ymax, zmin, zmax]$ [S1, unofficial].
*   **Voxel Spacing:** Buffer containing spacing in x, y, z dimensions [S1, unofficial].
*   **Rendering Mode:** Boolean flag `usingHQ` selects between Low Quality (LQ) and High Quality (HQ) parameter sets [S1, unofficial].

**Outputs:**
*   **Pixel Color:** Final accumulated color $C_{final}$ for each pixel [S1, unofficial].

### Data Structures
1.  **Brick Layout:** The volume is divided into $N_x \times N_y \times N_z$ bricks. Each brick contains a contiguous block of voxel data. The total number of chunks is `numChunksX`, `numChunksY`, `numChunksZ` [S1, unofficial].
2.  **Parameter Sets (LQ/HQ):** Two sets of thresholds and multipliers are maintained to toggle between performance and quality [S1, unofficial]:
    *   `gDensityForMarchSpaceSkippingLQ/HQ`: Threshold density value used to skip marching steps.
    *   `gSkipMultiplierLQ/HQ`: Multiplier for skipping steps based on density.
    *   `gSubtleSurfaceThresholdLQ/HQ`: Threshold for subtle surface detection.
    *   `gSurfaceThresholdLQ/HQ`: Threshold for surface rendering.
    *   `gMaxStepsLQ/HQ`: Maximum number of ray marching steps allowed.
    *   `gMinGradientMagnitudeLQ/HQ`: Minimum gradient magnitude required for gradient calculation.
    *   `gAccumulatedThresholdLQ/HQ`: Threshold for accumulated opacity to terminate early.
    *   `gTransmittanceThresholdLQ/HQ`: Threshold for transmittance to terminate early.
    *   `gUseGradientLQ/HQ`: Boolean flag enabling gradient calculation for LQ/HQ modes [S1, unofficial].

### Algorithm Steps

1.  **Ray Generation:** For each pixel, compute the ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S1, S2, unofficial]. Intersect with the volume bounding box to find $t_{start}$ and $t_{end}$ [S1, unofficial].
2.  **Brick Traversal:** The ray is marched through the volume. Instead of linear indexing, the algorithm determines which brick contains the current sample point [S2, unofficial].
3.  **Data Addressing (Refined):** To address a sample within a brick, the algorithm computes the brick offset and the element offset within that brick [S2, unofficial].
    *   *Pitfall:* Using standard `if-else` branches to check brick boundaries causes pipeline flushes on CPUs with long instruction pipelines due to branch prediction failures [S2, unofficial]. The implementation must minimize conditional branches during addressing [S2, unofficial].
4.  **Sampling and Classification:**
    *   Sample the scalar value $V(\mathbf{p}_i)$ at the current position [S1, S2, unofficial].
    *   Apply Window/Level mapping to determine opacity $\alpha_i$ and color $C_i$. The standard window/level transformation is:
        $$
        \alpha_i = \text{clamp}\left( \frac{V(\mathbf{p}_i) - (gLevel - gWindow/2)}{gWindow}, 0, 1 \right)
        $$
        (Note: Exact mapping formula may vary by implementation, but the parameters `gLevel` and `gWindow` are the defining inputs [S1, unofficial]).
5.  **Shading (Optional):** If `toggleGradient` is true and `usingHQ`/`usingLQ` permits, compute the gradient $\nabla V$ using a 26-connected neighborhood access pattern to estimate surface normals for Phong shading [S1, S2, unofficial].
6.  **Step Skipping (Acceleration):**
    *   If the current voxel value $V(\mathbf{p}_i)$ is below `gDensityForMarchSpaceSkipping`, skip forward by `skipMultiplier` steps [S1, unofficial].
    *   This reduces the number of samples taken in empty or low-density regions [S1, unofficial].
7.  **Compositing:**
    *   Update accumulated color $C'$ and opacity $\alpha'$ using the front-to-back over operator [S1, S2, unofficial]:
        $$
        C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
        $$
        $$
        \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
        $$
    *   **Early Termination:** If $\alpha' > gAccumulatedThreshold$ or transmittance $(1-\alpha') < gTransmittanceThreshold$, terminate the ray [S1, unofficial].
8.  **Output:** Write $C'$ to the frame buffer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_window_level_bricking(volume_bricks, camera, params):
    """
    Renders a volume using window/level raycasting with bricking.
    
    Args:
        volume_bricks: Array of bricks. Each brick is a 3D array of voxel values.
        camera: Object with origin 'o' and direction 'd' for each pixel.
        params: Object containing gLevel, gWindow, chunkSize, and LQ/HQ thresholds.
        
    Returns:
        Image buffer with final colors.
    """
    # Extract parameters based on HQ/LQ mode
    is_hq = params.usingHQ
    density_skip_threshold = params.gDensityForMarchSpaceSkippingHQ if is_hq else params.gDensityForMarchSpaceSkippingLQ
    skip_multiplier = params.gSkipMultiplierHQ if is_hq else params.gSkipMultiplierLQ
    max_steps = params.gMaxStepsHQ if is_hq else params.gMaxStepsLQ
    accum_threshold = params.gAccumulatedThresholdHQ if is_hq else params.gAccumulatedThresholdLQ
    transmittance_threshold = params.gTransmittanceThresholdHQ if is_hq else params.gTransmittanceThresholdLQ
    use_gradient = params.gUseGradientHQ if is_hq else params.gUseGradientLQ
    window_center = params.gLevel
    window_width = params.gWindow
    
    image = empty_buffer(camera.width, camera.height)
    
    for y in range(camera.height):
        for x in range(camera.width):
            ray_o = camera.origin(x, y)
            ray_d = camera.direction(x, y)
            
            # Intersect ray with volume bounds
            t_start, t_end = intersect_bbox(ray_o, ray_d, params.bounds)
            if t_start > t_end:
                image[x, y] = (0, 0, 0)
                continue
                
            # Discretize ray
            step_size = compute_step_size(ray_d, params.voxel_spacing)
            num_samples = int((t_end - t_start) / step_size)
            num_samples = min(num_samples, max_steps)
            
            accumulated_color = (0.0, 0.0, 0.0)
            accumulated_opacity = 0.0
            
            for i in range(num_samples):
                t = t_start + i * step_size
                p = ray_o + t * ray_d
                
                # Determine brick index and local coordinates
                # This step must avoid expensive if-else branches for performance [S2, unofficial]
                brick_idx, local_coords = get_brick_and_coords(p, params.chunk_size, params.num_chunks)
                
                # Sample voxel value
                voxel_val = sample_voxel(volume_bricks[brick_idx], local_coords)
                
                # Step skipping optimization [S1, unofficial]
                if voxel_val < density_skip_threshold:
                    continue # Skip this sample, effectively jumping in space
                
                # Window/Level Classification
                # Map voxel value to opacity [S1, unofficial]
                opacity = clamp((voxel_val - (window_center - window_width / 2.0)) / window_width, 0.0, 1.0)
                
                # Color is typically derived from the same mapping or a lookup table
                # Assuming a simple grayscale or pre-defined color map for this implementation
                color = (voxel_val, voxel_val, voxel_val) 
                
                # Shading (if enabled) [S1, unofficial]
                if use_gradient:
                    # Compute gradient using 26-connected neighborhood [S2, unofficial]
                    # This requires accessing neighboring bricks if at boundary
                    gradient = compute_gradient(volume_bricks, brick_idx, local_coords, params)
                    # Phong shading calculation would go here
                    # color = apply_phong_shading(color, gradient, light_dir)
                
                # Front-to-Back Compositing [S1, unofficial]
                transmittance = 1.0 - accumulated_opacity
                contribution = transmittance * opacity * color
                accumulated_color += contribution
                accumulated_opacity += transmittance * opacity
                
                # Early termination [S1, unofficial]
                if accumulated_opacity > accum_threshold:
                    break
                if (1.0 - accumulated_opacity) < transmittance_threshold:
                    break
            
            image[x, y] = accumulated_color
            
    return image
```

### Edge Cases and Pitfalls
*   **Branch Prediction:** Using `if-else` to check if a sample falls within the current brick causes pipeline flushes on CPUs with long pipelines, significantly degrading performance [S2, unofficial]. The addressing logic must be optimized to minimize these branches [S2, unofficial].
*   **Boundary Conditions:** When computing gradients or sampling neighborhoods, the algorithm must handle cases where the neighborhood extends into adjacent bricks or outside the volume [S2, unofficial].
*   **Precision:** Floating-point precision in ray position calculation can lead to sampling errors in large volumes. Using voxel spacing buffer ensures correct physical distance sampling [S1, unofficial].

## Subtopics

### [Octree-based Void Culling](3828-octree-based-void-culling.md)
Octree-based void culling is a spatial acceleration technique used in direct volume ray casting to reduce rasterization load by skipping empty regions within volume bricks. It builds on [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-raycasting-with-bricking.md) by introducing a per-brick octree structure that describes only the geometry of visible data, allowing the renderer to bypass void areas before invoking the GPU. Unlike sibling methods that may rely on global spatial structures or GPU-side culling, this approach generates and traverses the octree on the CPU to minimize GPU workload, while the GPU handles the actual interpolation, transfer function application, and blending.

### [Raycasting with Bricking and Phong Shading](6813-raycasting-with-bricking-and-phong-shading.md)
A CPU-based direct volume ray casting algorithm that accelerates traversal using a brick-based data layout and computes illumination via Phong shading with gradient estimation. It differs from its parent [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-raycasting-with-bricking.md) by explicitly integrating Phong shading and gradient calculation into the ray loop, and from its sibling [Octree-based Void Culling](3828-octree-based-void-culling.md) by relying on fixed-size bricks and distance volume tables rather than hierarchical void culling for acceleration.

### Refined Data Addressing for Bricked Volumes
An addressing scheme designed to minimize the performance penalty of computing two offsets (brick offset + element offset) in a bricked volume layout, specifically avoiding pipeline flushes caused by conditional branches in if-else statements [S2, unofficial]. The method tests if the needed data elements can be addressed within one brick; if true, it addresses data elements as fast as in a linear volume (single offset); if false, it performs costly address calculations [S2, unofficial]. Using an if-else statement to check brick boundaries incurs pipeline flushes on CPUs with long instruction pipelines that use branch prediction, as the CPU may flush the pipeline if the branch outcome is false [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the average number of samples per ray. Step skipping reduces $M$ in empty regions [S1, unofficial].
*   **Memory Complexity:** $O(V)$ where $V$ is the volume size, stored in bricks. The brick size is a trade-off between cache locality and memory overhead [S2, unofficial].
*   **Performance:** Optimized for standard PC hardware with large L2 caches (e.g., Pentium 4 Xeon 512KB L2) by minimizing cache misses and pipeline flushes [S2, unofficial].
*   **Strengths:** Efficient handling of large medical datasets; adaptive quality via LQ/HQ modes; acceleration via step skipping and bricking [S1, unofficial].
*   **Limitations:** Performance sensitive to branch prediction efficiency in data addressing; gradient computation is expensive [S1, S2, unofficial].
*   **When to use:** Real-time visualization of large medical volumes on CPU-based systems where GPU acceleration is limited or unavailable [S1, S2, unofficial].
*   **When not to use:** When GPU-based acceleration is available, or when hierarchical void culling is more appropriate for highly sparse volumes [S1, unofficial].

## Background
The method originates from research on real-time mono- and multi-volume rendering of large medical datasets on standard PC hardware [S2, unofficial]. It was developed to address the limitations of linear volume layouts in terms of cache efficiency and memory bandwidth [S2, unofficial]. The integration of window/level parameters allows for interactive adjustment of contrast and brightness, which is critical in medical imaging [S1, unofficial]. The LQ/HQ modes provide flexibility for different hardware capabilities [S1, unofficial].

## Sources
[S1] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S2] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
