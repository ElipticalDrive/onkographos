# Adaptive Ray Marching with Macrocell DDA

Also called: Adaptive Macrocell Ray Marching, Macrocell solution

## Overview
Adaptive Ray Marching with Macrocell DDA is a ray marching technique that traverses a volumetric grid using a 3D Digital Differential Analyzer (DDA) over macrocells, adapting the sampling step size based on opacity upper bounds to skip empty or low-opacity regions [S1][S2, unofficial]. It differs from its parent [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md) by dynamically adjusting the step size within occupied macrocells rather than only skipping entirely empty ones, and from its sibling [Triangulated Active Brick Boundary Mesh for Space Skipping](1426-triangulated-active-brick-boundary-mesh-for-space-skipping.md) by relying on a uniform grid structure with DDA traversal rather than explicit boundary meshes [S1].

## How it works
The method combines coarse-grained space skipping with fine-grained adaptive sampling [S1][S2, unofficial]. It uses a 3D DDA to traverse a grid of macrocells [S2, unofficial]. For each macrocell, it estimates an opacity upper bound [S2, unofficial]. If the bound is negligible, the cell is skipped [S2, unofficial]. Otherwise, the step size for sampling within the cell is scaled based on the opacity, allowing larger steps in transparent regions and smaller steps in dense regions [S2, unofficial].

### Inputs and Outputs
*   **Input**: Ray origin $O$, ray direction $D$ (normalized), ray interval $[t_{min}, t_{max}]$, volume data structure containing macrocell metadata [S1][S2, unofficial].
*   **Output**: Accumulated color and opacity, or a boolean indicating if the ray hit an opaque surface [S2, unofficial].

### Data Structures
The volume is represented by a `DeviceVolume` structure containing:
*   `macrocell_dims`: Dimensions of the macrocell grid (e.g., $N_x, N_y, N_z$) [S2, unofficial].
*   `macrocell_spacings_rcp`: Reciprocal of the macrocell spacing in world units. This allows conversion from world coordinates to macrocell indices via multiplication [S2, unofficial].
*   `opacity_upper_bounds`: A data structure (e.g., array or hash map) storing the maximum opacity estimate for each macrocell, derived from minimum and maximum scalar values of enclosed voxels [S1][S2, unofficial].

### Parameters
*   `ADAPTIVE_SAMPLING`: A preprocessor macro or boolean flag. If enabled, adaptive logic is used; otherwise, fixed-step marching is performed [S2, unofficial].
*   `float_epsilon`: A small constant used to determine if a macrocell is effectively empty. If the opacity upper bound $r \le \text{float\_epsilon}$, the cell is skipped [S2, unofficial].
*   `step`: The base sampling step size. This is the default step size before adaptive scaling is applied [S2, unofficial].
*   Brick size is $dx^n \times dy^n \times dz^n$, where $dx, dy, dz$ are the volume dimensions. The parameter $n$ was tested from 0.1 to 0.9, with best performance at $n=0.3$ [S1].

### Algorithm Steps

1.  **Transform Ray to Macrocell Space**:
    Calculate the ray origin and direction in macrocell index space:
    $$ m_{org} = O \times \text{macrocell\_spacings\_rcp} $$
    $$ m_{dir} = D \times \text{macrocell\_spacings\_rcp} $$

2.  **Initialize DDA**:
    Initialize a 3D DDA iterator with $m_{org}$, $m_{dir}$, and the ray interval $[t_{min}, t_{max}]$ [S2, unofficial]. The DDA yields a sequence of macrocells $(cell, t_0, t_1)$, where $t_0$ and $t_1$ are the entry and exit distances of the ray from the current macrocell in world units [S2, unofficial].

