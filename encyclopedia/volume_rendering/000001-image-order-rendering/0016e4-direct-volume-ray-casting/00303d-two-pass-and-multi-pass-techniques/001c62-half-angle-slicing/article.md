# Half-angle slicing

Also called: Halfway vector slicing, Half angle slice axis

## Overview
Half-angle slicing is a two-pass per-slice rendering technique for computing volumetric shadows [S1, unofficial][S5, book]. It orients slice planes perpendicular to the half-angle vector between the eye and light directions, allowing a single slice to be rendered from both viewpoints to compute light attenuation and final color [S1, unofficial][S2][S5, book]. This method achieves high-resolution shadow map effects without pre-computation or storage, using only a single additional 2D buffer instead of a potentially large 3D volumetric shadow map [S2][S5, book]. It avoids attenuation leakage artifacts (blurry shadows and overly dark surfaces) that occur when view and light directions are perpendicular and a standard 2D shadow buffer is used [S5, book].

## How it works
**Inputs and Outputs:**
- **Inputs:** Volume data (scalar field), camera position and view direction ($v$), light direction ($l$), number of slices ($N$), desired slice spacing along the view direction ($d_v$).
- **Outputs:** Final composited image in an eye buffer, which is then composited to the screen.

**Data Structures:**
- **Light Buffer:** An offscreen render buffer (texture) storing light transmissivity/attenuation. Initialized to $1 - \text{light intensity}$ [S5, book]. Resolution should be 30%–50% higher than the frame buffer to reduce shadow boundary artifacts [S5, book]. Format: 16-bit floating point per channel (e.g., `R16G16B16A16Sfloat`) to preserve HDR values [S1, unofficial].
- **Eye Buffer:** An offscreen render buffer for accumulating the final color [S1, unofficial].
- **Push Constants:**
    - **Light Pass:** `lightMVP` (4x4 matrix, 64 bytes), `opacityCorrection` (float, 4 bytes), `densityMin` (float, 4 bytes), `densityMax` (float, 4 bytes). Total: 76 bytes [S1, unofficial].
    - **Eye Pass:** `eyeMVP` (4x4 matrix, 64 bytes), `lightMVP` (4x4 matrix, 64 bytes), `opacityCorrection` (float, 4 bytes), `ambient` (float, 4 bytes), `densityMin` (float, 4 bytes), `densityMax` (float, 4 bytes). Total: 144 bytes [S1, unofficial]. Note: Vulkan guarantees only 128 bytes for push constants; 144 bytes requires drivers supporting at least 256 bytes (standard on modern desktop GPUs) [S1, unofficial].
- **Bounding Box:** 8 vertices in local space at $(\pm 0.5, \pm 0.5, \pm 0.5)$ [S4, unofficial].
- **Edge Directions:** 12 edge direction vectors of the unit cube for intersection testing [S4, unofficial].

**Algorithm Steps:**

1.  **Compute Half-Angle Vector:**
    Transform camera position and light vector into the object's local (inverted) space using the inverse model matrix $M^{-1}$ [S4, unofficial]:
    $$ \text{invCamPosition} = M^{-1} \cdot \text{camPosition} $$
    $$ \text{invLightVector} = \text{normalize}(M^{-1} \cdot \text{lightVector}) $$
    $$ \text{invCamVector} = \text{normalize}(\text{invCamPosition}) $$
    Calculate the dot product:
    $$ \text{dotProduct} = \text{invLightVector} \cdot \text{invCamVector} $$
    Determine if view is inverted:
    $$ bViewInverted = (\text{dotProduct} \ge 0) $$
    Compute the half-vector $h$:
    $$ h = \text{normalize}((bViewInverted ? \text{invCamVector} : -\text{invCamVector}) + \text{invLightVector}) $$
    [S4, unofficial]

2.  **Determine Slice Bounds and Spacing:**
    Project the 8 bounding box vertices onto $h$ to find min and max distances:
    $$ \text{dist}_i = h \cdot \text{boxVertices}_i \quad \text{for } i=0..7 $$
    $$ \text{min\_dist} = \min(\text{dist}_i), \quad \text{max\_dist} = \max(\text{dist}_i) $$
    Calculate the length of the volume along the half-vector:
    $$ L = \| h \cdot \text{max\_dist} - h \cdot \text{min\_dist} \| $$
    Calculate step size:
    $$ \text{stepSize} = L / N $$
    [S4, unofficial]
    The slice spacing $d_s$ along the half-vector direction is related to the desired view spacing $d_v$ and angle $\theta$ between $v$ and $l$ by:
    $$ d_s = \frac{d_v}{\cos(\theta/2)} $$
    [S5, book]

