# Half-angle slicing

Also called: Halfway vector slicing, Half angle slice axis

## Overview
Half-angle slicing is a two-pass per-slice rendering technique for computing volumetric shadows [S2, unofficial][S6, book]. It solves the problem of efficiently calculating light attenuation through a volume without pre-computing or storing large 3D shadow maps [S3][S6, book]. By orienting slice planes perpendicular to the half-angle vector between the eye and light directions, a single slice geometry can be rendered from both viewpoints, allowing the same rasterized primitives to compute eye-composited color and light transmissivity [S2, unofficial][S5, unofficial]. It differs from standard view-aligned slicing by minimizing the number of slices required to resolve lighting details and from light-aligned slicing by maintaining visibility coherence for the observer [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume texture (scalar field), camera position and projection matrix, light direction (and position for point lights), bounding box of the volume [S1][S5, unofficial].
*   **Outputs:** Final composited image in the eye buffer (framebuffer) [S2, unofficial].

**Data Structures:**
1.  **Light Buffer:** A 2D texture (render target) storing light transmissivity (opacity accumulation) [S2, unofficial][S6, book]. Resolution should be 30%–50% higher than the frame buffer to mitigate shadow boundary artifacts [S6, book]. Initialized to $1 - \text{light intensity}$ (or an arbitrary image for spotlights) [S6, book]. Format: 16-bit floating point (e.g., `R16G16B16A16Sfloat`) to preserve HDR values [S2, unofficial].
2.  **Eye Buffer:** A 2D texture (render target) storing the accumulated color and opacity for the eye pass [S1][S3]. Format: 16-bit floating point [S2, unofficial].
3.  **Push Constants:**
    *   **Light Pass PC:** `glm::mat4 lightMVP` (64 bytes), `float opacityCorrection` (4 bytes), `float densityMin` (4 bytes), `float densityMax` (4 bytes). Total: 76 bytes [S2, unofficial].
    *   **Eye Pass PC:** `glm::mat4 eyeMVP` (64 bytes), `glm::mat4 lightMVP` (64 bytes), `float opacityCorrection` (4 bytes), `float ambient` (4 bytes), `float densityMin` (4 bytes), `float densityMax` (4 bytes). Total: 144 bytes [S2, unofficial]. Note: Vulkan guarantees only 128 bytes; drivers supporting $\ge 256$ bytes are required [S2, unofficial].
4.  **Proxy Geometry:** A mesh representing the intersection of a slice plane with the volume's bounding box [S5, unofficial]. Constructed by finding intersection points of the 12 box edges with the plane, sorting them counter-clockwise around the centroid, and fan-triangulating to the centroid [S5, unofficial].

**Parameters:**
*   `numSlices`: Default 256 [S2, unofficial].
*   `kLightSize`: Resolution of the light buffer (e.g., 512x512) [S2, unofficial].

**Algorithm Steps:**

1.  **Coordinate Transformation:** Transform the camera position and light direction into the object's local (inverted) space using the inverse model matrix $M^{-1}$ [S5, unofficial].
    $$ \mathbf{v}_{cam} = \text{normalize}(M^{-1}\mathbf{p}_{cam} - M^{-1}\mathbf{p}_{obj}) $$
    $$ \mathbf{v}_{light} = \text{normalize}(M^{-1}\mathbf{v}_{light\_world}) $$

2.  **Half-Vector Calculation:** Compute the dot product of the normalized inverted camera vector and normalized light vector to determine orientation [S5, unofficial][S6, book].
    $$ d = \mathbf{v}_{cam} \cdot \mathbf{v}_{light} $$
    Determine the half-vector $\mathbf{h}$:
    *   If $d \ge 0$: $\mathbf{h} = \text{normalize}(\mathbf{v}_{cam} + \mathbf{v}_{light})$ [S5, unofficial][S6, book].
    *   If $d < 0$: $\mathbf{h} = \text{normalize}(-\mathbf{v}_{cam} + \mathbf{v}_{light})$ [S5, unofficial][S6, book].
    *   *Note:* The half-vector always lies between the eye and light directions, fixing the slice traversal order to front-to-back from the light's perspective [S2, unofficial].

3.  **Slice Bounds and Spacing:** Project the 8 bounding box vertices onto $\mathbf{h}$ to find min/max distances [S5, unofficial].
    $$ d_{min} = \min_{i \in 0..7} (\mathbf{h} \cdot \mathbf{b}_i), \quad d_{max} = \max_{i \in 0..7} (\mathbf{h} \cdot \mathbf{b}_i) $$
    Calculate the length of the slice volume along $\mathbf{h}$:
    $$ L = \| \mathbf{h} \cdot d_{max} - \mathbf{h} \cdot d_{min} \| $$
    Calculate step size:
    $$ s = \frac{L}{\text{numSlices}} $$
    *Note:* The slice spacing $d_s$ along the half-angle axis relates to the desired view-direction spacing $d_v$ and angle $\theta$ between view and light by $d_s = \frac{d_v}{\cos(\theta/2)}$ [S6, book].

4.  **Rendering Loop:** For each slice index $k$ from $0$ to $\text{numSlices}-1$:
    a.  **Generate Geometry:** Calculate slice center $C = \mathbf{h} \cdot (\text{min\_dist} + k \cdot s + s/2)$ [S5, unofficial]. Intersect the plane defined by $C$ and normal $\mathbf{h}$ with the box edges. Sort intersection points counter-clockwise around $C$ using the cross-product test with $\mathbf{h}$ [S5, unofficial]. Construct a fan mesh from the centroid to the sorted boundary vertices [S5, unofficial].
    b.  **Pass 1 (Eye Pass):** Render the slice from the eye's viewpoint [S2, unofficial]. Bind the Light Buffer as a texture. Sample the light transmissivity from the Light Buffer at the fragment's screen position. Compute the fragment color by multiplying the material color (from volume classification) by the sampled light transmissivity. Composite the result into the Eye Buffer using the "over" operator [S1][S3].
    c.  **Pass 2 (Light Pass):** Render the *same* slice from the light's viewpoint [S2, unofficial]. Bind the Light Buffer as a render target. Compute the fragment opacity (alpha) from the volume classification. Accumulate the opacity into the Light Buffer using the "over" operator [S2, unofficial][S3][S6, book]. *Note:* The volume is rendered in front-to-back order with respect to the light [S3][S6, book].

5.  **Final Composition:** Composite the Eye Buffer to the screen [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def half_angle_slice_render(volume, camera, light, bbox, num_slices=256):
    # 1. Transform to local space
    inv_model = inverse(camera.model_matrix)
    cam_pos_local = inv_model @ camera.position
    cam_dir_local = normalize(camera.forward)
    light_dir_local = normalize(light.direction)

    # 2. Compute Half Vector
    dot_prod = dot(cam_dir_local, light_dir_local)
    if dot_prod >= 0:
        half_vec = normalize(cam_dir_local + light_dir_local)
    else:
        half_vec = normalize(-cam_dir_local + light_dir_local)

    # 3. Compute Bounds
    box_verts = get_bbox_vertices(bbox)
    projections = [dot(half_vec, v) for v in box_verts]
    min_dist = min(projections)
    max_dist = max(projections)
    
    length = max_dist - min_dist
    step_size = length / num_slices
    
    # Initialize Light Buffer
    light_buffer = create_texture(width=frame_width * 1.5, height=frame_height * 1.5, format=RGBA16F)
    clear(light_buffer, 1.0) 
    
    # Initialize Eye Buffer
    eye_buffer = create_texture(width=frame_width, height=frame_height, format=RGBA16F)
    clear(eye_buffer, 0.0)

    # 4. Render Slices
    for k in range(num_slices):
        current_dist = min_dist + step_size * (k + 0.5)
        slice_center = half_vec * current_dist
        
        intersections = intersect_plane_with_bbox(slice_center, half_vec, bbox)
        if len(intersections) < 3:
            continue
            
        centroid = sum(intersections) / len(intersections)
        sorted_intersections = sort_ccw(intersections, centroid, half_vec)
        
        mesh = create_fan_mesh(centroid, sorted_intersections)
        
        # --- Pass 1: Eye Pass ---
        set_render_target(eye_buffer)
        set_view_projection(camera.projection_matrix, camera.view_matrix)
        set_push_constants(eye_pass_pc, eye_mvp=eye_buffer_mvp, light_mvp=light_buffer_mvp, opacity_correction=0.0, ambient=0.0, density_min=0.0, density_max=1.0)
        bind_texture(light_buffer, unit=0)
        draw_mesh(mesh)
        
        # --- Pass 2: Light Pass ---
        set_render_target(light_buffer)
        set_view_projection(light.projection_matrix, light.view_matrix)
        set_push_constants(light_pass_pc, light_mvp=light_buffer_mvp, opacity_correction=0.0, density_min=0.0, density_max=1.0)
        draw_mesh(mesh)

    # 5. Composite to Screen
    set_render_target(screen)
    draw_quad(eye_buffer)
```

## Subtopics

### Two-pass volume rendering
Uses two passes per slice but typically aligns slices with the view or light direction, requiring separate geometries or more complex state management for shadows. Half-angle slicing uses a single slice geometry for both passes [S2, unofficial][S5, unofficial].

### Two-Pass Ray Marching Volume Rendering
Uses ray marching within passes rather than slice-based rasterization. Half-angle slicing relies on hardware rasterization of proxy geometry, which is generally faster for large volumes but less accurate for complex internal structures [S2, unofficial].

### Multi-pass GPU Ray-Casting with Early Ray Termination
Uses ray casting with early termination to save computation. Half-angle slicing does not inherently support early termination in the same way, as it relies on fixed slice traversal, but it avoids the overhead of per-ray intersection tests [S3].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of samples per slice (determined by rasterization coverage) [G]. The cost is dominated by two rasterization passes per slice [S2, unofficial].
*   **Memory Complexity:** $O(W \cdot H)$ for the light and eye buffers, where $W, H$ are buffer dimensions. This is significantly lower than 3D shadow maps $O(V^3)$ [S6, book].
*   **Performance:** Achieves high-resolution shadow effects without pre-computation [S2, unofficial]. Performance is limited by the number of slices and the resolution of the light buffer. Using "render to texture" extensions avoids costly copies between buffers [S6, book].
*   **Strengths:** Decouples light transport from volume resolution; avoids 3D shadow map storage; minimizes slice count compared to view-aligned slicing for oblique angles [S4, unofficial][S6, book].
*   **Limitations:** Artifacts at shadow boundaries if opacity jumps sharply (mitigated by higher light buffer resolution) [S6, book]. Attenuation leakage (blurry shadows) can occur if view and light are perpendicular and a 2D buffer is used without sufficient resolution [S6, book]. Requires GPU driver support for large push constants (256 bytes) [S2, unofficial].
*   **When to use:** When volumetric shadows are required and the light/view angle is not perpendicular. Not suitable for real-time if the number of slices must be very high to resolve fine volume details, as it doubles the rasterization cost per slice.

## Background
Originated in the context of GPU-based volumetric shadow rendering, specifically proposed to avoid the storage and pre-computation costs of volumetric shadow maps [S2, unofficial][S6, book]. It was detailed in *GPU Gems* (Chapter 39) and the *Visualization Handbook* [S1][S6, book]. The technique leverages the geometric property that the half-angle vector provides a common slicing axis for both eye and light visibility.

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S3] PDF Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S4] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S5] dokipen3d/HalfAngleSlicing, https://github.com/dokipen3d/HalfAngleSlicing, fetched 2026-10-27 (unofficial)
[S6] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
