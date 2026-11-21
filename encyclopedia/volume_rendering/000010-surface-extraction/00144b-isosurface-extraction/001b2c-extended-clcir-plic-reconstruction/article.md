# Extended CLCIR PLIC Reconstruction

Also called: Extended CLCIR method, PLIC reconstruction on arbitrary grids

## Overview
The Extended CLCIR method reconstructs a Piecewise Linear Interface Concept (PLIC) plane for Volume of Fluid (VOF) simulations on arbitrary polyhedral grids by extracting an isosurface from an interpolated scalar field and computing a weighted normal vector [S1]. It distinguishes itself from standard VOF methods by handling non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by explicitly calculating geometric fluxes for advection [S1].

## How it works
**Inputs:**
- Scalar field $\phi$ at cell vertices, interpolated from fluid volume fractions $F_l(t_n)$ of neighboring cells [S1].
- Cell geometry (vertices, faces, volume $V_\Omega$, geometric center) for arbitrary polyhedral cells [S1].
- Target isosurface value $\tilde{\phi} = 0.5$ [S1].

**Outputs:**
- PLIC plane defined by unit normal $\mathbf{n}$ and constant $C$ such that $\mathbf{n} \cdot \mathbf{x} + C = 0$ [S1].
- The plane splits cell volume $V_\Omega$ into sub-volumes $F(t_n)V_\Omega$ and $(1-F(t_n))V_\Omega$ [S1].

**Data Structures:**
- **Interpolated Scalar Field:** $\phi_{ip}$ for each vertex $ip$ of the computational domain [S1].
- **Triangulated Surface:** A set of triangles forming the local or extended interface representation, where each triangle is defined by three vertices [S1].
- **Normal Vectors:** Unit vectors $\mathbf{n}_t$ for each triangular facet $t$ [S1].

**Steps:**

1.  **Interpolate Scalar Field:** For each vertex $ip$, compute $\phi_{ip}$ using weighted averaging of fluid volume fractions from all cells $l$ containing $ip$ [S1]:
    $$ \phi_{ip} = \frac{\sum_l F_l(t_n) w_l}{\sum_l w_l} $$
    where $w_l = 1 / \|\mathbf{x}_{ip} - \mathbf{x}_l\|$, $\mathbf{x}_{ip}$ is the position of vertex $ip$, and $\mathbf{x}_l$ is the geometric center of cell $l$ [S1].

2.  **Check for Interface:** Determine if the cell contains an interface by verifying if $\phi_{min} < 0.5 < \phi_{max}$, where $\phi_{min}$ and $\phi_{max}$ are the minimum and maximum interpolated scalar values at the cell's vertices [S1].

3.  **Extract Isosurface:** If the condition in Step 2 is met, extract the isosurface at $\tilde{\phi} = 0.5$ using the proposed isosurface extraction algorithm (Algorithm 1) [S1].

4.  **Validate Isosurface:** If the extracted isosurface consists of a single isopolygon, mark the cell as a "valid isosurface cell" [S1].

5.  **Construct Local Triangulated Surface:** For valid isosurface cells, construct a local triangulated surface by connecting the geometric center of the extracted isosurface to two consecutive $\tilde{\phi}$-vertices for each triangle [S1].

6.  **Compute Local Normal:** Calculate the local normal vector $\mathbf{n}_l$ using weighted averaging over the $N_t$ facets of the triangulated surface [S1]:
    $$ \mathbf{n}_l = \frac{\sum_{t=1}^{N_t} w_t \mathbf{n}_t}{\left\| \sum_{t=1}^{N_t} w_t \mathbf{n}_t \right\|} $$
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

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b): return sum(x*y for x,y in zip(a,b))
def norm(a): return math.sqrt(sum(x*x for x in a))
def normalize(a):
    n = norm(a)
    return [x/n for x in a] if n > 1e-12 else [0.0, 0.0, 0.0]