3.  **Iterate Through Slices:**
    For each slice $s$ from $0$ to $N-1$:
    
    a.  **Calculate Slice Plane Center:**
        $$ \text{newpos} = h \cdot \text{min\_dist} $$
        $$ C = \text{newpos} + (h \cdot s \cdot \text{stepSize}) + \frac{h \cdot \text{stepSize}}{2} $$
        [S4, unofficial]

    b.  **Generate Proxy Geometry (Slice Mesh):**
        Intersect the plane defined by center $C$ and normal $h$ with the bounding box edges to get intersection points.
        Sort intersection vertices in counter-clockwise order around $C$ using the cross-product test:
        $$ \text{compare}(A, B) = \text{sign}(h \cdot ((A-C) \times (B-C))) $$
        If result $\le 0$, return 1 (swap); else return -1.
        [S4, unofficial]
        Construct a mesh by adding a central vertex (centroid of intersection points) and connecting it to the sorted boundary vertices to form triangles. Handle winding order based on $bViewInverted$:
        - If inverted: indices are $((p+2)\%N+1)+\text{totalVerts}$ then $((p+1)\%N+1)+\text{totalVerts}$.
        - Otherwise: indices are $((p+1)\%N+1)+\text{totalVerts}$ then $((p+2)\%N+1)+\text{totalVerts}$.
        [S4, unofficial]

    c.  **First Pass (Light Pass):**
        Render the slice from the light's viewpoint.
        - Render order: Front-to-back with respect to the light [S3, unofficial][S5, book].
        - Fragment Shader: Accumulate opacity using the "over" operator to compute light transmissivity. Blend alpha values only [S3, unofficial].
        - Store result in the Light Buffer.
        [S1, S3, unofficial][S5, book]

    d.  **Second Pass (Eye Pass):**
        Render the same slice from the eye's viewpoint.
        - Render order:
            - If $\text{dot}(l, v) > 0$: Front-to-back with respect to the observer [S5, book].
            - If $\text{dot}(l, v) < 0$: Back-to-front with respect to the observer [S5, book].
        - Fragment Shader: Look up light color/transmissivity from the Light Buffer (bound as a texture). Multiply material color by this light color. Composite into the Eye Buffer.
        [S1, S3, unofficial][S5, book]

