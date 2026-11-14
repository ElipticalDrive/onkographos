# Ray Casting with Depth/Alpha Test for Empty Space Skipping

Also called: Empty Space Skipping via OpenGL Tests, Ray Casting using Alpha and Depth Tests, Texture Slab Volume Rendering with GPU-based Acceleration, Westermann & Sevenich Empty Space Skipping, Westermann & Sevenich Technique

## Overview
This method accelerates ray-based volume rendering by using the graphics hardware's depth buffer and alpha test to determine the entry and exit points of the volume along each pixel's ray, thereby skipping sampling in empty space [S3, unofficial]. Unlike its parent method, which relies on a pre-computed or dynamically updated secondary data structure (like an octree) to query occupancy, this technique constructs the acceleration structure (Slab Silhouette Maps) on the fly during the rendering pass using standard OpenGL state [S1, unofficial][S3, unofficial]. It sets itself apart from sibling methods like [Space Leaping](5863-space-leaping.md) by avoiding CPU-side preprocessing and hierarchical traversal logic, instead leveraging hardware z-occlusion culling to identify active segments [S1, unofficial].

## How it works
The method operates in three main phases: pre-processing (gradient calculation), acceleration structure generation (Slab Silhouette Maps), and ray casting with skipping.

### Inputs and Outputs
*   **Input:** A 3D volume dataset (scalar values), a transfer function mapping scalars to color and opacity, and a camera configuration (view and projection matrices).
*   **Output:** A 2D image (framebuffer) with the composited volume.

### Data Structures
1.  **Volume Data:** Stored in CPU memory or GPU texture memory.
2.  **Gradient Volume:** Pre-computed gradients for each voxel, stored in CPU memory or a GPU texture (typically RGB format for normal vectors) [S3, unofficial].
3.  **Texture Slabs:** The 3D volume is sliced into 2D textures. Slices are grouped into "slabs" to reduce the number of rendering passes [S1, unofficial].
4.  **Depth Buffer:** Used to store the near and far intersection distances. It is initialized and queried in two separate passes [S3, unofficial].
5.  **Slab Silhouette Maps (SSM):** Implicitly stored in the depth buffer. These maps identify empty spaces and terminated rays by storing the depth values of the first and last visible fragments of each slab [S1, unofficial].

### Steps

1.  **Pre-compute Gradients and Illumination (CPU):**
    *   Calculate the gradient for each volume element using central differences [S3, unofficial].
    *   Perform illumination and shading calculations based on the lighting model and store the results in main memory or a texture [S3, unofficial].
    *   *Note:* This step is performed once on the CPU, unlike methods that compute lighting per-sample on the GPU [S3, unofficial].

2.  **Prepare Textures and Display Lists:**
    *   For each axis (X, Y, Z), create 2D textures for planes parallel to that axis [S3, unofficial].
    *   Load these textures into GPU memory [S3, unofficial].
    *   Create display lists for front-to-back and back-to-front rendering orders [S3, unofficial].

3.  **Determine Rendering Axis:**
    *   Determine the object axis with the smallest angle to the projection plane [S3, unofficial].
    *   Select the farthest texture from the camera in the corresponding set [S3, unofficial].
    *   Rotate the camera and draw the textures starting from the selected one [S3, unofficial].

4.  **Generate Slab Silhouette Maps (SSM) via Two Passes:**
    The goal is to find the near ($P$) and far ($Q$) intersection points of the ray with the volume.

    *   **Pass 1 (Find Near Point):**
        *   Initialize the depth buffer to 1.0 (maximum value) [S3, unofficial].
        *   Set depth write criterion to `GL_LESS` [S3, unofficial].
        *   Enable the Alpha Test with `GL_GREATER` and a reference value $> 0$ (e.g., 0.001) to discard fragments with zero opacity [S3, unofficial].
        *   Render the volume slabs from front to back.
        *   The depth buffer now contains the distances of the closest significant fragments to the projection plane for each pixel [S3, unofficial].

    *   **Pass 2 (Find Far Point):**
        *   Initialize the depth buffer to 0.0 (minimum value) [S3, unofficial].
        *   Set depth write criterion to `GL_GREATER` [S3, unofficial].
        *   Keep Alpha Test enabled (`GL_GREATER`, ref $> 0$) [S3, unofficial].
        *   Render the volume slabs from back to front (or front to back, depending on implementation, but the logic captures the farthest visible fragment due to `GL_GREATER` and initialization) [S3, unofficial].
        *   The depth buffer now contains the distances of the farthest significant fragments [S3, unofficial].

