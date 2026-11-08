# Absorption-Only Radiography via Unsorted Hexahedron Projection

Also called: Unsorted Hexahedral Radiography, Additive Optical Thickness Projection

## Overview
This method simulates X-ray radiographs for curvilinear hexahedral meshes in the absorption-only regime ($J_\nu = 0$) by accumulating optical thickness additively, thereby eliminating the need for a global back-to-front visibility sort [S1, unofficial]. It differs from standard polyhedron projection methods, which typically require depth sorting to correctly composite front-to-back, by leveraging the commutative property of addition in the absorption-only recurrence relation [S1, unofficial]. It builds on [Hardware-Assisted Projection](5625-hardware-assisted-projection.md) by utilizing hardware fragment shaders for accumulation but specifically addresses the geometric complexity of hexahedral cells using Max's projection method [S1, unofficial].

## How it works
**Inputs:** A curvilinear hexahedral mesh (or a mixed mesh with tetrahedral fallbacks); extinction coefficients $K_\nu$ per cell or vertex; camera projection matrix.
**Outputs:** A 2D radiograph image representing integrated absorption.

**Data Structures:**
*   **P-buffer:** A floating-point off-screen buffer (texture) aligned with the image plane. Stores the accumulated optical thickness $\tau_{acc}$. Format: 32-bit floating point per channel (RGBA) [S1, unofficial].
*   **Cell Geometry:** Hexahedral cells defined by 8 vertices. Each vertex carries interpolated attenuation values.

**Algorithm Steps:**

1.  **Cell Classification and Tessellation:**
    For each hexahedral cell, determine if it can be projected directly. A cell is valid for direct projection if:
    *   It is non-degenerate (no coinciding vertices).
    *   The projection of each of its 6 faces onto the image plane is a convex quadrilateral [S1, unofficial].
    
    If valid, the cell is tessellated into triangle fans or strips using the method of Max [12] (see [Max Hexahedron Projection](link)). If invalid (degenerate or non-convex face projections), the cell is subdivided into tetrahedra using the method of Shirley and Tuchman [16] or Max et al. [13] [S1, unofficial].

2.  **Software Pre-computation (Thickness):**
    For each vertex of the cell (or tetrahedron), compute the geometric thickness $t_v$ along the viewing ray direction. This is performed in software using the cell's geometry [S1, unofficial].
    Compute the vertex attenuation $A_v$:
    $$ A_v = K_\nu \cdot t_v $$
    where $K_\nu$ is the extinction coefficient for the wavelength band [S1, unofficial].

3.  **Hardware Accumulation (Fragment Shader):**
    Render the tessellated primitives (triangles) to the P-buffer.
    *   **Vertex Shader:** Passes the pre-computed attenuation $A_v$ and position.
    *   **Fragment Shader:**
        1.  Interpolate the attenuation $A_{frag}$ from the vertex data across the triangle [S1, unofficial].
        2.  Read the current accumulated attenuation $\tau_{curr}$ from the P-buffer texture at the fragment's screen coordinate.
        3.  Compute the new accumulated value:
            $$ \tau_{new} = \tau_{curr} + A_{frag} $$
        4.  Write $\tau_{new}$ back to the P-buffer [S1, unofficial].

4.  **Texture Synchronization:**
    After rendering a batch of cells, copy the P-buffer contents to a read-only texture to ensure the texture cache is updated. This is necessary because writes to output buffers are not guaranteed to be in polygon order, and the hardware texture caching mechanism does not account for textures changing on the fly [S1, unofficial].

5.  **Final Image Generation:**
    After all cells are processed, the P-buffer contains the total optical thickness $\tau_{total}$ per pixel. The final intensity $I$ is computed from the initial intensity $I_0$ (source brightness) using the Beer-Lambert law:
    $$ I = I_0 e^{-\tau_{total}} $$
    This final conversion can be done in a post-processing pass or by interpreting the P-buffer values [S1, unofficial].

