# Extended CLCIR PLIC Reconstruction

Also called: Extended CLCIR method, PLIC reconstruction on arbitrary grids

## Overview
The Extended CLCIR method reconstructs a Piecewise Linear Interface Concept (PLIC) plane for Volume of Fluid (VOF) simulations on arbitrary polyhedral grids by extracting an isosurface from an interpolated scalar field and computing a weighted normal vector [S1]. It distinguishes itself from standard VOF methods by handling non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by explicitly calculating geometric fluxes for advection [S1]. It builds on [Isosurface extraction](30-isosurface-extraction.md) to determine interface orientation in complex topologies.

## How it works
**Inputs:**
- Scalar field $\phi$ at cell vertices, interpolated from fluid volume fractions $F_l(t_n)$ of neighboring cells [S1].
- Cell geometry (vertices, faces, volume $V_\Omega$) for arbitrary polyhedral cells [S1].
- Target isosurface value $\tilde{\phi} = 0.5$ [S1].

**Outputs:**
- PLIC plane defined by unit normal $\mathbf{n}$ and constant $C$ such that $\mathbf{n} \cdot \mathbf{x} + C = 0$ [S1].
- The plane splits cell volume $V_\Omega$ into sub-volumes $F(t_n)V_\Omega$ and $(1-F(t_n))V_\Omega$ [S1].

**Data Structures:**
- **Interpolated Scalar Field:** $\phi_{ip}$ for each vertex $ip$ of the computational domain [S1].
- **Triangulated Surface:** A set of triangles forming the local or extended interface representation. Each triangle is defined by three vertices [S1].
- **Normal Vectors:** Unit vectors $\mathbf{n}_t$ for each triangular facet $t$ [S1].

**Steps:**

1.  **Interpolate Scalar Field:** For each vertex $ip$, compute $\phi_{ip}$ using weighted averaging of fluid volume fractions from all cells $l$ containing $ip$ [S1]:
    $$ \phi_{ip} = \frac{\sum_l F_l(t_n) w_l}{\sum_l w_l} $$
    where $w_l = 1 / |\mathbf{x}_{ip} - \mathbf{x}_l|$, $\mathbf{x}_{ip}$ is the position of vertex $ip$, and $\mathbf{x}_l$ is the geometric center of cell $l$ [S1].

2.  **Check for Interface:** Determine if the cell contains an interface by verifying if $\phi_{min} < 0.5 < \phi_{max}$, where $\phi_{min}$ and $\phi_{max}$ are the minimum and maximum interpolated scalar values at the cell's vertices [S1].

3.  **Extract Isosurface:** If the condition in Step 2 is met, extract the isosurface at $\tilde{\phi} = 0.5$ using an isosurface extraction algorithm (e.g., Algorithm 1 from [S1]) [S1].

4.  **Validate Isosurface:** If the extracted isosurface consists of a single isopolygon, mark the cell as a "valid isosurface cell" [S1].

5.  **Construct Local Triangulated Surface:** For valid isosurface cells, construct a local triangulated surface by connecting the geometric center of the extracted isosurface to two consecutive $\tilde{\phi}$-vertices for each triangle [S1].

6.  **Compute Local Normal:** Calculate the local normal vector $\mathbf{n}_l$ using weighted averaging over the $N_t$ facets of the triangulated surface [S1]:
    $$ \mathbf{n}_l = \frac{\sum_{t=1}^{N_t} w_t \mathbf{n}_t}{\left| \sum_{t=1}^{N_t} w_t \mathbf{n}_t \right|} $$
    where $\mathbf{n}_t$ is the unit normal to facet $t$, and $w_t$ is a weighting factor [S1].
    - For cubic grids, $w_t = \frac{\sin(\theta)}{L_1 L_2}$, where $\theta$ is the angle between the two inner edges of the triangular facet, and $L_1, L_2$ are their lengths [S1].
    - For other grids, $w_t = 1$ [S1].

7.  **Construct Extended Triangulated Surface:** Create an extended triangulated surface by connecting the geometric center of the current surface to the geometric centers of isosurfaces extracted at adjacent cells [S1].

8.  **Compute Extended Normal:** Compute the extended normal vector $\mathbf{n}_e$ using the same weighted averaging formula as in Step 6, but applied to the extended surface [S1].

9.  **Select Primary Normal:**
    - If $\arccos(\mathbf{n}_e \cdot \mathbf{n}_l) < 1.2$ radians, set $\mathbf{n} = \mathbf{n}_e$ [S1].
    - Otherwise, set $\mathbf{n} = \mathbf{n}_l$ [S1].

10. **Fallback for Invalid Cells:** If the cell is not a valid isosurface cell, compute $\mathbf{n}$ using a Least-Squares Gradient Interface Reconstruction (LSGIR) method [S1].

11. **Compute PLIC Constant:** Compute the constant $C$ to locate the PLIC such that it splits the cell volume correctly [S1].
    - For arbitrary grids, use the CIBRAVE method [S1].
    - For cubic grids, use the analytical method of Scardovelli and Zaleski [S1].

12. **Update Vertices:** Move the vertices of the extended triangulated surfaces to the corresponding PLIC centers [S1].

13. **Compute Corrected Normal:** For valid isosurface cells, compute a corrected normal $\mathbf{n}_c$ from the updated extended triangulated surface using the formula from Step 6 [S1].