5.  **Coordinate Transformation:**
    *   The depth buffer stores values in Normalized Device Coordinates (NDC), range $[0, 1]$ [S3, unofficial].
    *   Transform these NDC coordinates to world coordinates to get the ray origin $P$ and ray end $Q$ [S3, unofficial].
    *   For a pixel with screen coordinates $(x_s, y_s, z_s)$, the world coordinates $(x, y, z, w)$ are calculated as:
        $$
        \begin{bmatrix} x \\ y \\ z \\ w \end{bmatrix} = (P M)^{-1} V \begin{bmatrix} x_s \\ y_s \\ z_s \\ 1 \end{bmatrix}
        $$
        where $P$ is the projection matrix, $M$ is the model matrix, and $V$ is the view matrix [S3, unofficial].
    *   The near point $P_{world}$ corresponds to the depth from Pass 1, and the far point $Q_{world}$ corresponds to the depth from Pass 2 [S3, unofficial].

6.  **Ray Casting with Empty Space Skipping:**
    *   For each pixel, cast a ray between $P$ and $Q$ [S3, unofficial].
    *   Sample the volume only within the interval $[P, Q]$ [S3, unofficial].
    *   Use parametric equations for $t \in [0, 1]$:
        $$
        \begin{aligned}
        X(t) &= P_x + (Q_x - P_x)t \\
        Y(t) &= P_y + (Q_y - P_y)t \\
        Z(t) &= P_z + (Q_z - P_z)t
        \end{aligned}
        $$
        The increment in $t$ is determined by the number of samples $n$ per ray (e.g., $\Delta t = 1/n$) [S3, unofficial].
    *   Composite the samples using the standard volume rendering integral, but restricted to the active segment:
        $$
        L(x) = \int_{x_A'}^{x_B'} \left[ e^{-\int_{x_A'}^{x_00} \phi_t(x')dx'} \epsilon(x_00) dx_00 \right]
        $$
        where $x_A'$ and $x_B'$ are the near and far points, $\phi_t$ is the extinction coefficient, and $\epsilon$ is the color [S3, unofficial].
    *   Draw the final color to the framebuffer [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_with_depth_alpha_skip(volume, transfer_function, camera):
    """
    Renders a volume using Westermann & Sevenich's empty space skipping technique.
    """
    # 1. Pre-compute gradients and illumination on CPU
    gradients = compute_central_difference_gradients(volume)
    illuminated_volume = apply_lighting_model(volume, gradients, transfer_function)
    
    # 2. Prepare 2D texture slabs
    slabs = create_texture_slabs(illuminated_volume)
    
    # 3. Determine rendering axis and order
    axis = get_axis_with_smallest_angle_to_projection_plane(camera)
    slab_order = get_sorted_slabs(slabs, axis, camera)
    
    # 4. Pass 1: Find Near Points (SSM)
    gl_clear_depth(1.0)
    gl_depth_func(GL_LESS)
    gl_enable(GL_ALPHA_TEST)
    gl_alpha_func(GL_GREATER, 0.001) # Discard zero opacity
    gl_depth_mask(GL_TRUE)
    
    for slab in slab_order: # Front to back
        gl_bind_texture(slab)
        gl_draw_slab(slab)
        
    near_depth_buffer = gl_read_depth_buffer()
    
    # 5. Pass 2: Find Far Points (SSM)
    gl_clear_depth(0.0)
    gl_depth_func(GL_GREATER)
    # Alpha test remains enabled
    
    for slab in reversed(slab_order): # Back to front
        gl_bind_texture(slab)
        gl_draw_slab(slab)
        
    far_depth_buffer = gl_read_depth_buffer()
    
    # 6. Ray Casting
    image_width, image_height = get_framebuffer_size()
    projection_matrix = camera.projection_matrix
    model_matrix = camera.model_matrix
    view_matrix = camera.view_matrix
    inv_pm = inverse(projection_matrix * model_matrix)
    
    for y in range(image_height):
        for x in range(image_width):
            # Get NDC depths
            z_near_ndc = near_depth_buffer[y, x]
            z_far_ndc = far_depth_buffer[y, x]
            
            # Skip if no volume intersection (e.g., if depth is default or invalid)
            if z_near_ndc == 1.0 or z_far_ndc == 0.0:
                continue
                
            # Transform NDC to World Coordinates
            # NDC x,y are in [-1, 1], z in [0, 1]
            # Assuming standard OpenGL viewport mapping
            ndc_x = (2.0 * x / image_width) - 1.0
            ndc_y = (2.0 * y / image_height) - 1.0
            
            # Reconstruct world coordinates for near point
            near_world = transform_ndc_to_world(ndc_x, ndc_y, z_near_ndc, inv_pm, view_matrix)
            # Reconstruct world coordinates for far point
            far_world = transform_ndc_to_world(ndc_x, ndc_y, z_far_ndc, inv_pm, view_matrix)
            
            # Ray Casting between near_world and far_world
            color = composite_ray(near_world, far_world, volume, transfer_function)
            
            set_framebuffer_pixel(x, y, color)

def transform_ndc_to_world(ndc_x, ndc_y, ndc_z, inv_pm, view_matrix):
    """
    Transforms NDC coordinates to world coordinates.
    Note: The exact transformation depends on the specific OpenGL pipeline state.
    This is a simplified representation of the matrix math described in [S3, unofficial].
    """
    # Homogeneous NDC coordinate
    ndc_coord = [ndc_x, ndc_y, ndc_z, 1.0]
    
    # Apply inverse projection and model view to get view space, then inverse view to get world
    # The source formula: [x,y,z,w] = (PM)^-1 V [xs, ys, zs, 1]
    # Here xs,ys,zs are assumed to be the NDC coords if V is identity or handled separately.
    # A more robust implementation would unproject from NDC to World directly.
    
    # Simplified: Unproject from NDC to World
    # This step requires careful handling of the specific coordinate system conventions.
    # For the purpose of this article, we assume a helper function exists that performs
    # the inverse projection and view transformations correctly.
    return unproject(ndc_x, ndc_y, ndc_z, inv_pm, view_matrix)

def composite_ray(origin, end, volume, transfer_function):
    """
    Standard front-to-back compositing for the ray segment [origin, end].
    """
    alpha = 0.0
    color = [0.0, 0.0, 0.0]
    
    # Calculate step size
    ray_length = distance(origin, end)
    num_samples = 100 # Or based on resolution
    step = ray_length / num_samples
    
    current_pos = origin
    
    for i in range(num_samples):
        # Sample volume at current_pos
        scalar = sample_volume(current_pos, volume)
        rgba = transfer_function(scalar)
        
        # Front-to-back compositing
        alpha += (1.0 - alpha) * rgba.alpha
        color += rgba.color * rgba.alpha * (1.0 - (alpha - rgba.alpha)) # Simplified approximation
        
        if alpha >= 0.99: # Early ray termination
            break
            
        current_pos += normalize(end - origin) * step
        
    return color
```

### Parameters and Typical Values
*   **Alpha Test Reference Value:** $> 0$ (e.g., 0.001) to discard fully transparent fragments [S3, unofficial].
*   **Depth Buffer Initialization:** 1.0 for near point pass, 0.0 for far point pass [S3, unofficial].
*   **Number of Samples ($n$):** Determined by desired image quality and ray length. Typical values range from 50 to 200 per ray segment [S3, unofficial].

### Edge Cases and Pitfalls
*   **Coordinate System Mismatch:** The depth buffer uses NDC $[0, 1]$, while ray casting requires local/world coordinates. A transformation is necessary [S3, unofficial].
*   **High Data Dispersion:** If the volume has high dispersion (many small isolated structures), the technique only skips external empty space. The ray length with skipping may be similar to the naive ray length, reducing effectiveness [S3, unofficial].
*   **Precision:** Floating-point precision in the depth buffer can affect the accuracy of the near/far points, especially for thin structures [G].

## Subtopics

### [Polygon Assisted Ray Casting](2930-polygon-assisted-ray-casting.md)
Uses external polygonal meshes instead of alpha/depth tests to define sampling intervals.

### [Impostor Depth Optimization](4434-impostor-depth-optimization.md)
Uses camera-facing quads and depth tests to cull occluded primitives before ray marching.

## Operating characteristics
*   **Time Complexity:** The acceleration structure generation (SSM) is very small compared to total rendering time [S1, unofficial]. The ray casting time is reduced to almost a quarter of a naive ray caster [S3, unofficial].
*   **Measured Performance:** 3 to 4 times faster than naive ray casting when using alpha and depth tests [S3, unofficial].
*   **Memory Complexity:** Requires storage for the volume, gradients, and 2D texture slabs. No additional hierarchical data structure (like an octree) is needed on the CPU [S1, unofficial].
*   **Strengths:** No CPU-side preprocessing required; leverages hardware z-occlusion culling; simple implementation using standard OpenGL features [S1, unofficial][S3, unofficial].
*   **Limitations:** Lighting is pre-computed on the CPU, making dynamic lighting difficult [S3, unofficial]. Effectiveness degrades with high data dispersion [S3, unofficial]. Limited to skipping external empty space; internal holes are not skipped [S3, unofficial].

## Background
The method was introduced by Westermann & Sevenich in [WS01] [S3, unofficial]. It is a hybrid volume rendering technique that uses 2D textures and OpenGL alpha/depth tests to discard non-contributing intervals of the volume rendering integral [S3, unofficial]. It differs from Engel, Kraus & Ertl's method by performing illumination calculation on the CPU in the initial phase, whereas Engel's method does it on the GPU [S3, unofficial].

## Sources
[S1] Acceleration of direct volume rendering with texture slabs on programmable graphics hardware, https://core.ac.uk/works/315972218, fetched 2026-10-26 (unofficial)
[S2] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
[S3] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
