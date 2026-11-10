# Ray Casting with Depth/Alpha Test for Empty Space Skipping

Also called: Empty Space Skipping via OpenGL Tests, Ray Casting using Alpha and Depth Tests, Texture Slab Volume Rendering with GPU-based Acceleration, Westermann & Sevenich Empty Space Skipping, Westermann & Sevenich Technique

## Overview
This method accelerates direct volume rendering by using OpenGL’s alpha test and depth test to determine the entry and exit points of rays intersecting the volume, allowing samples to be taken only between these points [S3, unofficial]. It solves the problem of wasted computation on transparent regions by skipping empty space intervals in the volume integral [S3, unofficial]. Unlike its parent method, which relies on CPU-side or generic GPU-side acceleration structures, this technique builds the acceleration data structure (Slab Silhouette Maps) on the fly using the depth buffer without CPU-side preprocessing [S1, unofficial]. It differs from [Polygon Assisted Ray Casting](2930-polygon-assisted-ray-casting.md) by using the volume's own alpha/depth tests rather than an external polygonal mesh, and from [Space Leaping](5863-space-leaping.md) by utilizing hardware z-occlusion culling via texture slabs rather than hierarchical grid traversal [S1, unofficial][S3, unofficial].

## How it works
The method operates in three phases: pre-processing (illumination and texture creation), near/far point determination (using OpenGL tests), and ray casting (sampling between determined points).

**Inputs:**
- Volume dataset (scalar field).
- Camera parameters (Projection matrix $P$, Modelview matrix $V$).
- Rendering resolution (width $W$, height $H$).

**Outputs:**
- Final color image (framebuffer).

**Data Structures:**
1.  **Volume Data:** Stored in main memory. Gradients are pre-calculated using central differences [S3, unofficial].
2.  **Illumination Texture:** RGB texture storing pre-computed lighting/shading values for each voxel, generated on the CPU [S3, unofficial].
3.  **Texture Slabs:** Groups of 2D texture slices. For each axis (X, Y, Z), 2D textures are created for planes parallel to that axis [S3, unofficial].
4.  **Display Lists:** Pre-compiled OpenGL lists for rendering slabs in front-to-back and back-to-front order [S3, unofficial].
5.  **Depth Buffer:** Used to store near and far intersection distances in Normalized Device Coordinates (NDC) [S3, unofficial].

**Parameters:**
-   **Alpha Test Reference:** A value $> 0$ (typically small epsilon) to discard fully transparent fragments [S3, unofficial].
-   **Number of Samples ($n$):** Determines the step size $\Delta t = 1/n$ along the ray [S3, unofficial].

**Steps:**

1.  **Pre-processing (CPU):**
    -   Calculate the gradient for each volume element using central differences [S3, unofficial].
    -   Perform illumination and shading based on the lighting model and store the result in main memory [S3, unofficial].
    -   Create 2D textures for each plane parallel to the X, Y, and Z axes [S3, unofficial].
    -   Load textures into GPU memory and create display lists for front-to-back and back-to-front rendering orders [S3, unofficial].

2.  **Determine Near and Far Points (GPU Passes):**
    -   **Pass 1 (Near Point):**
        -   Initialize the depth buffer to $1.0$ (maximum NDC depth).
        -   Set depth write criterion to `GL_LESS` [S3, unofficial].
        -   Render the volume slabs. The alpha test is enabled with `GL_GREATER` and reference value $> 0$ to discard fragments with zero opacity [S3, unofficial].
        -   This pass captures the distances of the closest significant fragments to the projection plane in the depth buffer [S3, unofficial].
    -   **Pass 2 (Far Point):**
        -   Initialize the depth buffer to $0.0$ (minimum NDC depth).
        -   Set depth write criterion to `GL_GREATER` [S3, unofficial].
        -   Render the volume slabs again with the same alpha test settings.
        -   This pass captures the distances of the farthest significant fragments from the projection plane [S3, unofficial].

