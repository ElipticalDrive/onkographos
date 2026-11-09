# Voxel Volume Rendering

Also called: Block model rendering, Shader-driven voxel volume

## Overview
Voxel volume rendering is a technique for visualizing 3D scalar fields by ray marching through a discrete voxel grid stored in a 3D texture. It solves the problem of rendering volumetric data (such as geological assays or simulation results) directly in a graphics pipeline. It is distinguished from implicit surface rendering by its direct manipulation of discrete voxel values and from standard ray marching by its support for "block mode," which snaps sampling to voxel centers for a crisp, block-model aesthetic, alongside standard smooth compositing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field represented as a `Float32Array` of dimensions $(D_x, D_y, D_z)$, where each element holds a value (e.g., concentration).
*   **Input:** Display parameters: `displayMin`, `displayMax`, `opacity`, `threshold`, and a color transfer function.
*   **Output:** A rendered 2D image of the volume.

**Data Structures:**
1.  **Voxel Grid:** A 1D `Float32Array` of size $D_x \times D_y \times D_z$. Each index $i$ corresponds to a voxel. Memory usage is $4 \times D_x \times D_y \times D_z$ bytes (e.g., $64^3 \approx 1$ MB) [S1, unofficial].
2.  **3D Texture:** The grid is uploaded to the GPU as a `Data3DTexture` (one float per voxel) [S1, unofficial].
3.  **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, typically generated per-pixel from a camera.

**Steps:**
1.  **Upload:** Transfer the `Float32Array` to GPU memory as a `Data3DTexture` [S1, unofficial].
2.  **Ray Marching:** For each pixel, cast a ray through the unit-cube bounding box of the volume.
3.  **Sampling:** At each step along the ray, sample the `Data3DTexture` at the current 3D coordinate.
    *   If the value is less than the nodata sentinel ($-1e29$), skip the voxel [S1, unofficial].
    *   If the value is below the `threshold`, treat as transparent [S1, unofficial].
4.  **Color Mapping:** Map the sampled scalar value $v$ to color $C$ and opacity $\alpha$ using a two-stop color transfer function based on `displayMin` and `displayMax` [S1, unofficial].
5.  **Compositing:** Apply front-to-back compositing to accumulate color and opacity along the ray.
6.  **Block Mode (Optional):** If enabled, snap the ray step positions to voxel centers to produce a crisp, discrete block appearance rather than smooth interpolation [S1, unofficial].

**Formulas:**
*   **Front-to-Back Compositing:**
    $$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
    $$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
    Where $C_{in}, \alpha_{in}$ are the accumulated color and opacity, and $C_{sample}, \alpha_{sample}$ are the color and opacity of the current voxel sample [S1, unofficial].

*   **Color Transfer:**
    The color $C$ and opacity $\alpha$ are derived from the scalar value $v$ using a linear or piecewise mapping between `displayMin` and `displayMax`, modulated by global `opacity` and `threshold` [S1, unofficial].

**Parameters:**
*   `dims`: Grid dimensions $(D_x, D_y, D_z)$. Typical: $[64, 64, 64]$ [S1, unofficial].
*   `displayMin`/`displayMax`: Range for color mapping. Typical: $0$ to $500$ [S1, unofficial].
*   `opacity`: Global opacity multiplier. Typical: $0.4$ [S1, unofficial].
*   `threshold`: Alpha cutoff. Voxels below this are skipped. Typical: $0.1$ [S1, unofficial].

**Pitfalls:**
*   **Precision:** Use `Float32Array` for the grid to balance memory and precision [S1, unofficial].
*   **Nodata Handling:** Ensure the sentinel value ($-1e29$) is distinct from valid data to avoid rendering artifacts [S1, unofficial].
*   **Uniform Updates:** Display parameters (min/max, opacity, threshold) are uniform-only updates and do not require re-uploading the texture, allowing real-time tweaking [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shader fragment logic for one ray step
def render_voxel_volume(ray_origin, ray_direction, texture, dims, display_min, display_max, opacity, threshold, block_mode):
    accumulated_color = [0.0, 0.0, 0.0, 0.0]
    t = 0.0
    step_size = 1.0 / max(dims) # Simplified step size
    
    while t < 3.0: # Assume unit cube bounding box
        # Calculate current 3D position
        pos = ray_origin + ray_direction * t
        
        # Block mode: snap to voxel center
        if block_mode:
            voxel_coords = floor(pos + 0.5)
        else:
            voxel_coords = pos
            
        # Sample texture
        value = texture_sample(voxel_coords, texture)
        
        # Skip nodata
        if value < -1e29:
            t += step_size
            continue
            
        # Apply threshold
        if value < threshold:
            t += step_size
            continue
            
        # Map value to color and opacity
        normalized = clamp((value - display_min) / (display_max - display_min), 0.0, 1.0)
        sample_color = transfer_function(normalized)
        sample_alpha = normalized * opacity
        
        # Front-to-back compositing
        accumulated_color = composite(accumulated_color, sample_color, sample_alpha)
        
        # Early exit if fully opaque
        if accumulated_color[3] >= 1.0:
            break
            
        t += step_size
        
    return accumulated_color
```

## Subtopics

### Block Mode Rendering
Snaps raymarching to voxel centers for a crisp, discrete look. Choose when a stylized, blocky aesthetic is desired or when performance is critical and smooth interpolation is unnecessary [S1, unofficial].

### Smooth Compositing
Standard raymarching with trilinear interpolation. Choose when visual fidelity and smooth gradients are required [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(R \cdot S)$, where $R$ is the number of rays (pixels) and $S$ is the number of steps per ray.
*   **Memory Complexity:** $O(D_x \cdot D_y \cdot D_z)$ for the voxel grid. Typical usage: $64^3 \approx 1$ MB, $128^3 \approx 8$ MB, $256^3 \approx 64$ MB [S1, unofficial].
*   **Performance:** Real-time tweaking of display parameters is possible without texture re-upload, as they are uniform updates [S1, unofficial].
*   **Strengths:** Simple implementation, supports real-time parameter adjustment, flexible block vs. smooth modes.
*   **Limitations:** Memory usage grows cubically with resolution; block mode may appear aliased at low resolutions.

## Background
This method is implemented in the Baselode project as part of a 3D IDW interpolation pipeline for visualizing drill-hole sample data. It uses Three.js for rendering and supports both block-style and smooth compositing [S1, unofficial].

## Sources
[S1] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
