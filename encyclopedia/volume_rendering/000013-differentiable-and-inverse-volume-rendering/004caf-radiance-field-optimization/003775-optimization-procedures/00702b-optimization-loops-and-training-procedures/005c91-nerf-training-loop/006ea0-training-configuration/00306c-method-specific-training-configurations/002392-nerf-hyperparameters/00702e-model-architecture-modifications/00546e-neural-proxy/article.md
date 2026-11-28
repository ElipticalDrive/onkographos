# Neural Proxy

Also called: Neural Proxy Model

## Overview
Neural Proxy is a neural rendering method that represents photo-realistic targets as animatable proxies to enable the rendering of unseen animations without temporal learning [S1, unofficial]. It combines neural volume rendering with neural texture techniques, injecting a proxy into a scene such that each proxy represents its photo-realistic counterpart, mimicking the typical CG pipeline of a 3D mesh for placement and a texture for appearance [S1, unofficial]. Unlike NeRF, which ties content to continuous Cartesian coordinates making animation difficult, Neural Proxy uses an animatable mesh proxy allowing particles to move with the animation [S1, unofficial]. Unlike Neural Texture, which requires animated training samples to render unseen animations, Neural Proxy renders unseen animations without temporal learning by animating the proxy geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference geometry (mesh $\bar{M}$), UV coordinates for the mesh, offset parameters for shell extrusion ($\epsilon_{\rightarrow}, \epsilon_{\leftarrow}$), and a neural texture $\Omega_{Tex}$.
*   **Outputs:** Rendered image $I$ for a given camera pose and animation state.

**Data Structures:**
1.  **Coarse Geometry ($\bar{M}$):** A mesh defined by vertices $\mathbf{v}_i$ and smoothed normals $\mathbf{n}_i$ for $i=1 \dots N_v$.
2.  **Proxy Shell:** A volume enclosing the target, constructed by extruding $\bar{M}$ along normals. It consists of an outward surface $M_{\rightarrow}$ and an inward surface $M_{\leftarrow}$.
    *   Outward vertices: $\overrightarrow{\mathbf{v}}_i = \mathbf{v}_i + \mathbf{n}_i \cdot \epsilon_{\rightarrow}$
    *   Inward vertices: $\overleftarrow{\mathbf{v}}_i = \mathbf{v}_i - \mathbf{n}_i \cdot \epsilon_{\leftarrow}$
    *   Constraints: $\epsilon_{\rightarrow}, \epsilon_{\leftarrow} \ge 0$ and $\epsilon_{\rightarrow} + \epsilon_{\leftarrow} > 0$.
3.  **UV Coordinates:**
    *   Original mesh: $\mathbf{u}_{2D}$ for each vertex.
    *   Extruded surfaces: Lifted to 3D with a depth component $w$.
        *   Outward: $\overrightarrow{\mathbf{u}}_i = (\mathbf{u}_{2D}, \overrightarrow{w})$ where $\overrightarrow{w}=1$.
        *   Inward: $\overleftarrow{\mathbf{u}}_i = (\mathbf{u}_{2D}, \overleftarrow{w})$ where $\overleftarrow{w}=0$.
    *   The proxy volume includes vertices from both surfaces: $\{\overrightarrow{\mathbf{v}}_i\} \cup \{\overleftarrow{\mathbf{v}}_i\}$.
4.  **Neural Texture ($\Omega_{Tex}$):** A learnable feature map wrapped around the proxy.

**Steps:**
1.  **Proxy Construction:** Retrieve reference geometry $\bar{M}$ (e.g., via COLMAP or manual low-res mesh) [S1, unofficial].
2.  **Shell Extrusion:** Compute outward and inward surfaces $M_{\rightarrow}$ and $M_{\leftarrow}$ using the offset parameters [S1, unofficial].
3.  **UV Lifting:** Assign 3D UV coordinates to the shell vertices using the depth labels $w=1$ (outward) and $w=0$ (inward) [S1, unofficial].
4.  **Tetrahedralization:** Break the prisms formed by the extrusion of triangles into three tetrahedra using the method from [PBFJ05] to enable Barycentric coordinate sampling [S1, unofficial].
5.  **Neural Texture Wrapping:** Wrap the neural texture $\Omega_{Tex}$ around the proxy shell [S1, unofficial].
6.  **Particle Sampling:**
    *   For a ray $r$, determine intersection with the proxy shell.
    *   Convert the Cartesian intersection point $\mathbf{x}$ to Barycentric coordinates within the tetrahedra.
    *   Map Barycentric coordinates to UV coordinates on the neural texture.
    *   Use a bilinear sampler $f_{Bi}$ to sample features from $\Omega_{Tex}$ based on UV coordinates.
    *   Feed sampled features into a neural proxy sampler $S_p$ to produce color $\mathbf{c}$ and density $\sigma$. Note: The view direction $\mathbf{d}$ is dropped for simplicity in the proxy sampler definition, or handled implicitly by the texture [S1, unofficial].
    *   Formula: $(\mathbf{c}, \sigma) = S_p(\mathbf{x})$ where the input to $S_p$ is derived from the texture sample [S1, unofficial].
