# Voxel Volume Rendering

Also called: Block model rendering, Shader-driven voxel volume

## Overview
Voxel volume rendering visualizes a 3D scalar field by projecting light absorption and emission through a discrete voxel grid, revealing internal density variations without surface extraction [S1, S2, unofficial]. It differs from isosurface methods (like [Marching Cubes](link)) by rendering the entire volume data rather than a single thresholded boundary, and from analytic methods by operating on pre-computed discrete samples [S1, S2, unofficial]. The method supports two distinct visual styles: "block mode," which snaps raymarching to voxel centers for a crisp, discrete appearance, and "smooth compositing," which interpolates values for a continuous look [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field represented as a voxel grid (e.g., `Float32Array` of size $N_x \times N_y \times N_z$) [S2, unofficial].
*   **Input:** Camera parameters (position, focal point, view-up vector) [S1, unofficial].
*   **Input:** Transfer function parameters (display min/max, opacity, threshold, color gradient) [S2, unofficial].
*   **Output:** A 2D rendered image (e.g., PNG) [S1, unofficial].

**Data Structures:**
1.  **Voxel Grid:** A 1D array of 32-bit floats representing the scalar value at each voxel. Memory usage scales as $N^3 \times 4$ bytes (e.g., $64^3 \approx 1$ MB) [S2, unofficial].
2.  **3D Texture:** The voxel grid is uploaded to GPU memory as a `Data3DTexture` (or equivalent 3D texture object) [S2, unofficial].
3.  **Transfer Function:** A mapping from scalar value to color and opacity. In the shader-driven approach, this is often a simple two-stop linear interpolation or threshold-based cutoff [S2, unofficial].

**Steps:**
1.  **Data Preparation:** Evaluate the scalar field onto a regular grid. If using procedural generation (e.g., IDW interpolation), compute values for each voxel center $(x_i, y_j, z_k)$ [S2, unofficial].
2.  **Texture Upload:** Upload the scalar array to the GPU as a 3D texture. The texture coordinates map linearly from the grid indices to the $[0, 1]$ range [S2, unofficial].
3.  **Geometry Setup:** Create a unit cube (or bounding box of the volume) to serve as the rendering primitive [S2, unofficial].
4.  **Ray Marching:** For each pixel in the output image, cast a ray through the volume. The ray is parameterized by $t$, where $P(t) = O + tD$ ($O$ is origin, $D$ is direction) [G].
5.  **Sampling:** For each step along the ray:
    *   Determine the 3D texture coordinate corresponding to the ray's current position.
    *   Sample the texture. If "block mode" is enabled, snap the coordinate to the nearest voxel center (nearest-neighbor filtering). If "smooth" mode is enabled, use trilinear interpolation [S2, unofficial].
    *   Check for sentinel values (e.g., $< -1e29$) indicating empty space or nodata. Skip these voxels [S2, unofficial].
6.  **Compositing:** Apply front-to-back compositing. For each step $i$ with sample value $v_i$, compute color $C_i$ and opacity $\alpha_i$ using the transfer function. Update the accumulated color $C_{acc}$ and opacity $\alpha_{acc}$:
    $$
    \alpha_i = \alpha_{global} \cdot \text{opacity}(v_i)
    $$
    $$
    C_{acc} = C_{acc} + C_i \cdot \alpha_i \cdot (1 - \alpha_{acc})
    $$
    $$
    \alpha_{acc} = \alpha_{acc} + \alpha_i \cdot (1 - \alpha_{acc})
    $$
    Stop if $\alpha_{acc}$ exceeds a threshold (e.g., 1.0) or the ray exits the volume [S2, unofficial].
7.  **Display Parameter Application:** Apply global opacity and threshold cutoffs. Voxels with scalar values below the threshold are treated as transparent [S2, unofficial].

**Pseudocode (Shader Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the fragment shader logic
def ray_march_volume(ray_origin, ray_direction, texture_3d, params):
    # Initialize accumulators
    acc_color = [0.0, 0.0, 0.0, 0.0]
    acc_alpha = 0.0
    
    # Ray parameters
    t_min = 0.0
    t_max = 1.0 # Assuming unit cube or normalized bounds
    step_size = 1.0 / params.ray_steps # Fixed step size for simplicity
    
    t = t_min
    while t < t_max and acc_alpha < 0.99:
        # Current position in texture space [0,1]
        pos = ray_origin + ray_direction * t
        tex_coord = map_to_texture_space(pos)
        
        # Sample voxel value
        if params.block_mode:
            # Snap to nearest voxel center
            voxel_val = texture_3d_sample_nearest(tex_coord)
        else:
            # Trilinear interpolation
            voxel_val = texture_3d_sample_trilinear(tex_coord)
            
        # Check for nodata sentinel
        if voxel_val < -1e29:
            t += step_size
            continue
            
        # Apply threshold
        if voxel_val < params.threshold:
            t += step_size
            continue
            
        # Transfer function: map scalar to color and opacity
        # Simple two-stop linear mapping between displayMin and displayMax
        normalized_val = clamp((voxel_val - params.display_min) / (params.display_max - params.display_min), 0.0, 1.0)
        color = lerp(params.color_start, params.color_end, normalized_val)
        opacity = params.opacity * normalized_val # Simplified opacity model
        
        # Front-to-back compositing
        alpha_contribution = opacity * (1.0 - acc_alpha)
        acc_color = acc_color + color * alpha_contribution
        acc_alpha = acc_alpha + alpha_contribution
        
        t += step_size
        
    return acc_color
```

**Parameters:**
*   `dims`: Voxel grid dimensions $(N_x, N_y, N_z)$. Typical: $[64, 64, 64]$ [S2, unofficial].
*   `displayMin/Max`: Range of scalar values mapped to the color transfer function. Typical: $[0, 500]$ [S2, unofficial].
*   `opacity`: Global volume opacity. Typical: $0.4$ [S2, unofficial].
*   `threshold`: Minimum scalar value to render (alpha cutoff). Typical: $0.1$ [S2, unofficial].
*   `ImageResolution`: Output image size. Typical: $[1600, 1200]$ [S1, unofficial].

## Subtopics

### [Simple Shader (3D Texture Mapping)](12431-simple-shader-3d-texture-mapping.md)
Performs basic volume rendering by mapping the volume data to a 3D texture and rendering polygonal geometry generated from the volume's slicing. It differs from advanced shading methods by relying solely on raw interpolated texel values without computing surface normals. Choose this for foundational, low-complexity rendering.

## Operating characteristics
*   **Time Complexity:** $O(P \cdot S)$, where $P$ is the number of pixels and $S$ is the number of ray steps per pixel. Performance is heavily dependent on the number of steps and the efficiency of texture sampling [S2, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the voxel grid storage [S2, unofficial].
*   **Performance:** A $64^3$ grid uses ~1 MB of memory; $256^3$ uses ~64 MB [S2, unofficial]. Real-time tweaking of display parameters (opacity, threshold) is possible without re-uploading the texture, as these are uniform updates in the shader [S2, unofficial].
*   **Strengths:** Direct visualization of internal structures; supports real-time parameter adjustment; simple implementation in modern graphics APIs (e.g., Three.js) [S2, unofficial].
*   **Limitations:** Block mode can appear aliased; smooth mode requires interpolation which may blur sharp features; performance degrades with high resolution or many ray steps [S2, unofficial].
*   **When to use:** When internal density variations are important and surface extraction is not desired [S1, unofficial]. When real-time parameter tweaking is needed [S2, unofficial].
*   **When not to use:** When high-fidelity surface geometry is required (use isosurfaces instead) [S1, unofficial]. When memory is extremely constrained and the volume is very large [S2, unofficial].

## Background
Voxel volume rendering is a standard technique in scientific visualization, often implemented in tools like ParaView and VTK [S1, unofficial]. It allows for the visualization of complex 3D scalar fields, such as concentration data or velocity fields, by leveraging 3D textures and ray marching in shaders [S2, unofficial]. The method has been adapted for various domains, including geological modeling (e.g., drill-hole sample interpolation) and fluid dynamics [S1, S2, unofficial].

## Sources
[S1] cjcamp16/sciviz-paraview, https://github.com/cjcamp16/sciviz-paraview, fetched 2026-10-26 (unofficial)
[S2] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