3.  **Coordinate Transformation:**
    -   Read the depth buffer values $(X_s, Y_s, Z_s)$ for each pixel. These are in NDC.
    -   Transform these coordinates to world coordinates using the inverse of the projection and modelview matrices [S3, unofficial]:
        $$
        \begin{bmatrix} x \\ y \\ z \\ w \end{bmatrix} = (P M)^{-1} V \begin{bmatrix} X_s \\ Y_s \\ Z_s \\ 1 \end{bmatrix}
        $$
        where $P$ is the projection matrix, $M$ is the model matrix (part of Modelview), and $V$ is the view transformation [S3, unofficial].
    -   Define the near point $P = (P_x, P_y, P_z)$ and far point $Q = (Q_x, Q_y, Q_z)$ in world coordinates [S3, unofficial].

4.  **Ray Casting and Compositing:**
    -   For each pixel, cast a ray from $P$ to $Q$.
    -   Sample the volume along the ray using parametric equations for $t \in [0, 1]$ with increment $\Delta t = 1/n$ [S3, unofficial]:
        $$
        \begin{aligned}
        X(t) &= P_x + (Q_x - P_x)t \\
        Y(t) &= P_y + (Q_y - P_y)t \\
        Z(t) &= P_z + (Q_z - P_z)t
        \end{aligned}
        $$
    -   Retrieve color and opacity from the pre-computed illumination texture at the sampled positions.
    -   Composite the samples using front-to-back blending.
    -   Draw the final color to the framebuffer [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume(volume_data, camera_params):
    # Step 1: Pre-processing (CPU)
    gradients = calculate_central_differences(volume_data)
    illumination_texture = compute_lighting(volume_data, gradients)
    slab_textures = create_2d_textures_from_volume(volume_data)
    display_lists = create_display_lists(slab_textures, order='front-to-back')
    
    # Step 2: Determine Near and Far Points
    # Pass 1: Find Near Point
    gl.clear_depth(1.0)
    gl.enable(gl.DEPTH_TEST)
    gl.depth_func(gl.LESS)
    gl.depth_mask(gl.TRUE)
    gl.enable(gl.ALPHA_TEST)
    gl.alpha_func(gl.GREATER, 0.001) # Reference > 0
    
    render_slabs(display_lists['front-to-back'])
    near_depth_buffer = gl.read_depth_buffer()
    
    # Pass 2: Find Far Point
    gl.clear_depth(0.0)
    gl.depth_func(gl.GREATER)
    gl.depth_mask(gl.TRUE)
    
    render_slabs(display_lists['front-to-back'])
    far_depth_buffer = gl.read_depth_buffer()
    
    # Step 3: Coordinate Transformation
    P_matrix = camera_params.projection
    M_matrix = camera_params.model
    V_matrix = camera_params.view
    inv_PM = inverse(P_matrix * M_matrix)
    
    near_points_world = []
    far_points_world = []
    
    for y in range(height):
        for x in range(width):
            zs_near = near_depth_buffer[y, x]
            zs_far = far_depth_buffer[y, x]
            
            # Convert NDC to World Coordinates
            # Homogeneous coordinates for NDC: (x/w, y/w, zs, 1)
            # Assuming perspective projection, x_ndc = x / width * 2 - 1, etc.
            # Simplified: assume normalized screen coords xs, ys in [-1, 1]
            xs = (x / width) * 2 - 1
            ys = (y / height) * 2 - 1
            
            ndc_near = [xs, ys, zs_near, 1.0]
            ndc_far = [xs, ys, zs_far, 1.0]
            
            world_near = transform(inv_PM, ndc_near)
            world_far = transform(inv_PM, ndc_far)
            
            # Normalize w
            P = [c / world_near[3] for c in world_near[:3]]
            Q = [c / world_far[3] for c in world_far[:3]]
            
            near_points_world.append(P)
            far_points_world.append(Q)
            
    # Step 4: Ray Casting
    final_image = zeros((height, width, 3))
    n_samples = 100 # Example parameter
    
    for y in range(height):
        for x in range(width):
            P = near_points_world[y * width + x]
            Q = far_points_world[y * width + x]
            
            # If P or Q are invalid (e.g., no intersection), skip
            if is_invalid(P) or is_invalid(Q):
                continue
                
            color = [0.0, 0.0, 0.0]
            alpha = 0.0
            
            dt = 1.0 / n_samples
            t = 0.0
            
            while t <= 1.0:
                # Ray position
                rx = P[0] + (Q[0] - P[0]) * t
                ry = P[1] + (Q[1] - P[1]) * t
                rz = P[2] + (Q[2] - P[2]) * t
                
                # Sample illumination texture
                # Note: Texture coordinates must be mapped from world to texture space
                tex_color, tex_alpha = sample_illumination_texture(rx, ry, rz)
                
                # Front-to-back compositing
                # C_out = C_in + (1 - alpha_in) * C_sample * alpha_sample
                # A_out = A_in + (1 - A_in) * alpha_sample
                
                contribution = (1.0 - alpha) * tex_alpha
                color[0] += contribution * tex_color[0]
                color[1] += contribution * tex_color[1]
                color[2] += contribution * tex_color[2]
                alpha += contribution
                
                if alpha >= 1.0:
                    break
                    
                t += dt
                
            final_image[y, x] = color
            
    return final_image
```

## Subtopics

### [Polygon Assisted Ray Casting](2930-polygon-assisted-ray-casting.md)
Uses an external polygonal mesh to define valid sampling intervals, reducing primary ray computation by a factor of four.

### [Impostor Depth Optimization](4434-impostor-depth-optimization.md)
Renders volumetric primitives as 2D quads to leverage depth testing for culling occluded objects, avoiding fragment shader execution for hidden primitives.

## Operating characteristics
-   **Time Complexity:** The pre-processing phase (gradient and lighting calculation) is $O(V)$ where $V$ is the number of voxels, performed once or when data changes. The rendering phase is $O(N \cdot L)$ where $N$ is the number of pixels and $L$ is the average length of the active ray segment (samples between near and far points).
-   **Memory Complexity:** Requires storage for the volume data, pre-computed illumination texture ($O(V)$), and slab textures ($O(V)$). The depth buffer requires $O(W \cdot H)$ [S3, unofficial].
-   **Measured Performance:** Reduces ray casting time to almost a quarter of the time taken by a naive ray caster that samples along the entire ray extension [S3, unofficial]. Performance improvements of 3 to 4 times faster are observed when using alpha and depth tests compared to not using them [S3, unofficial].
-   **Strengths:** No CPU-side preprocessing for the acceleration structure; relies on hardware z-occlusion culling; reduces framebuffer write operations by discarding transparent fragments [S1, unofficial][S3, unofficial].
-   **Limitations:** Illumination is pre-computed on the CPU and static, preventing dynamic lighting changes during rendering [S3, unofficial]. High data dispersion reduces effectiveness because the technique only skips external empty spaces; if dispersion is high, the ray length with skipping is similar to the length without optimization [S3, unofficial].
-   **When to Use:** Suitable for static volumes with significant empty space around the object, where dynamic lighting is not required.
-   **When Not to Use:** Unsuitable for dynamic lighting scenarios or highly dispersed volumes where the skipping benefit is minimal [S3, unofficial].

## Background
The technique was introduced by Westermann & Sevenich in work [WS01] [S3, unofficial]. It is a hybrid volume rendering technique that uses the GPU's depth buffer to skip empty spaces outside the volume, allowing ray casting to sample only non-transparent material [S3, unofficial]. Unlike Engel, Kraus & Ertl, this technique performs illumination calculation uniquely and in the initial phase on the CPU, whereas Engel's method does it on the GPU [S3, unofficial]. The method groups texture slices together to form texture slabs and creates Slab Silhouette Maps (SSM) from the alpha component of the slab to identify empty spaces and terminated rays [S1, unofficial].

## Sources
[S1] Acceleration of direct volume rendering with texture slabs on programmable graphics hardware, https://core.ac.uk/works/315972218, fetched 2026-10-26 (unofficial)
[S2] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
[S3] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
