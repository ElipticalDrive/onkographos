# Two-Pass Ray Entry/Exit Determination

Also called: Krüger and Westermann approach, Object-order ray setup, Rasterize bounding box, Rasterized Bounding Box Ray Setup, Ray Setup Pass, Ray casting setup rendering, Two-Pass Rasterization Intersection, Two-Pass Raycasting with Back-Face Depth Buffering

## Overview
This method determines the 3D entry and exit points of view rays intersecting a volume’s bounding box by rasterizing the box geometry to textures, thereby avoiding procedural intersection calculations in the fragment shader [S1, unofficial][S3]. It sets apart from analytical ray-box intersection tests by leveraging the GPU's parallel rasterization pipeline to generate ray setup data (origins and directions) as a byproduct of rendering a simple color cube [S3]. Unlike the parent method which uses geometry shaders to output hit positions, this approach relies on standard vertex/fragment pipelines with face culling to separate front and back intersections [S2, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A unit cube (or scaled bounding box) representing the volume boundaries; the concatenated World-View-Projection matrix ($M_{WVP}$); the camera position in world space ($P_{cam}$); and the volume texture dimensions [S2, S4, unofficial].
*   **Outputs:** Two textures storing the 3D texture coordinates of the ray entry point and the ray exit point for every pixel [S4, unofficial]. From these, the ray direction is derived [S3].

**Data Structures:**
1.  **Bounding Box Geometry:** A cube with 8 vertices. In model space, vertices are typically in the range $[-0.5, 0.5]^3$ or $[0, 1]^3$ [S2, S4, unofficial]. Each vertex is assigned a color equal to its 3D texture coordinate (e.g., vertex at $(0,0,0)$ has color $(0,0,0)$, vertex at $(1,1,1)$ has color $(1,1,1)$) [S4, unofficial].
2.  **Ray Setup Textures:** Two 2D textures (Resolution: $W \times H$, matching the screen) to store the 3D coordinates [S2, S4, unofficial].
3.  **Constant Buffer:** Contains the $M_{WVP}$ matrix [S2, unofficial].

**Numbered Steps:**
1.  **Prepare Geometry:** Create a cube mesh. Assign vertex colors corresponding to the volume's texture coordinate space. If the volume texture coordinates are $[0,1]^3$, map the cube's corners accordingly: $(0,0,0)$ is black, $(1,0,0)$ is red, $(0,1,0)$ is green, $(0,0,1)$ is blue, etc. [S4, unofficial].
2.  **Pass 1: Determine Exit Point (Back Faces):** Enable face culling to cull front faces (`GL_FRONT` or equivalent). Render the cube to the "Exit Point" texture. The fragment shader outputs the interpolated vertex color (which represents the 3D texture coordinate of the intersection) as RGB, with alpha set to 1.0 [S2, S4, unofficial]. Because front faces are culled, the depth buffer retains the farthest intersection (the exit point) for each ray [S3][S4, unofficial].
3.  **Pass 2: Determine Entry Point (Front Faces) and Compute Direction:** Disable face culling or enable culling to cull back faces (`GL_BACK`). Render the cube to the "Entry Point" texture. The fragment shader outputs the interpolated vertex color (entry point texture coordinate) as RGB [S4, unofficial]. In some implementations, Pass 2 reads the Exit Point texture from Pass 1. The fragment shader computes the direction vector $D = P_{exit} - P_{entry}$ and stores the direction in a separate texture or computes it on-the-fly during the main raycasting pass [S1, S4, unofficial].
4.  **Compute Ray Direction:** For each pixel, retrieve the entry point texture coordinate $T_{entry}$ and exit point texture coordinate $T_{exit}$ from the setup textures. The ray direction in texture space is $D_{tex} = T_{exit} - T_{entry}$ [S1, unofficial][S3]. Note that $T_{entry}$ and $T_{exit}$ are in normalized texture coordinates $[0,1]^3$ [S4, unofficial].
5.  **Main Raycasting Pass:** Use the entry point and direction to traverse the volume. Accumulate color and opacity along the ray using the volume rendering equation [S1, unofficial][S3].

**Formulas:**
*   **Vertex Shader Transformation:**
    $$Pos_{clip} = M_{WVP} \times Pos_{model}$$
    Where $Pos_{model}$ is the vertex position in model space [S2, unofficial].
*   **Texture Coordinate Mapping (if model space is $[-0.5, 0.5]$):**
    $$T_{xyz} = Pos_{model}.xyz + (0.5, 0.5, 0.5)$$
    This shifts coordinates to $[0.0, 1.0]$ [S2, unofficial].
*   **Ray Direction:**
    $$D = P_{exit} - P_{entry}$$
    Where $P_{exit}$ and $P_{entry}$ are the 3D texture coordinates retrieved from the setup textures [S3].
