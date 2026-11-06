# Half-angle slicing

Also called: Halfway vector slicing, Half angle slice axis

## Overview
Half-angle slicing is a two-pass per-slice rendering technique for volumetric shadows that orients slice planes perpendicular to the half-angle vector between the eye and light directions, allowing a single slice to be rendered from both viewpoints to compute light attenuation and eye compositing [S1, unofficial][S5, book]. It differs from standard view-aligned or light-aligned slicing by minimizing the number of slices required to resolve lighting details and decoupling light transport computation from data volume resolution, thereby avoiding attenuation leakage artifacts common in 2D shadow buffer approaches [S4, unofficial][S5, book].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume scalar field, transfer function (density, color, opacity), camera position/view direction, light direction, bounding box of the volume.
*   **Outputs:** Final composited image in the eye buffer.

**Data Structures:**
*   **Light Buffer:** Offscreen render target (texture) storing light transmissivity/attenuation. Initialized to $1 - \text{light intensity}$ [S5, book]. Resolution should be 30%–50% higher than the frame buffer to mitigate shadow boundary artifacts [S5, book]. Format: Floating-point (e.g., `R16G16B16A16Sfloat`) to preserve HDR values [S1, unofficial].
*   **Eye Buffer:** Offscreen render target for accumulating the final image. Format: Same as Light Buffer [S1, unofficial].
*   **Push Constants:**
    *   `LightPC` (76 bytes): `glm::mat4 lightMVP` (64 bytes), `float opacityCorrection` (4 bytes), `float densityMin` (4 bytes), `float densityMax` (4 bytes) [S1, unofficial].
    *   `EyePC` (144 bytes): `glm::mat4 eyeMVP` (64 bytes), `glm::mat4 lightMVP` (64 bytes), `float opacityCorrection` (4 bytes), `float ambient` (4 bytes), `float densityMin` (4 bytes), `float densityMax` (4 bytes) [S1, unofficial]. Note: Vulkan guarantees only 128 bytes for push constants; this requires a driver supporting $\ge 256$ bytes [S1, unofficial].
*   **Bounding Box:** 8 vertices in local space at $(\pm 0.5, \pm 0.5, \pm 0.5)$ [S4, unofficial].

**Parameters:**
*   `kReferenceSlices`: Default number of slices (e.g., 256) [S1, unofficial].
*   `kLightSize`: Resolution of the light buffer (e.g., 512) [S1, unofficial].
*   `opacityCorrection`: Factor to maintain image intensity invariance across discrete slice counts [S1, S4, unofficial].

**Algorithm Steps:**

1.  **Coordinate Transformation:** Transform the camera position and light direction into the object's local (inverted) space using the inverse model matrix $M^{-1}$ [S4, unofficial].
    *   `invCamPosition = M.inverse.MultiplyPoint3x4(cameraPosition)`
    *   `invLightVector = M.inverse.MultiplyVector(lightDirection).normalized`
    *   `invCamVector = (invCamPosition - cameraPosition).normalized` (Note: Source [S4, unofficial] defines `cameraVector` as `gameObject.transform.position - mainCamera.transform.position`, normalized. In local space, this is the direction from camera to object).

2.  **Compute Half-Angle Vector:**
    *   Calculate dot product: `DotProduct = Dot(invLightVector, invCamVector)` [S4, unofficial].
    *   Determine inversion flag: `bViewInverted = (DotProduct >= 0)` [S4, unofficial].
    *   Compute half vector:
        $$ \mathbf{h} = \frac{(\mathbf{v}_{inv} \cdot \text{sign} + \mathbf{l})}{\|\mathbf{v}_{inv} \cdot \text{sign} + \mathbf{l}\|} $$
        Where $\mathbf{v}_{inv}$ is `invCamVector.normalized`, $\mathbf{l}$ is `invLightVector.normalized`, and $\text{sign} = -1$ if `bViewInverted` is false, else $1$. Specifically from [S4, unofficial]:
        ```python
        if bViewInverted:
            halfVector = (invCamVector.normalized + invLightVector.normalized).normalized
        else:
            halfVector = (-invCamVector.normalized + invLightVector.normalized).normalized
        ```