def extended_clcir_reconstruction(cell, phi_field, adjacent_cells):
    # Step 1: Scalar field phi_ip is precomputed per [S1]
    # Step 2: Check for interface
    phi_vals = [phi_field[v] for v in cell.vertices]
    phi_min, phi_max = min(phi_vals), max(phi_vals)

    if not (phi_min < 0.5 < phi_max):
        # Step 10: Fallback to LSGIR for non-interfacial cells [S1]
        n = [0.0, 0.0, 0.0] # LSGIR result [S1]
        C = compute_plic_constant(cell, n, cell.volume_fraction)
        return n, C

    # Step 3: Extract isosurface at 0.5 [S1]
    # Note: Algorithm 1 from [S1] is not detailed in the provided sources.
    isosurface = [] # Represents extracted isopolygon(s)

    # Step 4: Validate isosurface
    is_valid = len(isosurface) == 1

    if is_valid:
        # Step 5: Construct local triangulated surface [S1]
        center = cell.center
        polygon = isosurface[0]
        local_triangles = []
        for i in range(len(polygon)):
            v1, v2 = polygon[i], polygon[(i + 1) % len(polygon)]
            local_triangles.append([center, v1, v2])

        # Step 6: Compute local normal [S1]
        weighted_sum = [0.0, 0.0, 0.0]
        for tri in local_triangles:
            n_t = normalize([dot(tri[1]-tri[0], tri[2]-tri[0])]) # Simplified normal calc
            if cell.grid_type == "cubic":
                e1, e2 = tri[1]-center, tri[2]-center
                cos_theta = max(-1.0, min(1.0, dot(e1, e2) / (norm(e1) * norm(e2))))
                w_t = math.sin(math.acos(cos_theta)) / (norm(e1) * norm(e2))
            else:
                w_t = 1.0
            weighted_sum = [weighted_sum[i] + w_t * n_t[i] for i in range(3)]
        n_l = normalize(weighted_sum)

        # Step 7: Construct extended triangulated surface [S1]
        extended_triangles = list(local_triangles)
        for adj in adjacent_cells:
            adj_center = adj.center
            for tri in local_triangles:
                extended_triangles.append([center, tri[0], adj_center])

        # Step 8: Compute extended normal [S1]
        weighted_sum_e = [0.0, 0.0, 0.0]
        for tri in extended_triangles:
            n_t = normalize([dot(tri[1]-tri[0], tri[2]-tri[0])])
            if cell.grid_type == "cubic":
                e1, e2 = tri[1]-center, tri[2]-center
                cos_theta = max(-1.0, min(1.0, dot(e1, e2) / (norm(e1) * norm(e2))))
                w_t = math.sin(math.acos(cos_theta)) / (norm(e1) * norm(e2))
            else:
                w_t = 1.0
            weighted_sum_e = [weighted_sum_e[i] + w_t * n_t[i] for i in range(3)]
        n_e = normalize(weighted_sum_e)

        # Step 9: Select primary normal [S1]
        cos_angle = max(-1.0, min(1.0, dot(n_e, n_l)))
        if math.acos(cos_angle) < 1.2:
            n = n_e
        else:
            n = n_l

        # Step 11: Compute PLIC constant [S1]
        C = compute_plic_constant(cell, n, cell.volume_fraction)

        # Step 12: Update vertices to PLIC centers [S1]
        for tri in extended_triangles:
            for idx in range(3):
                d = dot(n, tri[idx]) + C
                tri[idx] = [tri[idx][i] - d * n[i] for i in range(3)]

        # Step 13: Compute corrected normal [S1]
        weighted_sum_c = [0.0, 0.0, 0.0]
        for tri in extended_triangles:
            n_t = normalize([dot(tri[1]-tri[0], tri[2]-tri[0])])
            if cell.grid_type == "cubic":
                e1, e2 = tri[1]-center, tri[2]-center
                cos_theta = max(-1.0, min(1.0, dot(e1, e2) / (norm(e1) * norm(e2))))
                w_t = math.sin(math.acos(cos_theta)) / (norm(e1) * norm(e2))
            else:
                w_t = 1.0
            weighted_sum_c = [weighted_sum_c[i] + w_t * n_t[i] for i in range(3)]
        n_c = normalize(weighted_sum_c)

        # Step 14: Final normal update [S1]
        cos_angle_c = max(-1.0, min(1.0, dot(n_c, n)))
        if math.acos(cos_angle_c) < 1.2:
            n = n_c
            C = compute_plic_constant(cell, n, cell.volume_fraction)
    else:
        # Fallback to LSGIR for invalid isosurface cells [S1]
        n = [0.0, 0.0, 0.0]
        C = compute_plic_constant(cell, n, cell.volume_fraction)

    return n, C

def compute_plic_constant(cell, n, volume_fraction):
    # Step 11: Compute C to split volume [S1]
    if cell.grid_type == "cubic":
        # Scardovelli and Zaleski analytical method [S1]
        pass
    else:
        # CIBRAVE method [S1]
        pass
    return 0.0 # Placeholder for actual volume enforcement calculation [S1]
```

## Subtopics

### [VOF Advection with Isosurface Extraction](1063-vof-advection-with-isosurface-extraction.md)
This method simulates multiphase fluid flows by advecting a Volume of Fluid (VOF) indicator function on arbitrary polyhedral grids. It solves the advection equation geometrically using Edge-Matched Flux Polygon Advection (EMFPA) and reconstructs the fluid interface using the CLCIR method, which relies on isosurface extraction to determine the Piecewise Linear Interface Concept (PLIC) plane orientation. It is distinguished from standard VOF methods by its ability to handle non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by its explicit geometric flux calculation.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1].
- **Memory Complexity:** Not given in the sources [S1].
- **Measured Performance:** Not given in the sources [S1].
- **Strengths:** Handles arbitrary polyhedral grids, including non-convex and self-intersecting cells, without decomposition [S1]. Provides accurate PLIC reconstruction via isosurface extraction and extended triangulation [S1].
- **Limitations:** Requires robust isosurface extraction algorithm [S1]. Fallback to LSGIR may be less accurate in low-resolution regions [S1].
- **When to Use:** VOF simulations on complex, unstructured polyhedral grids where interface topology is complex [S1].
- **When Not to Use:** Simple cubic grids where analytical methods are sufficient and faster [S1].

## Background
The Extended CLCIR method was introduced in a 2021 paper by López et al. [S1]. It is used in Volume of Fluid (VOF) simulations to reconstruct fluid interfaces on arbitrary polyhedral grids [S1]. The method combines isosurface extraction techniques with PLIC reconstruction to improve accuracy in complex geometries [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