4.  **Final Composition:**
    Composite the Eye Buffer to the screen [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def half_angle_slice_render(volume_data, camera, light, num_slices, d_v):
    # 1. Compute Half-Angle Vector
    # Transform to local space
    inv_cam_pos = volume_data.inverse_model_matrix * camera.position
    inv_light_vec = normalize(volume_data.inverse_model_matrix * light.direction)
    inv_cam_vec = normalize(inv_cam_pos)
    
    dot_product = dot(inv_light_vec, inv_cam_vec)
    b_view_inverted = dot_product >= 0
    
    cam_vec_for_half = inv_cam_vec if b_view_inverted else -inv_cam_vec
    half_vector = normalize(cam_vec_for_half + inv_light_vec)
    
    # 2. Determine Bounds
    box_vertices = [(-0.5, -0.5, 0.5), (-0.5, 0.5, 0.5), (0.5, 0.5, 0.5), (0.5, -0.5, 0.5),
                    (-0.5, -0.5, -0.5), (-0.5, 0.5, -0.5), (0.5, 0.5, -0.5), (0.5, -0.5, -0.5)]
    
    projections = [dot(half_vector, v) for v in box_vertices]
    min_dist = min(projections)
    max_dist = max(projections)
    
    half_vector_length = magnitude(half_vector * max_dist - half_vector * min_dist)
    step_size = half_vector_length / num_slices
    
    # Initialize Light Buffer (e.g., 1.0 for full light, or 1 - intensity)
    light_buffer = initialize_buffer(1.0) 
    
    eye_buffer = empty_buffer()
    
    # 3. Iterate Slices
    for s in range(num_slices):
        # Calculate Slice Center
        newpos = half_vector * min_dist
        center = newpos + (half_vector * s * step_size) + (half_vector * step_size / 2)
        
        # Generate Mesh
        intersections = intersect_plane_with_box(center, half_vector, box_vertices)
        sorted_vertices = sort_ccw(intersections, center, half_vector)
        mesh = create_fan_mesh(sorted_vertices, center, b_view_inverted)
        
        # Pass 1: Light Pass
        # Render from light POV, front-to-back relative to light
        light_pass_result = render_slice(mesh, light.camera_matrix, volume_data, light_buffer)
        # Update light buffer with accumulated opacity/transmissivity
        light_buffer = accumulate_opacity(light_buffer, light_pass_result)
        
        # Pass 2: Eye Pass
        # Render from eye POV
        # Determine order based on dot product of light and view directions
        # Note: inv_light_vec and inv_cam_vec are in local space, 
        # but the sign of dot product is invariant to rotation, so we can use them.
        if dot_product > 0:
            render_order = "front_to_back"
        else:
            render_order = "back_to_front"
            
        eye_pass_result = render_slice(mesh, camera.camera_matrix, volume_data, light_buffer, render_order)
        eye_buffer = composite(eye_buffer, eye_pass_result)
        
    return eye_buffer

def sort_ccw(vertices, center, normal):
    # Sort vertices around center using cross product test with normal
    # Returns vertices in counter-clockwise order around normal
    return sorted(vertices, key=lambda v: angle_from_reference(v, center, normal))

def create_fan_mesh(sorted_vertices, center, b_view_inverted):
    # Add centroid
    centroid = sum(sorted_vertices) / len(sorted_vertices)
    vertices = [centroid] + sorted_vertices
    
    indices = []
    n = len(sorted_vertices)
    for p in range(n):
        if b_view_inverted:
            indices.append((p + 2) % n + 1)
            indices.append((p + 1) % n + 1)
        else:
            indices.append((p + 1) % n + 1)
            indices.append((p + 2) % n + 1)
            
    return Mesh(vertices, indices)
```

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of slices and $V$ is the number of vertices per slice (typically small, bounded by box intersections). Each slice requires two render passes.
- **Memory Complexity:** $O(R^2)$ for the light and eye buffers, where $R$ is the buffer resolution. This is significantly less than $O(R^3)$ for volumetric shadow maps [S5, book].
- **Performance:** Performance is comparable to standard two-pass volume rendering but adds the cost of the light pass. Using "render to texture" hardware extensions avoids copy operations between the light buffer and texture [S5, book].
- **Strengths:**
    - Avoids attenuation leakage artifacts common in perpendicular view/light configurations [S5, book].
    - No pre-computation or large 3D shadow map storage required [S2][S5, book].
    - Decouples light transport computation from data volume resolution [S5, book].
- **Limitations:**
    - Artifacts may appear at shadow boundaries if opacity jumps sharply [S5, book].
    - Requires GPU driver support for large push constants (144 bytes) if using the specific Vulkan implementation described [S1, unofficial].
    - When view and light directions are perpendicular, the half-angle vector is at 45 degrees, potentially requiring more slices to maintain resolution along the view direction compared to view-aligned slicing [S5, book].
- **When to use:** Interactive volume rendering with dynamic lights and cameras where shadow quality is important and memory is constrained [S2][S5, book].
- **When not to use:** When extreme shadow boundary sharpness is required and higher resolution buffers are not feasible, or on hardware with strict push constant limits [S1, unofficial].

## Background
Half-angle slicing was proposed for volume shadow computation in [18] (cited in [S2][S5, book]) and implemented in interactive volume rendering systems [S2]. It is described in *The Visualization Handbook* (Chapter 9) [S5, book] and demonstrated in GPU Gems implementations [S1, unofficial]. The technique modifies proxy geometry slicing to allow rendering from both eye and light points of view using two different frame buffers [S3, unofficial]. It is used in scientific visualization and medical imaging applications [S3, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] dokipen3d/HalfAngleSlicing, https://github.com/dokipen3d/HalfAngleSlicing, fetched 2026-10-27 (unofficial)
[S5] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
