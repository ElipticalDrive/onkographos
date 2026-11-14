# Local Maximum Intensity Projection

Also called: LMIP Compositing, Local Maximum Importance Projection

## Overview
Local Maximum Intensity Projection (LMIP) is a volume rendering compositing technique that projects the first local maximum intensity value along a viewing ray that exceeds a user-defined threshold, rather than the global maximum as in standard Maximum Intensity Projection (MIP) [S1, S3, S6, unofficial]. It solves the depth ambiguity and occlusion problems inherent in MIP, where high-intensity structures in the foreground can obscure lower-intensity structures behind them, by preserving spatial relationships through early ray termination [S2, S8, unofficial]. Unlike MIP, which ignores intermediate values, LMIP provides depth cues and surface-like appearance without requiring the complex transfer function definitions of Direct Volume Ray Casting (DVR) [S1, S8, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D volume dataset $V$, a viewing ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, a user-defined intensity threshold $T$, and the volume bounds.
- **Output:** A single color value for each pixel, corresponding to the intensity of the first local maximum above $T$ encountered along the ray.

**Data Structures:**
- **Volume Data:** A 3D grid of scalar values (e.g., density or attenuation).
- **Threshold Mask (Optional Optimization):** A binary representation where each voxel is 1 if $V(x,y,z) \ge T$ and 0 otherwise. This allows bit-parallel processing [S3, unofficial].
- **Lookup Table (LUT):** A precomputed table mapping byte patterns to the position of the first non-zero bit, used to accelerate the search for the first sample above the threshold [S3, unofficial].

**Algorithm Steps:**

1. **Ray Setup:**
   - Compute the intersection of the viewing ray with the volume bounding box (typically normalized to a unit cube $[0,1]^3$) [S4, unofficial].
   - Calculate entry distance $t_{entry}$ and exit distance $t_{exit}$ [S4, unofficial].
   - If $t_{entry} > t_{exit}$, discard the ray (no intersection) [S4, unofficial].
   - Clamp $t_{entry}$ to be at least 0.0 [S4, unofficial].
   - Normalize the ray direction vector $\mathbf{d}$ [S4, S5, unofficial].
   - Compute the step size $\Delta t$ based on the desired sampling resolution [S4, unofficial].

2. **Initial Search (Threshold Crossing):**
   - Initialize current position $p = \mathbf{o} + t_{entry}\mathbf{d}$ [S4, unofficial].
   - March along the ray in front-to-back order [S5, S8, unofficial].
   - Sample the volume at $p$. If using the bit-parallel optimization, pack voxel values into integers and use the LUT to skip to the first voxel where $V(p) \ge T$ [S3, unofficial].
   - If no voxel above $T$ is found within the volume bounds, fall back to standard MIP: find the global maximum along the entire ray path using a min/max grid or full traversal [S3, S8, unofficial].

3. **Local Maximum Detection:**
   - Once a sample $V_i$ at position $p_i$ satisfies $V_i \ge T$, begin searching for the first local maximum [S3, S5, unofficial].
   - A local maximum is defined as a sample $V_i$ such that $V_i \ge V_{i-1}$ and $V_i \ge V_{i+1}$ (considering boundary conditions) [S3, S5, unofficial].
   - Continue marching forward until a local maximum is identified [S3, unofficial].
   - Alternatively, some implementations define LMIP as simply the *first* sample above the threshold, assuming it corresponds to the surface entry, effectively treating the threshold crossing as the local maximum reference [S8, unofficial].

4. **Refinement (Optional):**
   - To improve precision, perform a refinement pass around the identified local maximum position [S4, unofficial].
   - Set the refinement step size $\Delta t_{refine} = \Delta t / N_{refine}$, where $N_{refine}$ is a constant (e.g., 4) [S4, unofficial].
   - Center the search around the found maximum position [S4, unofficial].
   - Iterate $N_{refine}$ times, sampling the volume and updating the maximum value found [S4, unofficial].

5. **Compositing and Output:**
   - Assign the final intensity value $V_{max}$ to the pixel [S8, unofficial].
   - Discard all subsequent samples along the ray (early ray termination) [S8, unofficial].
   - Map $V_{max}$ to a color using a color map $C(V_{max})$ [S4, unofficial].
   - Output the color as the fragment color [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_lmip(ray_origin, ray_direction, volume, threshold, refinement_steps=4):
    """
    Renders a single pixel using Local Maximum Intensity Projection.
    
    Args:
        ray_origin: vec3, origin of the view ray.
        ray_direction: vec3, direction of the view ray (should be normalized).
        volume: 3D array, the volume dataset.
        threshold: float, the intensity threshold T.
        refinement_steps: int, number of refinement samples around the max.
        
    Returns:
        vec3, the final color for the pixel.
    """
    # 1. Ray-Box Intersection
    # Assuming volume is in a unit cube [0,1]^3 for simplicity
    t_min = 0.0
    t_max = 1.0
    
    # Calculate intersection with unit cube
    # This is a simplified intersection logic; real implementations use slab method
    t_entry = 0.0
    t_exit = 1.0
    
    # Example: intersect with unit cube bounds
    # In practice, use the standard ray-aabb intersection algorithm
    # For this pseudocode, assume t_entry and t_exit are computed correctly
    # and ray_direction is normalized.
    
    if t_entry > t_exit:
        return vec3(0, 0, 0) # No intersection
    
    # Clamp entry to start of volume
    t_entry = max(t_entry, 0.0)
    
    # 2. Marching
    step_size = 1.0 / volume.resolution # Assuming uniform grid resolution
    current_pos = ray_origin + t_entry * ray_direction
    
    max_val = -infinity
    found_above_threshold = False
    local_max_val = -infinity
    
    # Check if any voxel is above threshold
    # Optimization: Use bit-parallel search or min/max grid to skip
    # Here we do a simple linear scan for clarity
    
    samples = []
    pos = current_pos
    while pos_inside_volume(pos):
        val = sample_volume(pos, volume)
        samples.append((val, pos))
        if val >= threshold:
            found_above_threshold = True
        pos += ray_direction * step_size
        
    if not found_above_threshold:
        # Fallback to Global MIP
        for val, pos in samples:
            if val > max_val:
                max_val = val
        return apply_color(max_val)
    
    # 3. Find First Local Maximum Above Threshold
    # Iterate through samples to find the first local max >= threshold
    for i in range(1, len(samples) - 1):
        val_prev, pos_prev = samples[i-1]
        val_curr, pos_curr = samples[i]
        val_next, pos_next = samples[i+1]
        
        if val_curr >= threshold:
            # Check if local maximum
            if val_curr >= val_prev and val_curr >= val_next:
                local_max_val = val_curr
                local_max_pos = pos_curr
                break
    
    # If no strict local max found, take the first sample above threshold (surface entry)
    if local_max_val == -infinity:
        for val, pos in samples:
            if val >= threshold:
                local_max_val = val
                local_max_pos = pos
                break
    
    # 4. Refinement
    # Refine the position and value around local_max_pos
    refine_step = step_size / refinement_steps
    refined_max = local_max_val
    
    # Center refinement around the found position
    refine_start = local_max_pos - ray_direction * (refine_step * refinement_steps / 2.0)
    
    for i in range(refinement_steps):
        refine_pos = refine_start + ray_direction * (i * refine_step)
        refine_val = sample_volume(refine_pos, volume)
        if refine_val > refined_max:
            refined_max = refine_val
            
    return apply_color(refined_max)

def sample_volume(pos, volume):
    # Trilinear interpolation
    # ... implementation details ...
    pass

def pos_inside_volume(pos):
    # Check if pos is within [0,1]^3
    return all(0 <= c <= 1 for c in pos)

def apply_color(val):
    # Map intensity to color
    # ... implementation details ...
    pass
```

**Parameters:**
- **Threshold ($T$):** A user-defined intensity value. Typical values depend on the data range (e.g., 0.5 for normalized data) [S5, S8, unofficial].
- **Refinement Steps ($N_{refine}$):** Number of samples in the refinement pass. A common value is 4 [S4, unofficial].

**Edge Cases and Pitfalls:**
- **No Threshold Crossing:** If no voxel along the ray exceeds the threshold, the method must fall back to global MIP to avoid black pixels [S3, S5, unofficial].
- **Occlusion:** LMIP does not support translucent materials; it discards all samples after the first local maximum, which can lead to occlusion artifacts if the local maximum is not the true surface [S5, S8, unofficial].
- **Ray Direction Normalization:** The ray direction must be normalized after interpolation to ensure correct step sizes [S4, unofficial].
- **Boundary Conditions:** Care must be taken when sampling at the volume boundaries to avoid out-of-bounds access [S4, unofficial].

## Subtopics

### [Depth-Enhanced Maximum Intensity Projection](14725-depth-enhanced-maximum-intensity-projection.md)
Depth-Enhanced Maximum Intensity Projection (DEMIP) is a variant that improves depth perception in MIP by modifying intensity accumulation via depth weighting and applying a sphere-based color mapping to disambiguate the ordering of internal structures. Unlike standard MIP, which loses spatial context, DEMIP provides depth cues and occlusion visibility without requiring the complex transfer function definition of Direct Volume Ray Casting (DVR). It differs from isosurface-based depth methods by directly rendering the model using classical GPU raycasting.

### Cylindrical MImP
Cylindrical MImP is a Maximum Importance Projection variant that does not clearly show spatial relationships between focus and context objects [S9, unofficial]. The focus object appears in front of the context object, creating a visual artifact [S9, unofficial].

### Maximum Intensity Projection with Multidimensional Transfer Function
MIP does not benefit from multidimensional transfer functions for tissue differentiation because it ignores non-maximum values where tissue boundaries might be defined [S10, unofficial]. When using MIP, multidimensional transfer function results are nearly identical to scalar-only transfer function results because MIP only considers maximum values, ignoring tissue/flat surface issues [S10, unofficial]. Using only gradient magnitude for opacity in MIP provides a much more different and interesting rendering compared to multidimensional or scalar-only approaches [S10, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** LMIP is generally faster than traditional MIP due to early ray termination once the first local maximum above the threshold is found [S3, unofficial]. However, the initial search for the threshold crossing and local maximum can add overhead compared to simple MIP [S3, unofficial].
- **Memory:** Requires storage for the volume data and optionally a binary threshold mask or LUT for acceleration [S3, unofficial].

**Measured Performance:**
- Performance depends on the threshold setting and the density of high-intensity structures. Higher thresholds lead to earlier termination and faster rendering [S3, unofficial].
- The bit-parallel optimization with LUTs can significantly speed up the search for the first sample above the threshold [S3, unofficial].

**Strengths and Limitations:**
- **Strengths:** Preserves spatial relationships and depth cues better than MIP [S2, S8, unofficial]. Provides surface-like appearance without isosurface extraction [S2, unofficial]. Interactive control over threshold allows flexibility [S5, unofficial].
- **Limitations:** Suffers from occlusion problems because it does not support translucent materials [S5, unofficial]. Cannot apply precise density encoding like DVR [S3, unofficial]. Choice of threshold is critical and may require user interaction [S5, unofficial].

**When to Use:**
- Use LMIP when depth perception and spatial relationships are important, such as in medical angiography visualization [S2, unofficial].
- Use when a surface-like appearance is desired without the computational cost of isosurface extraction [S2, unofficial].

**When Not to Use:**
- Do not use when translucent materials or internal structures need to be visualized simultaneously [S5, unofficial].
- Do not use when precise density encoding is required [S3, unofficial].

## Background
Local Maximum Intensity Projection was proposed by Sato et al. to overcome the limitations of MIP, specifically the loss of depth information and occlusion issues [S2, S6, unofficial]. It was introduced in the context of medical imaging, particularly for CT and MRI angiographies, where visualizing vascular structures with clear spatial relationships is crucial [S2, unofficial]. The technique has been implemented in various volume rendering systems and is often compared to isosurface rendering and DVR for its balance between performance and visual quality [S1, S8, unofficial].

## Sources
[S1] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S3] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S4] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S5] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S6] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S7] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S8] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S9] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
[S10] Volume Rendering, Maximum Intensity Projection and ..., https://ryanrossi.com/sv3.php, fetched 2026-10-28 (unofficial)