3.  **Determine Slice Bounds and Spacing:**
    *   Project the 8 bounding box vertices onto $\mathbf{h}$ to find min/max distances:
        $$ d_i = \mathbf{h} \cdot \mathbf{boxVertices}_i $$
        $$ min\_dist = \min(d_i), \quad max\_dist = \max(d_i) $$
    *   Calculate total length along half-vector:
        $$ L = \| \mathbf{h} \cdot max\_dist - \mathbf{h} \cdot min\_dist \| $$
    *   Calculate step size:
        $$ stepSize = \frac{L}{numSlices} $$
    *   *Note:* The slice spacing along the half-vector direction $d_s$ relates to the desired view-direction spacing $d_v$ and angle $\theta$ between view and light by $d_s = \frac{d_v}{\cos(\theta/2)}$ [S5, book].

4.  **Iterate Slices:** For each slice $s$ from $0$ to $numSlices - 1$:

    A. **Calculate Slice Plane Center:**
        $$ C = \mathbf{h} \cdot min\_dist + \mathbf{h} \cdot (s \cdot stepSize) + \frac{\mathbf{h} \cdot stepSize}{2} $$

    B. **Generate Proxy Geometry (Slice Mesh):**
        *   Intersect the plane defined by center $C$ and normal $\mathbf{h}$ with the volume bounding box edges to get intersection points [S4, unofficial].
        *   Sort intersection points counter-clockwise around $C$ using the cross-product test:
            ```python
            def sort_key(A, B):
                # Returns 1 if B is clockwise from A relative to h, else -1
                if Dot(h, Cross(A - C, B - C)) <= 0:
                    return 1
                else:
                    return -1
            ```
        *   Construct mesh: Add centroid as vertex 0. Connect centroid to sorted boundary vertices to form triangles. Handle winding order based on `bViewInverted` [S4, unofficial]:
            ```python
            indices = []
            for p in range(num_boundary_verts):
                if bViewInverted:
                    indices.append((p + 2) % num_boundary_verts + 1)
                    indices.append((p + 1) % num_boundary_verts + 1)
                else:
                    indices.append((p + 1) % num_boundary_verts + 1)
                    indices.append((p + 2) % num_boundary_verts + 1)
            ```

    C. **Pass 1: Eye Pass (Render from Camera):**
        *   Render the slice mesh using the camera's view/projection matrices.
        *   In the fragment shader:
            1.  Sample the volume density at the fragment's 3D position.
            2.  Evaluate transfer function for color $C_{color}$ and opacity $\alpha$.
            3.  Sample the **Light Buffer** (bound as texture) at the fragment's screen position to get available light intensity $I_{light}$.
            4.  Modulate color: $C_{out} = C_{color} \times I_{light}$.
            5.  Composite into the **Eye Buffer** using the "over" operator (or "under" blend if compositing back-to-front, but standard is front-to-back for eye pass relative to light attenuation accumulation) [S1, S3, unofficial][S5, book].
            *   *Correction:* Source [S3, unofficial] states Eye pass uses "Under Blend" to composite slice colors using precomputed light attenuation. Source [S5, book] states the slice is rendered from observer's POV using results from previous light pass to modulate brightness.

    D. **Pass 2: Light Pass (Render from Light):**
        *   Render the **same** slice mesh using the light's view/projection matrices.
        *   Render order must be front-to-back with respect to the light [S3, unofficial][S5, book].
        *   In the fragment shader:
            1.  Sample volume density.
            2.  Evaluate opacity $\alpha$.
            3.  Accumulate opacity into the **Light Buffer** using the "over" operator to compute transmissivity [S3, unofficial][S5, book].
            4.  The Light Buffer is initialized to $1 - \text{light intensity}$ [S5, book].
            5.  Use "render to texture" hardware extension to bind the pixel buffer directly as a texture to avoid copy overhead [S5, book].

    *   *Ordering Note:* The half-vector always lies between eye and light, fixing slice traversal order to front-to-back from the light. No user toggle is needed [S1, unofficial]. If `DotProduct < 0`, the eye pass renders back-to-front, but light pass remains front-to-back [S5, book].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def half_angle_slice_render(volume, camera, light, num_slices=256):
    # 1. Transform to local space
    inv_model = volume.model_matrix.inverse()
    inv_cam_pos = inv_model.transform_point(camera.position)
    inv_light_dir = inv_model.transform_vector(light.direction).normalized()
    inv_cam_dir = (inv_cam_pos - camera.position).normalized() # Direction from cam to object in local space? 
    # Note: Source [S4, unofficial] uses gameObject.transform.position - mainCamera.transform.position. 
    # In local space, this is the vector from camera to object origin.
    
    # 2. Compute Half Vector
    dot_prod = inv_light_dir.dot(inv_cam_dir)
    b_view_inverted = dot_prod >= 0
    
    if b_view_inverted:
        half_vec = (inv_cam_dir + inv_light_dir).normalized()
    else:
        half_vec = (-inv_cam_dir + inv_light_dir).normalized()
        
    # 3. Bounds
    box_verts = get_box_vertices() # 8 vertices
    projections = [half_vec.dot(v) for v in box_verts]
    min_dist = min(projections)
    max_dist = max(projections)
    
    length = (half_vec * max_dist - half_vec * min_dist).magnitude()
    step_size = length / num_slices
    
    # Initialize Buffers
    light_buffer = create_texture(width=kLightSize, height=kLightSize, format=HDR)
    light_buffer.clear(1.0 - light.intensity) # Init to 1 - light intensity
    eye_buffer = create_texture(width=frame_width, height=frame_height, format=HDR)
    
    for s in range(num_slices):
        # 4. Slice Center
        center = half_vec * min_dist + half_vec * (s * step_size) + (half_vec * step_size) / 2.0
        
        # 5. Generate Mesh
        intersections = intersect_plane_with_box(center, half_vec, box_verts)
        sorted_intersections = sort_ccw(intersections, center, half_vec)
        mesh = build_fan_mesh(sorted_intersections, center, b_view_inverted)
        
        # 6. Eye Pass
        set_viewport(eye_buffer)
        set_shader(EyeShader, {
            'eye_mvp': camera.get_mvp(),
            'light_mvp': light.get_mvp(),
            'light_buffer': light_buffer,
            'opacity_correction': opacity_correction_factor,
            'density_min': volume.density_min,
            'density_max': volume.density_max
        })
        draw(mesh)
        
        # 7. Light Pass
        set_viewport(light_buffer)
        set_shader(LightShader, {
            'light_mvp': light.get_mvp(),
            'opacity_correction': opacity_correction_factor,
            'density_min': volume.density_min,
            'density_max': volume.density_max
        })
        draw(mesh)
        
    # 8. Composite to Screen
    set_viewport(screen)
    set_shader(CompositeShader, {'eye_buffer': eye_buffer})
    draw_quad()
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of samples per slice (or pixels per slice). Two passes per slice.
*   **Memory Complexity:** $O(W \cdot H)$ for the two 2D buffers (Light and Eye), where $W, H$ are buffer resolutions. Significantly less than 3D volumetric shadow maps [S5, book].
*   **Performance:** High performance on modern GPUs due to "render to texture" optimization avoiding copies [S5, book]. Resolution of light buffer should be 30-50% higher than frame buffer to reduce artifacts [S5, book].
*   **Strengths:** Avoids attenuation leakage (blurry shadows) by decoupling light transport from data resolution [S5, book]. No pre-computation or 3D shadow map storage required [S2][S5, book]. Single additional 2D buffer [S5, book].
*   **Limitations:** Artifacts at shadow boundaries when opacity jumps sharply [S5, book]. Requires GPU driver supporting $\ge 256$ bytes push constants for the eye pass (144 bytes used) [S1, unofficial]. Attenuation leakage can still occur if view and light are perpendicular and a 2D buffer is used without sufficient resolution [S5, book].
*   **When to use:** Interactive volume rendering with dynamic lights/cameras where shadow quality is important.
*   **When not to use:** When push constant size is limited to 128 bytes. When extreme perpendicular light/view angles cause severe leakage despite resolution boost.

## Background
Half-angle slicing was proposed for volume shadow computation to allow a single slice to serve both observer and light attenuation calculations [S2][S5, book]. It is described in *GPU Gems* (Chapter 39) and *The Visualization Handbook* (Chapter 9) [S1, unofficial][S5, book]. It achieves high-resolution shadow map effects without pre-computation [S2].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S3] CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] dokipen3d/HalfAngleSlicing, https://github.com/dokipen3d/HalfAngleSlicing, fetched 2026-10-27 (unofficial)
[S5] Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
