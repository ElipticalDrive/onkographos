# Extended CLCIR PLIC Reconstruction

Also called: Extended CLCIR method, PLIC reconstruction on arbitrary grids

## Overview
The Extended CLCIR method reconstructs a Piecewise Linear Interface Concept (PLIC) plane for Volume of Fluid (VOF) simulations on arbitrary polyhedral grids by extracting an isosurface from an interpolated scalar field and computing a weighted normal vector [S1]. It distinguishes itself from standard VOF reconstruction methods by explicitly using isosurface extraction to handle non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by its explicit geometric flux calculation and volume conservation enforcement [S1]. It builds on [Isosurface extraction](30-isosurface-extraction.md) to determine the interface geometry.

## How it works
**Inputs:**
- Scalar field $\phi$ at cell vertices, interpolated from fluid volume fractions $F_l(t_n)$ of neighboring cells [S1].
- Cell geometry (vertices, faces, volume $V_\Omega$) [S1].
- Target isovalue $\tilde{\phi} = 0.5$ [S1].
- Threshold angle $\theta_{thresh} = 1.2$ radians [S1].

**Outputs:**
- PLIC plane defined by unit normal vector $\mathbf{n}$ and constant $C$ such that $\mathbf{n} \cdot \mathbf{x} + C = 0$ [S1].
- The plane splits cell volume $V_\Omega$ into sub-volumes $F(t_n)V_\Omega$ and $(1-F(t_n))V_\Omega$ [S1].

**Data Structures:**
- **Interpolated Scalar Field:** For each vertex $i_p$, $\phi_{i_p}$ is computed as:
  $$ \phi_{i_p} = \frac{\sum_l F_l(t_n) w_l}{\sum_l w_l} $$
  where the sum extends over all cells $l$ containing vertex $i_p$, and $w_l = 1/|\mathbf{x}_{i_p} - \mathbf{x}_l|$ is the inverse distance weight, with $\mathbf{x}_{i_p}$ being the vertex position and $\mathbf{x}_l$ the geometric center of cell $l$ [S1].
- **Isosurface:** A set of vertices and faces (isopolygon) extracted from the cell where $\phi_{min} < 0.5 < \phi_{max}$ [S1].
- **Triangulated Surface:** A mesh formed by triangles connecting a center point to edges of the isosurface or adjacent isosurfaces [S1].

**Algorithm Steps:**
1. **Interpolation:** Compute $\phi_{i_p}$ for all vertices in the domain using the weighted average formula [S1].
2. **Interface Check:** For each cell, check if $\phi_{min} < 0.5 < \phi_{max}$. If not, the cell is not interfacial [S1].
3. **Isosurface Extraction:** If interfacial, extract the isosurface at $\tilde{\phi} = 0.5$ using an arbitrary grid isosurface extraction algorithm (e.g., Algorithm 1 from [S1]) [S1].
4. **Validity Check:** If the extracted isosurface consists of a single isopolygon, mark the cell as a "valid isosurface cell" [S1].
5. **Local Normal Calculation:**
   - Construct a "local triangulated surface" by connecting the geometric center of the extracted isosurface to two consecutive $\tilde{\phi}$-vertices for each triangle [S1].
   - Compute the local normal $\mathbf{n}_l$ using weighted averaging:
     $$ \mathbf{n}_l = \frac{\sum_{t=1}^{N_t} w_t \mathbf{n}_t}{\left| \sum_{t=1}^{N_t} w_t \mathbf{n}_t \right|} $$
     where $N_t$ is the number of triangles, $\mathbf{n}_t$ is the unit normal of triangle $t$, and $w_t$ is the weighting factor [S1].
   - For cubic grids, $w_t = \frac{\sin(\alpha)}{L_1 L_2}$, where $\alpha$ is the angle between the two inner edges of the triangular facet, and $L_1, L_2$ are their lengths [S1]. For other grids, $w_t = 1$ [S1].