3.  **Iterate Macrocells**:
    For each macrocell $(cell, t_0, t_1)$ returned by the DDA:
    
    a. **Check Opacity Bound**:
    Retrieve the opacity upper bound $r$ for the current cell [S2, unofficial].
    
    b. **Skip Empty Cells**:
    If $|r| \le \text{float\_epsilon}$, the cell is considered empty. Skip to the next macrocell [S2, unofficial].
    
    c. **Calculate Adaptive Step Size**:
    Determine the scaling factor based on the opacity bound. Let $ss$ be the adaptive step size:
    $$ ss = \text{sample\_size\_scaler}(\text{adaptiveSamplingRate}(step, r), t_0, t_1) $$
    *Note: `adaptiveSamplingRate` returns a factor that increases as $r$ decreases. `sample_size_scaler` ensures the step fits within the remaining interval $[t_0, t_1]$ [S2, unofficial].*
    
    d. **Sample Within Macrocell**:
    Initialize the current sampling interval $t$ within the macrocell:
    $$ t_{start} = t_0 $$
    $$ t_{end} = \min(t_1, t_0 + ss) $$
    
    While $t_{end} > t_{start}$:
    i. Evaluate the volume rendering body function (sample density, apply transfer function, accumulate color/opacity) at the current position defined by $t_{start}$ [S2, unofficial].
    ii. If the body function indicates the ray is fully opaque or terminated, return immediately [S2, unofficial].
    iii. Advance the interval:
    $$ t_{start} = t_{end} $$
    $$ t_{end} = \min(t_1, t_{start} + ss) $$