*   **Volume Rendering Equation (Discrete):**
    $$C \approx \sum_{i=0}^{n-1} \left( (1 - \alpha_i) \sum_{j=0}^{i-1} (1 - \alpha_j) \alpha_i c_i \right)$$
    Where $\alpha_i$ is opacity, $c_i$ is color, and $n$ is the number of samples [S3].

**Parameters:**
*   **Matrix $M_{WVP}$:** Concatenated World-View-Projection matrix. Must be updated every frame to reflect camera movement [S2, unofficial].
*   **Culling Mode:** `GL_FRONT` for exit point pass, `GL_BACK` or none for entry point pass [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Buffer:** The method relies on the depth buffer to resolve front/back intersections. Ensure the depth buffer is cleared and enabled correctly between passes [S4, unofficial].
*   **Orthographic vs. Perspective:** The two-pass method works for both. In orthographic projection, the entry and exit points are simply the front and back faces of the box along the view direction. In perspective, they are the intersections with the near and far clipping planes of the box [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_ray_entry_exit(screen_width, screen_height, volume_bbox, m_wvp):
    # Pass 1: Render back faces to get exit point texture coordinates
    exit_texture = create_texture(screen_width, screen_height, format=RGB32F)
    bind_fbo(exit_texture)
    clear_depth_buffer()
    enable_culling()
    set_cull_face(FRONT) # Cull front faces, render back faces
    render_cube(m_wvp) # Vertices colored by texture coordinates in [0,1]^3
    exit_tex = read_fbo()
    
    # Pass 2: Render front faces to get entry point texture coordinates
    entry_texture = create_texture(screen_width, screen_height, format=RGB32F)
    bind_fbo(entry_texture)
    clear_depth_buffer()
    enable_culling()
    set_cull_face(BACK) # Cull back faces, render front faces
    render_cube(m_wvp)
    entry_tex = read_fbo()
    
    return entry_tex, exit_tex

def compute_ray_direction(entry_tex, exit_tex, x, y):
    t_entry = sample_texture(entry_tex, x, y) # RGB vector in [0,1]^3
    t_exit = sample_texture(exit_tex, x, y)   # RGB vector in [0,1]^3
    return t_exit - t_entry
```

## Subtopics

### Procedural Ray-Box Intersection
Computes intersections analytically in the fragment shader. Simpler to implement but more computationally expensive per pixel. Choose when memory bandwidth for setup textures is a bottleneck or when the volume shape is not a simple box [S1, unofficial][S3].

### Geometry Shader Ray Setup
Uses geometry shaders to output ray data directly from primitive assembly. Avoids the need for two rasterization passes but requires hardware support for geometry shaders and may have higher overhead for simple boxes. Choose when using modern OpenGL/DirectX and avoiding FBO binds is desired [Parent].

## Operating characteristics
*   **Time Complexity:** $O(W \times H)$ for the setup passes (rasterizing a constant-size cube) plus $O(W \times H \times N)$ for the main raycasting pass, where $N$ is the number of samples per ray. The setup cost is negligible compared to the traversal.
*   **Measured Performance:** The rasterization setup is extremely fast, leveraging the GPU's fixed-function rasterizer. It avoids the overhead of branch-heavy procedural intersection tests in the fragment shader [S1, unofficial][S3].
*   **Strengths:**
    *   Simple to implement on older GPUs that lack advanced shader features [S1, unofficial].
    *   Efficient use of parallel rasterization hardware [S3].
    *   Independent of projection type (orthographic or perspective) [S1, unofficial].
*   **Limitations:**
    *   Requires two additional render passes and FBO binds [S4, unofficial].
    *   Only works for convex, box-like volumes [S3].
*   **When to use:** When the volume is axis-aligned or can be tightly bounded by a box, and when procedural intersection is too slow or complex. Ideal for real-time raycasting on GPUs with limited shader complexity [S1, unofficial][S3].
*   **When not to use:** For non-convex volumes, or when memory bandwidth is extremely constrained and FBO binds are costly. Also less suitable if the volume is not box-shaped [S3].

## Background
The method was introduced by Krüger and Westermann in 2003 as an efficient way to perform volume raycasting on GPUs by leveraging rasterization [S1, unofficial][S3]. It was developed as an alternative to procedural ray-box intersection, which placed a heavy load on fragment shaders [S1, unofficial]. The technique has been widely adopted in GPU-based volume rendering pipelines, particularly in early implementations of real-time raycasting [S1, unofficial][S3].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] BernhardKidalka/D3DVolumeRaycaster, https://github.com/BernhardKidalka/D3DVolumeRaycaster, fetched 2026-10-26 (unofficial)
[S3] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S4] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