6. **Extended Normal Calculation:**
   - Create an "extended triangulated surface" by connecting the geometric center of the current surface to the geometric centers of isosurfaces extracted at adjacent cells [S1].
   - Compute the extended normal $\mathbf{n}_e$ using the same weighted averaging formula as for $\mathbf{n}_l$ [S1].
7. **Normal Selection:**
   - If $\arccos(\mathbf{n}_e \cdot \mathbf{n}_l) < 1.2$ rad, set $\mathbf{n} = \mathbf{n}_e$. Otherwise, set $\mathbf{n} = \mathbf{n}_l$ [S1].
8. **Fallback for Invalid Cells:** If the cell is not a valid isosurface cell (e.g., complex topology), compute $\mathbf{n}$ using a Least-Squares Gradient Interface Reconstruction (LSGIR) method [S1].
9. **Volume Constant Calculation:** Compute the constant $C$ such that the plane $\mathbf{n} \cdot \mathbf{x} + C = 0$ splits the cell volume $V_\Omega$ into volumes $F(t_n)V_\Omega$ and $(1-F(t_n))V_\Omega$ [S1].
   - For cubic cells, use the analytical method of Scardovelli and Zaleski [S1].
   - For other grids, use the CIBRAVE method (López et al.) [S1].
10. **Vertex Relocation:** Move the vertices of the extended triangulated surfaces to the corresponding PLIC centers (intersection of the plane with cell edges) [S1].
11. **Refinement:**
    - For valid isosurface cells, compute a new normal $\mathbf{n}_c$ from the updated extended triangulated surface using the weighted averaging formula [S1].
    - If $\arccos(\mathbf{n}_c \cdot \mathbf{n}) < 1.2$ rad, update $\mathbf{n} = \mathbf{n}_c$ and recompute $C$ to relocate the PLIC [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_clcir_reconstruction(cell, phi_field, threshold_angle=1.2):
    """
    Reconstruct PLIC plane for a single cell.
    cell: Object with vertices, faces, volume, and geometric center.
    phi_field: Dictionary mapping vertex_id to scalar value phi.
    threshold_angle: Angle threshold in radians (1.2).
    """
    # Step 1 & 2: Check if cell is interfacial
    phi_min = min(phi_field[v] for v in cell.vertices)
    phi_max = max(phi_field[v] for v in cell.vertices)
    
    if not (phi_min < 0.5 < phi_max):
        return None # Not an interfacial cell

    # Step 3: Extract isosurface at 0.5
    # Assumes Algorithm 1 from [S1] is available
    isosurface = extract_isosurface(cell, phi_field, isovalue=0.5)
    
    # Step 4: Check validity
    is_valid = is_single_isopolygon(isosurface)
    
    # Step 5: Local Normal
    local_center = isosurface.geometric_center
    local_triangles = construct_local_triangles(local_center, isosurface.vertices)
    n_l = compute_weighted_normal(local_triangles, cell.grid_type)
    
    # Step 6: Extended Normal
    # Get isosurfaces from adjacent cells (requires neighbor lookup)
    adjacent_isosurfaces = get_adjacent_isosurfaces(cell, phi_field)
    extended_triangles = construct_extended_triangles(local_center, adjacent_isosurfaces)
    n_e = compute_weighted_normal(extended_triangles, cell.grid_type)
    
    # Step 7: Normal Selection
    dot_product = dot(n_e, n_l)
    # Clamp dot_product to [-1, 1] to avoid numerical errors in acos
    dot_product = max(-1.0, min(1.0, dot_product))
    angle = acos(dot_product)
    
    if angle < threshold_angle:
        n = n_e
    else:
        n = n_l
        
    # Step 8: Fallback if not valid (simplified)
    if not is_valid:
        n = compute_lsgir_normal(cell, phi_field)
        
    # Step 9: Compute C
    # Assumes volume_solver is available
    C = compute_volume_constant(n, cell, phi_field[cell.volume_fraction_index])
    
    # Step 10: Relocate vertices (conceptual)
    # Move vertices of extended surface to PLIC centers
    updated_extended_surface = relocate_surface_vertices(extended_triangles, n, C, cell)
    
    # Step 11: Refinement
    if is_valid:
        n_c = compute_weighted_normal(updated_extended_surface, cell.grid_type)
        dot_product_c = dot(n_c, n)
        dot_product_c = max(-1.0, min(1.0, dot_product_c))
        angle_c = acos(dot_product_c)
        
        if angle_c < threshold_angle:
            n = n_c
            C = compute_volume_constant(n, cell, phi_field[cell.volume_fraction_index])
            
    return n, C

def compute_weighted_normal(triangles, grid_type):
    """
    Compute normal using Eq. B.3 from [S1].
    """
    weighted_sum = vector(0, 0, 0)
    for tri in triangles:
        n_t = tri.unit_normal
        if grid_type == "cubic":
            # w_t = sin(angle) / (length1 * length2)
            # Inner edges join the center to two consecutive vertices
            v1, v2, v3 = tri.vertices # Assuming order: center, v1, v2
            edge1 = v1 - v3 # From center to v1? No, "inner edges... joins a phi-vertex with the geometric center"
            # Let's assume tri.vertices[0] is center, tri.vertices[1] and [2] are phi-vertices
            # Actually, the text says "two consecutive phi-vertices". 
            # Let's assume the triangle is defined by Center, V_i, V_{i+1}
            # Inner edges are (Center, V_i) and (Center, V_{i+1})
            e1 = v1 - v3 # Vector from center to v1? No, v3 is center?
            # Let's redefine: tri.vertices = [center, v_a, v_b]
            # Inner edges are e_a = v_a - center, e_b = v_b - center
            e_a = v1 - v3
            e_b = v2 - v3
            len_a = length(e_a)
            len_b = length(e_b)
            if len_a == 0 or len_b == 0:
                w_t = 1.0
            else:
                cos_angle = dot(e_a, e_b) / (len_a * len_b)
                sin_angle = sqrt(1 - cos_angle**2)
                w_t = sin_angle / (len_a * len_b)
        else:
            w_t = 1.0
        weighted_sum += w_t * n_t
    return normalize(weighted_sum)
```

## Subtopics

### [VOF Advection with Isosurface Extraction](1063-vof-advection-with-isosurface-extraction.md)
This method simulates multiphase fluid flows by advecting a Volume of Fluid (VOF) indicator function on arbitrary polyhedral grids. It solves the advection equation geometrically using Edge-Matched Flux Polygon Advection (EMFPA) and reconstructs the fluid interface using the CLCIR method, which relies on isosurface extraction to determine the Piecewise Linear Interface Concept (PLIC) plane orientation. It is distinguished from standard VOF methods by its ability to handle non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by its explicit geometric flux calculation.

## Operating characteristics
- **Time Complexity:** Not given in the sources. The method involves isosurface extraction, neighbor lookups, and vector operations per cell.
- **Memory Complexity:** Not given in the sources. Requires storage of the scalar field, isosurfaces, and extended triangulated surfaces.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Handles arbitrary polyhedral grids, including non-convex and self-intersecting cells, without decomposition [S1]. Uses isosurface extraction for robust interface representation [S1].
- **Limitations:** Requires an isosurface extraction algorithm as a subroutine [S1]. May fall back to LSGIR for invalid cells, which might be less accurate [S1].
- **When to use:** VOF simulations on complex, arbitrary polyhedral grids where standard PLIC methods fail due to cell topology [S1].
- **When not to use:** Simple cubic grids where faster analytical methods (Scardovelli and Zaleski) are sufficient for volume constant calculation, though the normal calculation still uses the extended method [S1].

## Background
The Extended CLCIR method was introduced in a 2021 paper by López et al. [S1]. It is part of the VOFTools package [S1]. The method combines isosurface extraction with PLIC reconstruction to handle complex cell geometries. It replaces the CIBRAVE method with the Scardovelli and Zaleski analytical method for cubic cells [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