4.  **Termination**:
    If the ray exits the volume or is fully absorbed, return the accumulated result [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
FLOAT_EPSILON = 1e-6

def adaptive_ray_march(ray_origin, ray_direction, t_min, t_max, volume):
    # 1. Transform to macrocell space
    m_org = ray_origin * volume.macrocell_spacings_rcp
    m_dir = ray_direction * volume.macrocell_spacings_rcp
    
    # 2. Initialize 3D DDA iterator
    # Yields (cell_index, t_entry, t_exit) for each macrocell intersected
    dda = DDA3(m_org, m_dir, t_min, t_max, volume.macrocell_dims)
    
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    
    # 3. Iterate macrocells
    for cell, t0, t1 in dda:
        # Get opacity upper bound for this macrocell
        r = volume.get_opacity_upper_bound(cell)
        
        # Skip empty cells
        if abs(r) <= FLOAT_EPSILON:
            continue
            
        # Calculate adaptive step size
        step_factor = adaptive_sampling_rate(volume.base_step, r)
        ss = min(step_factor, t1 - t0)
        
        # 4. Sample within the macrocell
        t_current = t0
        t_next = min(t1, t0 + ss)
        
        while t_next > t_current:
            pos = ray_origin + ray_direction * t_current
            
            # Evaluate volume (density, color, etc.)
            # Updates accumulated_color and accumulated_opacity
            # Returns True if ray is terminated (fully opaque)
            if volume_render_body(pos, accumulated_color, accumulated_opacity):
                return vec4(accumulated_color, accumulated_opacity)
            
            # Advance interval
            t_current = t_next
            t_next = min(t1, t_current + ss)
            
    return vec4(accumulated_color, accumulated_opacity)

def adaptive_sampling_rate(base_step, opacity_bound):
    """
    Returns a scaling factor for the step size.
    Higher opacity bound -> smaller step (more samples).
    Lower opacity bound -> larger step (fewer samples).
    """
    if opacity_bound < FLOAT_EPSILON:
        return base_step
    # Scale factor inversely proportional to opacity bound
    return base_step / max(opacity_bound, FLOAT_EPSILON)

def volume_render_body(pos, accumulated_color, accumulated_opacity):
    """
    Standard volume rendering step: sample density, apply transfer function, accumulate.
    Returns True if ray is fully opaque.
    # Implementation depends on specific transfer function and density field.
    """
    pass 

class DDA3:
    """
    Standard 3D Digital Differential Analyzer iterator.
    Traverses a uniform grid along a ray, yielding cell indices and entry/exit distances.
    # Standard 3D DDA implementation not detailed in sources [G].
    """
    pass
```

### Edge Cases and Pitfalls
*   **Precision**: When $t_0$ and $t_1$ are very close, floating-point errors may cause $t_{next} \le t_{current}$, leading to infinite loops. A minimum step size guard or epsilon check is required [G].
*   **Boundary Conditions**: The DDA must correctly handle rays that start or end inside a macrocell. The initial $t_0$ and final $t_1$ must be clamped to the ray interval $[t_{min}, t_{max}]$ [S2, unofficial].
*   **Dynamic Volume Overhead**: For dynamic volumes, the macrocell structure must be updated every frame, which can be costly and become a performance bottleneck, especially when the volume image ratio is large [S1].

## Subtopics

### [Resumable Ray Marching Iterator](1505-resumable-ray-marching-iterator.md)
Extends the parent by supporting interruption and resumption of ray traversal state, allowing pausing mid-ray for external operations.

### [Network Ray Marching Iterator](1510-network-ray-marching-iterator.md)
Extends the parent for distributed rendering by introducing block-level divergence control via `block_any` to manage thread synchronization across macrocell boundaries.

### [Macrocell Grid Traversal with Majorant Densities](1528-macrocell-grid-traversal-with-majorant-densities.md)
Combines macrocell traversal with Delta Tracking (Woodcock tracking) using alpha as a majorant extinction coefficient for probabilistic free-flight distances.

### [Voxel Ray Marching with Cube-Level DDA](3069-voxel-ray-marching-with-cube-level-dda.md)
Uses a strict two-level DDA (macrocell cube then voxel) implemented in fragment shaders, differing from the parent's adaptive step size by using fixed-size macrocells.

### Space Skipping with 3DDDA
A mechanism to skip empty regions in sparse volume data using a uniform grid and 3DDDA algorithm [S3]. The fragment shader traverses the macrocell grid using 3DDDA in an outer loop [S3]. If a macrocell is nonempty, it enters the volume rendering loop with peak finding tests between samples [S3]. To begin sampling, it finds the first $t$ to sample when entering a macrocell by solving for the maximum $x$ after $T_{enter}$: $x_{tenter} = (-b + \sqrt{b^2 + 2aT_{enter}})/a$ [S3]. It then computes floor values and samples as in Equation 17 of [S3]. Coarse grids (64^3 cells) work best on GPU [S3].

### Adaptive Ray Marching with Occupancy Grid
A variant that uses a bitfield occupancy grid to skip empty space [S4, unofficial]. For each ray, it iterates through spatial steps [S4, unofficial]. It checks occupancy using a bitfield: `sity_bitfield[ti.u32(idx//8)] & (1 << ti.u32(idx%8))` [S4, unofficial]. If occupied, it stores position, direction, time, and delta into buffers and marks for network evaluation [S4, unofficial]. If not occupied, it calculates the distance to the next grid boundary using `txyz = (((nxyz+0.5+0.5*ti.math.sign(ray_d))*grid_size_inv*2-1)*mip_bound-xyz)*d_inv` and advances time to skip empty space until the next boundary is reached [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_{macrocells} + N_{samples})$, where $N_{macrocells}$ is the number of macrocells traversed and $N_{samples}$ is the total number of samples taken. Adaptive sampling reduces $N_{samples}$ in transparent regions [S2, unofficial].
*   **Memory Complexity**: $O(N_{macrocells})$ for storing opacity upper bounds [S2, unofficial].
*   **Performance**: Significantly faster than fixed-step marching in sparse volumes. Performance depends on the accuracy of the opacity upper bounds [S1]. Empty space skipping structures are ideal for static volumes or for dynamic volumes when the rendered image resolution is large enough to hide the structure reconstruction overheads [S1]. Warp Marching outperforms Macrocell methods when the volume image ratio is large (resolutions 256×256 or 512×512), because updating the macrocell structure every frame is a bottleneck; at 1024×1024, Macrocell performs better [S1].
*   **Strengths**: Efficient space skipping, adaptive resolution, simple implementation on GPU [S1][S2, unofficial].
*   **Limitations**: Requires precomputed opacity bounds. Construction and update overhead can dominate rendering time for high-resolution dynamic volumes [S1].

## Background
The method is based on the `instantvnr` project, which implements interactive volume visualization via multi-resolution hash encoding based neural representation [S2, unofficial]. The core idea of using macrocells with DDA for space skipping originates from early implementations using uniform grids and 3DDDA for peak finding and differential sampling [S3]. The adaptive sampling aspect was introduced to improve performance on GPUs where purely adaptive methods based on local gradients suffer from poor thread coherence [S2, unofficial].

## Sources
[S1] A cache-friendly sampling strategy for texture-based volume rendering on GPU, https://doi.org/10.1016/j.visinf.2017.08.001, fetched 2026-10-26
[S2] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S3] Volume Ray Casting with Peak Finding and Differential ..., https://www.sci.utah.edu/~knolla/avrc.pdf, fetched 2026-10-27
[S4] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