14. **Final Normal Update:**
    - If $\arccos(\mathbf{n}_c \cdot \mathbf{n}) < 1.2$ radians, update $\mathbf{n} = \mathbf{n}_c$ and recompute $C$ to relocate the PLIC [S1].
    - Otherwise, keep $\mathbf{n}$ unchanged [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_clcir_reconstruction(cell, phi_field, adjacent_cells):
    """
    Reconstructs PLIC plane for a given cell using Extended CLCIR method.
    
    Args:
        cell: Object containing cell geometry (vertices, faces, volume, center)
        phi_field: Dictionary mapping vertex_index -> interpolated scalar value
        adjacent_cells: List of adjacent cell objects
        
    Returns:
        n: Unit normal vector
        C: PLIC constant
    """
    # Step 1: Interpolation is assumed pre-computed in phi_field
    
    # Step 2: Check for interface
    phi_min = min(phi_field[v] for v in cell.vertices)
    phi_max = max(phi_field[v] for v in cell.vertices)
    
    if not (phi_min < 0.5 < phi_max):
        # Fallback to LSGIR if no interface or invalid topology
        n = lsgir_reconstruction(cell, phi_field)
        C = compute_plic_constant(cell, n, cell.volume_fraction)
        return n, C

    # Step 3: Extract isosurface at 0.5
    isosurface = extract_isosurface(cell, phi_field, target=0.5)
    
    # Step 4: Validate isosurface
    is_valid = is_single_isopolygon(isosurface)
    
    if is_valid:
        # Step 5: Construct local triangulated surface
        local_surface = construct_local_triangulation(isosurface, cell)
        
        # Step 6: Compute local normal
        n_l = compute_weighted_normal(local_surface, cell.grid_type)
        
        # Step 7: Construct extended triangulated surface
        extended_surface = construct_extended_triangulation(local_surface, adjacent_cells)
        
        # Step 8: Compute extended normal
        n_e = compute_weighted_normal(extended_surface, cell.grid_type)
        
        # Step 9: Select primary normal
        angle = acos(dot(n_e, n_l))
        if angle < 1.2:
            n = n_e
        else:
            n = n_l
            
        # Step 11: Compute PLIC constant
        C = compute_plic_constant(cell, n, cell.volume_fraction)
        
        # Step 12: Update vertices to PLIC centers
        update_surface_vertices(extended_surface, n, C)
        
        # Step 13: Compute corrected normal
        n_c = compute_weighted_normal(extended_surface, cell.grid_type)
        
        # Step 14: Final normal update
        angle_c = acos(dot(n_c, n))
        if angle_c < 1.2:
            n = n_c
            C = compute_plic_constant(cell, n, cell.volume_fraction)
            
    else:
        # Fallback to LSGIR for invalid isosurface cells
        n = lsgir_reconstruction(cell, phi_field)
        C = compute_plic_constant(cell, n, cell.volume_fraction)
        
    return n, C

def compute_weighted_normal(surface, grid_type):
    """
    Computes weighted normal vector from triangulated surface.
    """
    weighted_sum = vector_zero()
    for triangle in surface.triangles:
        n_t = triangle.normal()
        if grid_type == "cubic":
            # w_t = sin(angle) / (length1 * length2)
            angle = angle_between_inner_edges(triangle)
            l1, l2 = lengths_of_inner_edges(triangle)
            w_t = sin(angle) / (l1 * l2)
        else:
            w_t = 1.0
        weighted_sum += w_t * n_t
        
    norm = length(weighted_sum)
    if norm == 0:
        return vector_zero()
    return weighted_sum / norm

def compute_plic_constant(cell, n, volume_fraction):
    """
    Computes C such that plane splits cell volume according to volume_fraction.
    """
    if cell.grid_type == "cubic":
        # Use Scardovelli and Zaleski analytical method
        return scardovelli_zaleski_cubic(cell, n, volume_fraction)
    else:
        # Use CIBRAVE method
        return cibrave_method(cell, n, volume_fraction)
```

## Subtopics

### [VOF Advection with Isosurface Extraction](1063-vof-advection-with-isosurface-extraction.md)
This method simulates multiphase fluid flows by advecting a Volume of Fluid (VOF) indicator function on arbitrary polyhedral grids. It solves the advection equation geometrically using Edge-Matched Flux Polygon Advection (EMFPA) and reconstructs the fluid interface using the CLCIR method, which relies on isosurface extraction to determine the Piecewise Linear Interface Concept (PLIC) plane orientation. It is distinguished from standard VOF methods by its ability to handle non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by its explicit geometric flux calculation.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Handles arbitrary polyhedral grids, including non-convex and self-intersecting cells, without decomposition [S1]. Provides accurate PLIC reconstruction via isosurface extraction and extended triangulation [S1].
- **Limitations:** Requires robust isosurface extraction algorithm [S1]. Fallback to LSGIR may be less accurate in low-resolution regions [S1].
- **When to Use:** VOF simulations on complex, unstructured polyhedral grids where interface topology is complex [S1].
- **When Not to Use:** Simple cubic grids where analytical methods are sufficient and faster [S1].

## Background
The Extended CLCIR method was introduced in a 2021 paper by López et al. [S1]. It is used in Volume of Fluid (VOF) simulations to reconstruct fluid interfaces on arbitrary polyhedral grids [S1]. The method combines isosurface extraction techniques with PLIC reconstruction to improve accuracy in complex geometries [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
