# Two-Pass GPU Approach

Also called: BT Volume Rendering Pipeline, GPU Ray Casting for Tetrahedral Meshes, GPU-Accelerated Cell Projection (Two-Pass), Tetrahedral Cell Projection, Weiler tetrahedral ray caster

## Overview
The Two-Pass GPU Approach is an efficient ray-casting algorithm for tetrahedral meshes implemented on consumer graphics hardware using fragment programs, pre-integration, and multiple rendering passes [S1, S4, unofficial]. It projects tetrahedra into screen space, classifies their geometry, and renders them as triangle fans with interpolated colors and opacities, supporting both volume ray integral and isosurface rendering [S4, unofficial]. It sets itself apart from its parent [Projected Tetrahedra](877-projected-tetrahedra.md) by offloading the heavy projection, classification, and sorting logic to the GPU via a two-pass architecture, thereby reducing CPU–GPU data transfer and enabling interactive frame rates for mid-sized models [S3][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh (scalar field and gradient data), a transfer function (mapping scalars to color/opacity), and an optional isosurface level [S1, S4, unofficial].
*   **Output:** A 2D image where each pixel contains the accumulated color and opacity of the ray passing through the tetrahedra [S4, unofficial].

**Data Structures:**
The method relies on four GPU textures to store pre-computed tetrahedron data [S3][S4, unofficial]:
1.  **Tetrahedral Texture:** A 32-bit RGBA texture. Each texel corresponds to one tetrahedron and contains four 32-bit indices pointing to the vertex data in the Vertex and Gradient Textures [S3][S4, unofficial].
2.  **Vertex Texture:** A 32-bit RGBA texture. Each texel stores the 3D coordinates (RGB) and scalar value (A) of a single vertex [S4, unofficial].
3.  **Gradient Texture:** A 32-bit RGB texture (3 components per texel). Each texel stores the gradient vector (normal) of a vertex [S3][S4, unofficial].
4.  **Classification Texture:** An 8-bit texture storing a Ternary Truth Table. It maps the combination of four cross-product test results to a projection class (1–4) and vertex ordering [S4, unofficial].

**Numbered Steps:**

1.  **Boundary Rasterization (Ray Entry):**
    The first tetrahedron along each ray is identified by rasterizing the extracted boundary faces of the model [S1, unofficial]. This determines the entry point for ray traversal.

2.  **Geometry Shader Pre-Pass (Culling):**
    A geometry shader pre-pass transforms tetrahedra into screen space and culls those that do not contribute to the current isosurface level or transfer function settings. Vertices passing this test are streamed into a second buffer [S2][S4, unofficial].

3.  **Pass 1: Projection and Classification (Fragment Shader):**
    For each tetrahedron, the fragment shader computes the following:
    *   **Screen-Space Projection:** Project the four vertices $v_i$ to screen space $v_{proj,i}$.
    *   **Cross-Product Tests:** Compute four tests to determine the projection class and handle degeneracy [S4, unofficial]:
        $$
        \begin{aligned}
        \text{vec}_1 &= v_{proj,1} - v_{proj,0} \\
        \text{vec}_2 &= v_{proj,2} - v_{proj,0} \\
        \text{vec}_3 &= v_{proj,3} - v_{proj,0} \\
        \text{vec}_4 &= v_{proj,1} - v_{proj,2} \\
        \text{vec}_5 &= v_{proj,1} - v_{proj,3}
        \end{aligned}
        $$
        $$
        \begin{aligned}
        t_1 &= \text{sign}((\text{vec}_1 \times \text{vec}_2).z) + 1 \\
        t_2 &= \text{sign}((\text{vec}_1 \times \text{vec}_3).z) + 1 \\
        t_3 &= \text{sign}((\text{vec}_2 \times \text{vec}_3).z) + 1 \\
        t_4 &= \text{sign}((\text{vec}_4 \times \text{vec}_5).z) + 1
        \end{aligned}
        $$
        Each $t_i \in \{0, 1, 2\}$. The combination of $(t_1, t_2, t_3, t_4)$ indexes the Classification Texture to determine the class and vertex order [S4, unofficial].
    *   **Degeneracy Handling:**
        *   If one test result is 1: Class 3, project as two triangles.
        *   If two test results are 1: Class 4, project as one triangle.
        *   If three or four tests yield 1: Discard the tetrahedron as degenerate [S4, unofficial].
    *   **Scalar and Gradient Computation:**
        *   Compute front scalar $s_f$ and back scalar $s_b$ via ray-plane intersection [S1, unofficial].
        *   Compute front gradient $g_f$ and back gradient $g_b$ by interpolating vertex gradients [S4, unofficial].
        *   Compute cell thickness $l$ and centroid z-coordinate $c_z$ [S4, unofficial].
    *   **Output to CPU (MRT):**
        Data is sent back to the CPU via Multiple Render Targets (MRT) with four 2D RGBA frame buffer attachments (32 bits/component) [S4, unofficial]:
        *   **Attachment 1:** Intersection vertex coordinates $(x_{vi}, y_{vi})$ (for Class 2), centroid $c_z$, and Ternary Truth Table index $id_{TTT}$.
        *   **Attachment 2:** Scalars $s_f, s_b$, thickness $l$, and triangle count.
        *   **Attachment 3:** Front gradient $g_f$.
        *   **Attachment 4:** Back gradient $g_b$.

4.  **Pass 2: Preparation (CPU/GPU Sync):**
    *   Retrieve data from MRT textures.
    *   Sort tetrahedra (bucket sort for rotation, merge-sort for static) [S4, unofficial].
    *   Create an indices array (6 integers per tetrahedron defining fan order) and a count array (vertices per fan). Max vertices per fan is 6 [S4, unofficial].

5.  **Pass 2: Rendering (Fragment Shader):**
    *   Draw primitives as triangle fans using `glMultiDrawElements` [S4, unofficial].
    *   Interpolate thick and thin vertex colors/opacities across fragments.
    *   **Opacity Calculation:**
        $$ \alpha = 1 - e^{-\tau_{avg} \cdot l} $$
        where $\tau_{avg}$ is the average of front and back extinction coefficients $\tau(s_f)$ and $\tau(s_b)$, and $l$ is cell thickness [S4, unofficial].
    *   **Color Compositing:**
        $$ I_{new} = \alpha C + (1 - \alpha) I $$
        where $I$ is the previous framebuffer color, and $C$ and $\alpha$ are the interpolated color and opacity [S4, unofficial].
    *   **Isosurface Detection:**
        If the isovalue lies between the interpolated front and back scalars, render with Phong shading using the interpolated gradient as the normal vector [S4, unofficial]. Otherwise, apply the volume ray integral [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard math helpers
def dot(a, b): return a.x*b.x + a.y*b.y + a.z*b.z
def cross(a, b): return vec3(a.y*b.z - a.z*b.y, a.z*b.x - a.x*b.z, a.x*b.y - a.y*b.x)
def length(a): return sqrt(dot(a, a))
def normalize(a): return a / length(a)
def sign(x): return 1 if x > 0 else (-1 if x < 0 else 0)
def max(a, b): return a if a > b else b
def min(a, b): return a if a < b else b
def clamp(v, lo, hi): return max(lo, min(hi, v))

# Texture sampling helpers
def sample_1d_texture(tex, coord):
    idx = int(clamp(coord, 0.0, 1.0) * (len(tex) - 1))
    return tex[idx]

def sample_2d_texture(tex, uv):
    return tex[int(uv.y * (len(tex)-1))][int(uv.x * (len(tex[0])-1))]

# Geometric helpers
def project_to_screen(v):
    # Applies view-projection matrix to vertex v
    return v # Placeholder for standard projection math

def centroid(verts):
    return (verts[0] + verts[1] + verts[2] + verts[3]) / 4.0

def ray_plane_intersection(ray_origin, ray_dir, plane_point, plane_normal):
    denom = dot(ray_dir, plane_normal)
    if abs(denom) < 1e-6: return None
    t = dot(plane_point - ray_origin, plane_normal) / denom
    return ray_origin + t * ray_dir

def compute_ray_tet_intersection(ray_origin, ray_dir, verts):
    # Simplified: finds entry and exit points by intersecting ray with tetrahedron faces
    # Returns (entry_point, exit_point)
    # In practice, uses slab method or face intersections [Garrity90]
    return ray_origin, ray_origin + ray_dir # Placeholder for actual intersection logic

def interpolate_scalar(point, verts):
    # Barycentric interpolation of scalar values
    return verts[0].s # Placeholder for actual barycentric math

def interpolate_gradient(point, grads):
    # Barycentric interpolation of gradient vectors
    return grads[0] # Placeholder for actual barycentric math

# Pass 1: Fragment Shader (Per Tetrahedron)
def pass1_fragment_shader(tet_index, ray_origin, ray_dir):
    v_indices = TetrahedralTexture[tet_index]
    verts = [VertexTexture[idx] for idx in v_indices]
    grads = [GradientTexture[idx] for idx in v_indices]
    
    v_proj = [project_to_screen(v.xyz) for v in verts]
    
    vec1 = v_proj[1] - v_proj[0]
    vec2 = v_proj[2] - v_proj[0]
    vec3 = v_proj[3] - v_proj[0]
    vec4 = v_proj[1] - v_proj[2]
    vec5 = v_proj[1] - v_proj[3]
    
    t1 = sign(cross(vec1, vec2).z) + 1
    t2 = sign(cross(vec1, vec3).z) + 1
    t3 = sign(cross(vec2, vec3).z) + 1
    t4 = sign(cross(vec4, vec5).z) + 1
    
    ones_count = sum([t1==1, t2==1, t3==1, t4==1])
    if ones_count >= 3:
        return None # Discard degenerate
    
    class_id = ClassificationTexture.lookup(t1, t2, t3, t4)
    
    entry_point, exit_point = compute_ray_tet_intersection(ray_origin, ray_dir, verts)
    
    s_f = interpolate_scalar(entry_point, verts)
    s_b = interpolate_scalar(exit_point, verts)
    g_f = interpolate_gradient(entry_point, grads)
    g_b = interpolate_gradient(exit_point, grads)
    l = length(exit_point - entry_point)
    c_z = centroid(verts).z
    
    MRT1[tet_index] = (entry_point.x, entry_point.y, c_z, class_id)
    MRT2[tet_index] = (s_f, s_b, l, triangle_count(class_id))
    MRT3[tet_index] = g_f
    MRT4[tet_index] = g_b
    
    return class_id, v_indices

# Pass 2: Fragment Shader (Per Fragment)
def pass2_fragment_shader(fan_indices, fan_count, s_f_interp, s_b_interp, l_interp, g_f_interp, g_b_interp, isovalue, FramebufferColor):
    tau_f = sample_1d_texture(ExtinctionTexture, s_f_interp)
    tau_b = sample_1d_texture(ExtinctionTexture, s_b_interp)
    tau_avg = (tau_f + tau_b) / 2.0
    
    u = tau_avg * l_interp
    u_clamped = clamp(u, 0.0, 1.0)
    alpha = 1.0 - sample_1d_texture(ExponentialTexture, u_clamped)
    
    C_avg = sample_1d_texture(ColorTexture, (s_f_interp + s_b_interp) / 2.0)
    C = C_avg
    
    I_prev = FramebufferColor
    I_new = alpha * C + (1.0 - alpha) * I_prev
    
    if isovalue is not None and s_f_interp <= isovalue <= s_b_interp:
        N = g_f_interp
        LightDir = normalize(light_position - fragment_position)
        Diffuse = max(dot(N, LightDir), 0.0)
        Specular = pow(max(dot(reflect(-LightDir, N), view_dir), 0.0), shininess)
        I_new = Diffuse * C + Specular * ambient
        
    FramebufferColor = I_new
```

## Subtopics

### Pure Emissive Rendering
A variant that simply adds the contribution of each tetrahedron without complex compositing [S5]. It uses a 2D texture map containing the integral of the transfer function for the interval $[\min(s_f, s_b), \max(s_f, s_b)]$ at position $(s_f, s_b)$ [S5]. The dependency of color on ray segment length is linear, allowing integration by modulating the shading lookup with the minimum texture lookup using register combiners [S5].

### Rendering Algorithm (2nd Order)
A variant that projects front and back facing quadrilaterals of the bounding box to two distinct 32-bit floating point auxiliary buffers using a fragment shader that writes interpolated 3D texture coordinates [S6]. It uses the `GL_ARB_draw_buffers` extension for multiple render targets [S6].

## Operating characteristics
*   **Time Complexity:** GPU-bound; CPU is under-utilized [S2]. Performance dips slightly when the isosurface level changes due to pre-pass re-culling overhead [S2].
*   **Memory Complexity:** High. Pre-computation requires 160 bytes per tetrahedron [S1, unofficial]. Model size is restricted to 600,000 tetrahedra on an ATI Radeon 9700 (128 MB RAM) due to memory constraints [S1, unofficial].
*   **Performance:** Achieves interactive frame rates up to 2.0 M Tets/s on current graphics hardware [S3]. Mid-sized models achieve 2 to 5 fps [S1, unofficial]. Volumes up to $64^3$ run in interactive rates after optimization; larger volumes require multiple passes [S2].
*   **Strengths:** Interactive frame rates, supports interactive transfer function editing and isosurface highlighting with Phong shading [S3][S4, unofficial].
*   **Limitations:** High memory consumption; transfer function updates (changing opacity levels) require recreating arrays, reducing interactivity but increasing FPS [S4, unofficial]. Convolution of a $128^3$ volume with a $6^3$ filter takes >30 seconds [S2].
*   **When to use:** When interactive visualization of tetrahedral meshes is required on consumer hardware and memory allows for the pre-computed data structures [S1, unofficial].
*   **When not to use:** For very large volumes ($>64^3$) without multi-pass strategies, or when memory is severely constrained [S2].

## Background
The method was developed by Weiler et al. as an efficient ray-caster for tetrahedral meshes on consumer graphics cards [S1, unofficial]. It builds on the Projected Tetrahedra algorithm and incorporates techniques from BT Volume rendering [S2]. The approach was presented in papers around 2007-2008 [S3][S4, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Interactive volume isosurface rendering using BT volumes, https://doi.org/10.1145/1342250.1342257, fetched 2026-10-30
[S3] PDF Volume and Isosurface Rendering with GPU-Accelerated Cell Projection*, https://graphics.tudelft.nl/~marroquim/publications/pdfs/marroquim-CGF2008.pdf, fetched 2026-10-27
[S4] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S5] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S6] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