**Parallelism:**
The method supports accumulating attenuation for four different X-ray wavelength bands in parallel by using the Red, Green, Blue, and Alpha channels of the P-buffer texture [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-pass: Tessellation and Thickness Calculation
def preprocess_mesh(hex_mesh):
    primitives = []
    for cell in hex_mesh:
        if is_valid_hexahedron(cell):
            # Max's method for convex face projections
            triangles = max_hexahedron_to_triangles(cell)
        else:
            # Subdivide into tetrahedra
            tetrahedra = subdivide_hexahedron_to_tetrahedra(cell)
            triangles = []
            for tet in tetrahedra:
                triangles.extend(shirley_tuchman_tet_to_triangles(tet))
        
        for tri in triangles:
            # Compute thickness and attenuation in software
            attenuations = []
            for vertex in tri.vertices:
                thickness = compute_geometric_thickness(vertex, cell_geometry)
                att = vertex.extinction_coefficient * thickness
                attenuations.append(att)
            primitives.append({
                'vertices': tri.vertices,
                'attenuations': attenuations
            })
    return primitives

# Rendering Pass
def render_radiograph(primitives, p_buffer_texture):
    # Batch rendering to minimize texture copies
    # Note: Exact batching logic depends on hardware limits; 
    # sources suggest rendering non-overlapping cells where possible [S1, unofficial]
    
    for batch in split_into_batches(primitives):
        for prim in batch:
            render_primitive(prim, p_buffer_texture)
        
        # Synchronize texture cache
        copy_p_buffer_to_texture(p_buffer_texture)

def render_primitive(prim, p_buffer_texture):
    # Vertex Shader
    # Input: prim.vertices, prim.attenuations
    # Output: gl_Position, frag_attenuation
    
    # Fragment Shader
    # Input: frag_attenuation (interpolated)
    # Output: out_color
    
    current_tau = read_texture(p_buffer_texture, gl_FragCoord.xy)
    new_tau = current_tau + frag_attenuation
    write_texture(p_buffer_texture, gl_FragCoord.xy, new_tau)

# Post-process
def generate_image(p_buffer_texture, initial_intensity):
    for pixel in p_buffer_texture:
        tau = pixel.value
        intensity = initial_intensity * exp(-tau)
        pixel.value = intensity
```

**Edge Cases and Pitfalls:**
*   **Degenerate Cells:** Hexahedral cells with coinciding vertices or non-convex face projections must be subdivided. If these resulting tetrahedra share non-planar faces with adjacent non-subdivided cells, gaps or overlaps in viewing ray segments may occur, introducing small errors in total absorption [S1, unofficial].
*   **Precision:** Graphics chips may support 32-bit internal fragment computations, but framebuffer compositing operations often lack this accuracy. Using a floating-point P-buffer is required for correct attenuation accumulation [S1, unofficial].
*   **Texture Caching:** Failure to copy the P-buffer to a texture between batches results in stale data reads due to hardware caching mechanisms not accounting for on-the-fly texture changes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cells, assuming constant-time projection and rasterization. The cost is dominated by the number of fragments generated and the texture copy overhead.
*   **Memory Complexity:** $O(W \times H)$ for the P-buffer, where $W, H$ are image dimensions. Additional memory for tessellated primitives on the GPU.
*   **Performance:** The method avoids the $O(N \log N)$ or worse cost of global sorting. However, the necessity of copying the P-buffer to a texture between batches introduces latency. Rendering as many non-overlapping cells as possible in a single pass minimizes these copies [S1, unofficial].
*   **Strengths:** No global sort required; hardware-accelerated accumulation; supports curvilinear hexahedra directly (saving storage and rendering time compared to tetrahedral conversion); supports multi-spectral (4-channel) accumulation.
*   **Limitations:** Limited to absorption-only (no emission); requires floating-point framebuffer support; potential errors from non-planar face artifacts in mixed tessellation; texture copy overhead can be significant if many batches are needed.
*   **When to use:** Industrial and medical X-ray simulation where self-emission is negligible and high-performance radiography of hexahedral meshes is required.
*   **When not to use:** Scenes with significant self-emission ($J_\nu \neq 0$); meshes with severe degeneracy that cannot be efficiently tessellated; hardware lacking floating-point framebuffer support.

## Background
This technique was developed for industrial and medical X-ray simulation where the self-emission of X-rays from the subject is negligible [S1, unofficial]. It extends the polyhedron projection method by leveraging the commutative property of addition in the absorption-only recurrence relation to bypass the need for back-to-front sorting [S1, unofficial]. The geometric projection of hexahedra relies on the method of Max [12], while tetrahedral fallbacks use Shirley and Tuchman [16] or Max et al. [13]. The use of a floating-point P-buffer and texture synchronization addresses the precision and caching limitations of early graphics hardware [S1, unofficial].

## Sources
[S1] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
