# Sorted Rasterization of Projected Tetrahedra

Also called: GPU DVR for unstructured data via sorted PT, Radiance Mesh Rendering via Hardware Rasterization, Tet-by-Tet Ray Casting

## Overview
Sorted Rasterization of Projected Tetrahedra is a GPU-based direct volume rendering (DVR) method for unstructured tetrahedral data that achieves interactive rates by treating each tetrahedron as a distinct primitive and rasterizing them front-to-back [S1, S2, unofficial]. It differs from traditional ray casting by leveraging hardware rasterization and blending operations to compute the final pixel color, avoiding the need for explicit ray-tetrahedron intersection tests per pixel [S1, unofficial]. This approach builds on [GPU-Based Projected Tetrahedra](1951-gpu-based-projected-tetrahedra.md) by optimizing the rendering pipeline for modern programmable graphics cards [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh where each tetrahedron $k$ has constant density $\sigma_k$ and color fields $\mathbf{c}_k(\mathbf{r})$ defined at any point $\mathbf{r}$ within it. The camera position (origin) is required for sorting.
*   **Output:** A 2D image where each pixel contains the accumulated color $C$ and opacity $\alpha$ based on front-to-back blending.

**Data Structures:**
*   **Tetrahedron Attributes:** For each tetrahedron $k$, store:
    *   Vertices: 4 vertices defining the geometry.
    *   Density: Scalar $\sigma_k$.
    *   Color Fields: Functions or values defining $\mathbf{c}_k^{\text{in}}$ and $\mathbf{c}_k^{\text{out}}$ at entry and exit points.
    *   Circumsphere Power: A scalar value used for sorting relative to the camera origin.
*   **Sorted List:** Tetrahedra sorted by their circumsphere power relative to the camera, loaded from back to front.

**Numbered Steps:**

1.  **Sorting:** Sort all tetrahedra based on the power of their circumspheres relative to the camera origin. This ensures a back-to-front loading order for the rasterizer [S1, unofficial].
2.  **Rendering Setup:**
    *   **Without Mesh Shaders:** Use instanced shaders. Draw an instance of a triangle strip for each tetrahedron. The vertex shader uses the instance index to load tetrahedron attributes and plane equations. Each tetrahedron is loaded 4 times, once per vertex [S1, unofficial].
    *   **With Mesh Shaders:** Replace vertex shaders with mesh shaders. An entire thread block collaboratively generates mesh data passed directly to the hardware rasterizer. Use warp shuffle operations to share data loaded within each tetrahedron, loading each tetrahedron only once [S1, unofficial].
3.  **Vertex Processing:** For each tetrahedron, emit 4 vertices with tetrahedron data attached to each to the interpolation registers. The data includes information necessary to reconstruct the back intersection in the fragment shader [S1, unofficial].
4.  **Fragment Shader Execution:**
    *   For each pixel hit by a tetrahedron, read back the whole tetrahedron data attached to the triangle hit to reconstruct the back intersection [S1, unofficial].
    *   Compute the entry point $t_k^{\text{in}}$ and exit point $t_k^{\text{out}}$ along the ray $\mathbf{r}$ for the tetrahedron.
    *   Calculate the optical depth $d_k$ and opacity $\alpha_k$:
        $$d_k = (t_k^{\text{out}} - t_k^{\text{in}}) \sigma_k$$
        $$\alpha_k = 1 - e^{-d_k}$$
    *   Evaluate the color at the entry and exit points:
        $$\mathbf{c}_k^{\text{in}} = \mathbf{c}_k(\mathbf{r}(t_k^{\text{in}}))$$
        $$\mathbf{c}_k^{\text{out}} = \mathbf{c}_k(\mathbf{r}(t_k^{\text{out}}))$$
    *   Compute the color change $\Delta \mathbf{C}_k$ using the closed-form solution:
        $$\Delta \mathbf{C}_k = \left(1 - \frac{\alpha_k}{d_k}\right) \mathbf{c}_k^{\text{in}} + \left(\frac{\alpha_k}{d_k} - e^{-d_k}\right) \mathbf{c}_k^{\text{out}}$$
5.  **Compositing:** Blend the pre-sorted tetrahedra along the ray. The final pixel color is:
    $$C = \sum_k w_k \Delta C_k$$
    where the weight $w_k = \prod_{l=1}^{k-1} (1 - \alpha_\ell)$ is efficiently computed using hardware blending operations [S1, unofficial]. The compositor composites the fragment according to the load order (back to front) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume tetrahedra are sorted back-to-front in a list `sorted_tets`
# Camera origin is `camera_origin`
# Ray for pixel (x, y) is `ray`

def render_pixel(ray, sorted_tets):
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    
    for tet in sorted_tets:
        # 1. Intersect ray with tetrahedron
        t_in, t_out = intersect_ray_tet(ray, tet)
        
        # If no intersection, skip
        if t_in is None or t_out is None:
            continue
            
        # 2. Compute optical depth and opacity
        sigma = tet.density
        d_k = (t_out - t_in) * sigma
        alpha_k = 1.0 - exp(-d_k)
        
        # If fully transparent, skip
        if alpha_k < 1e-6:
            continue
            
        # 3. Evaluate colors at entry and exit
        c_in = tet.color_field(ray.origin + ray.direction * t_in)
        c_out = tet.color_field(ray.origin + ray.direction * t_out)
        
        # 4. Compute color change
        if d_k < 1e-6:
            delta_C = c_in # Avoid division by zero
        else:
            delta_C = (1.0 - alpha_k / d_k) * c_in + (alpha_k / d_k - exp(-d_k)) * c_out
            
        # 5. Blend into accumulated color
        # Weight w_k is product of (1 - alpha_l) for all previous tets
        # This is maintained in accumulated_opacity: w_k = 1 - accumulated_opacity
        weight = 1.0 - accumulated_opacity
        accumulated_color += weight * delta_C
        accumulated_opacity += weight * alpha_k
        
        # Early exit if fully opaque
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color, accumulated_opacity

def intersect_ray_tet(ray, tet):
    # Compute intersection of ray with tetrahedron
    # Returns (t_in, t_out) or (None, None) if no intersection
    # This involves solving for intersections with the 4 planes of the tetrahedron
    # and finding the entry and exit points along the ray segment within the tet
    pass
```

**Parameters:**
*   **Density ($\sigma_k$):** Constant for each tetrahedron. Typical values depend on the dataset.
*   **Color Fields ($\mathbf{c}_k$):** Defined by the dataset.
*   **Sorting Criterion:** Power of circumsphere relative to camera origin.

**Edge Cases and Pitfalls:**
*   **Division by Zero:** When $d_k$ is very small, $\frac{\alpha_k}{d_k}$ approaches 1. Handle this case to avoid numerical instability [S1, unofficial].
*   **Precision:** Interpolation of tetrahedron data in the fragment shader must be precise enough to reconstruct the back intersection accurately [S1, unofficial].
*   **Degenerate Tetrahedra:** Tetrahedra with zero volume or near-zero volume should be handled or filtered out.

## Subtopics

### Radiance Meshes
A specific implementation of sorted rasterization of projected tetrahedra that treats each tetrahedron as a distinct primitive and outputs individually wound tetrahedra [S1, unofficial].

### Instanced Rendering
A rendering technique used when mesh shaders are not available, drawing an instance of a triangle strip for each tetrahedron [S1, unofficial].

### Mesh Shaders
A modern GPU feature that allows collaborative generation of mesh data, reducing redundant data loads [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting takes $O(N \log N)$ where $N$ is the number of tetrahedra. Rasterization is $O(N \cdot P)$ where $P$ is the number of pixels, but hardware blending optimizes the compositing step.
*   **Memory Complexity:** $O(N)$ to store tetrahedron attributes.
*   **Performance:** Achieves interactive rates on commodity GPUs [S1, unofficial]. Mesh shaders provide the fastest rendering approach by reducing redundant data loads [S1, unofficial].
*   **Strengths:** Leverages hardware rasterization and blending, avoiding explicit ray casting per pixel. Efficient for unstructured data.
*   **Limitations:** Requires sorting of tetrahedra. Performance may degrade with very large datasets if sorting becomes a bottleneck.
*   **When to Use:** Interactive volume rendering of unstructured tetrahedral data on modern GPUs.
*   **When Not to Use:** When mesh shaders are not available and performance is critical (instanced rendering is slower). When the dataset is too large for efficient sorting.

## Background
The method is part of the broader field of GPU-based direct volume rendering for unstructured data [S2, unofficial]. It builds on the concept of projected tetrahedra and optimizes it for hardware rasterization [S1, unofficial]. The specific implementation using mesh shaders and warp shuffle operations is described in [S1, unofficial].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