7.  **Rendering:**
    *   Use a mixture of a neural proxy sampler (for animatable targets) and a static sampler (for static surroundings) for particle sampling [S1, unofficial].
    *   Render the image using standard volume rendering integration over the sampled particles [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_neural_proxy(ray, proxy_shell, neural_texture, static_sampler):
    """
    Renders a single ray using the Neural Proxy method.
    
    Args:
        ray: Ray origin and direction.
        proxy_shell: Object containing mesh vertices, normals, UVs, and tetrahedra.
        neural_texture: The learnable texture map Omega_Tex.
        static_sampler: Sampler for static background/surroundings.
        
    Returns:
        Rendered color and density.
    """
    # 1. Intersect ray with proxy shell
    intersection = intersect_ray_with_shell(ray, proxy_shell)
    
    if intersection is not None:
        # 2. Convert Cartesian intersection point to Barycentric coordinates
        # This requires identifying the specific tetrahedron in the proxy shell
        barycentric_coords = cartesian_to_barycentric(intersection.point, proxy_shell.tetrahedra)
        
        # 3. Map Barycentric coordinates to UV coordinates
        # The UVs are pre-computed for the shell vertices
        uv_coords = barycentric_to_uv(barycentric_coords, proxy_shell.uv_map)
        
        # 4. Sample features from the neural texture using bilinear sampling
        # f_Bi is the bilinear sampler
        features = bilinear_sampler(neural_texture, uv_coords)
        
        # 5. Generate color and density using the neural proxy sampler
        # S_p takes the sampled features (and potentially the Cartesian point x)
        # Note: View direction d is dropped in the simplified definition
        color, density = neural_proxy_sampler(features, intersection.point)
        
        return color, density
    else:
        # 6. If no intersection with proxy, use static sampler for surroundings
        return static_sampler(ray)

def build_proxy_shell(mesh, normals, epsilon_out, epsilon_in):
    """
    Constructs the proxy shell from a coarse mesh.
    
    Args:
        mesh: List of vertices v_i.
        normals: List of smoothed normals n_i.
        epsilon_out: Offset for outward surface.
        epsilon_in: Offset for inward surface.
        
    Returns:
        ProxyShell object containing extruded vertices, UVs, and tetrahedra.
    """
    N_v = len(mesh)
    outward_vertices = []
    inward_vertices = []
    
    for i in range(N_v):
        v = mesh[i]
        n = normals[i]
        
        # Extrude vertices
        v_out = v + n * epsilon_out
        v_in = v - n * epsilon_in
        
        outward_vertices.append(v_out)
        inward_vertices.append(v_in)
        
    # Lift UV coordinates to 3D
    # Original UVs are assumed to be available for the coarse mesh
    outward_uvs = [(uv, 1.0) for uv in original_mesh_uvs]
    inward_uvs = [(uv, 0.0) for uv in original_mesh_uvs]
    
    # Combine vertices for the shell
    all_shell_vertices = outward_vertices + inward_vertices
    all_shell_uvs = outward_uvs + inward_uvs
    
    # Break prisms into tetrahedra for Barycentric sampling
    # Method from [PBFJ05]
    tetrahedra = decompose_prisms_into_tetrahedra(mesh, outward_vertices, inward_vertices)
    
    return ProxyShell(all_shell_vertices, all_shell_uvs, tetrahedra)
```

**Parameters:**
*   $\epsilon_{\rightarrow}, \epsilon_{\leftarrow}$: Offset parameters for shell extrusion. Must be non-negative and sum to $>0$. Set such that the shell encloses the target object [S1, unofficial].
*   $w$: Depth component for UV lifting. $\overrightarrow{w}=1$ for outward, $\overleftarrow{w}=0$ for inward [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on accurate intersection detection with the proxy shell. Degenerate tetrahedra or poor mesh quality in the reference geometry can lead to sampling errors [S1, unofficial].
*   **Boundaries:** The shell must fully enclose the photo-realistic target to ensure relevant particles are contained [S1, unofficial].
*   **Animation:** The proxy geometry is animated. If the animation causes self-intersection or extreme deformation, the Barycentric mapping may fail or produce artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The proposed model significantly outperforms current neural rendering works in rendering unseen animations [S1, unofficial].
*   **Strengths:** Enables animation of neural rendering targets without temporal learning; combines the localization of neural texture with the volumetric nature of NeRF.
*   **Limitations:** Requires a reference geometry proxy; limited to animatable targets that can be represented by a mesh proxy.

## Background
Neural Proxy was proposed to address the lack of discussion on enabling neural rendering for synthesizing frames for unseen animations [S1, unofficial]. It builds on Neural Volume Rendering [LSS*19] and Neural Radiance Fields (NeRF) [MST*20], while incorporating ideas from Neural Texture [TZN19]. The method mimics the typical computer graphics pipeline by using a 3D mesh for placement and a texture for appearance [S1, unofficial].

## Sources
[S1] Neural Proxy: Empowering Neural Volume Rendering for Animation, https://doi.org/10.2312/pg.20211384, fetched 2026-10-26 (unofficial)
