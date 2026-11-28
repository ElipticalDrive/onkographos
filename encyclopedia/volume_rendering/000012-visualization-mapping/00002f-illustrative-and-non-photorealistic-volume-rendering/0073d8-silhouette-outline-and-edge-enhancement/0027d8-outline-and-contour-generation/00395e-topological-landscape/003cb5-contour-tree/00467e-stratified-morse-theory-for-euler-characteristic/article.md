# Stratified Morse Theory for Euler Characteristic

Also called: MMS-SMT, Method of Manufactured Solutions (MMS) using Stratified Morse Theory, SMT Verification, Stratified Morse Theory (SMT) Verification

## Overview
Stratified Morse Theory (SMT) for Euler Characteristic is a mathematical framework and verification pipeline used to compute the Euler characteristic ($\chi$) of isosurfaces generated from piecewise trilinear scalar fields, specifically handling non-smoothness at cell boundaries and surfaces with boundary components [S1, unofficial]. It solves the problem of verifying the topological correctness of isosurface extraction algorithms (such as Marching Cubes) by comparing the Euler characteristic derived from critical point analysis against the Euler characteristic of the extracted mesh [S1, unofficial]. Unlike its siblings, which often rely on smooth manifold assumptions or digital topology constraints, SMT explicitly stratifies the grid into vertices, edges, faces, and cells to apply Morse theory locally, making it robust for verifying algorithms that guarantee topological correctness in the presence of ambiguous configurations [S1, unofficial].

## How it works
The method operates as a verification pipeline, typically within a Method of Manufactured Solutions (MMS) context, where random scalar fields are generated, and the topological invariants of the resulting isosurfaces are computed via two independent methods: critical point analysis (SMT) and mesh analysis [S1, unofficial].

### Inputs and Outputs
*   **Input:** A rectilinear grid $G$ of size $n \times n \times n$ with scalar values sampled in the interval $[-1, 1]$ [S1, unofficial].
*   **Output:** A boolean verification result (Pass/Fail) indicating whether the Euler characteristic computed from critical points matches the Euler characteristic computed from the extracted mesh [S1, unofficial].

### Data Structures
1.  **Stratified Grid $M$:** The grid is partitioned into four disjoint strata:
    *   **Vertex Set ($V$):** All grid vertices [S1, unofficial].
    *   **Edge Set ($E$):** Interiors of all grid edges [S1, unofficial].
    *   **Face Set ($F$):** Interiors of all grid faces [S1, unofficial].
    *   **Cell Set ($C$):** Interiors of all grid cells [S1, unofficial].
2.  **Critical Points ($CPs$):** A list of points $p$ where the gradient is zero within their respective stratum, plus all points in the Vertex Set (which are considered critical by definition in SMT) [S1, unofficial].

### Numbered Steps
1.  **Stratify the Grid:** Partition the grid $G$ into the four sets defined above: Vertex, Edge, Face, and Cell [S1, unofficial].
2.  **Identify Critical Points:**
    *   For each stratum (Edge, Face, Cell), find points where the gradient of the trilinear interpolant restricted to that stratum is zero [S1, unofficial].
    *   Include all vertices in the Vertex Set as critical points [S1, unofficial].
3.  **Check for Degeneracy:** If any critical point $p$ is degenerate or is an internal saddle close to edges or faces, discard the current grid sample and restart from Step 1 to avoid numerical instabilities [S1, unofficial].
4.  **Extract Isosurface:** Extract the isosurface mesh $K$ from $G$ using the isosurfacing technique under verification, typically at an isovalue $\alpha$ that is not a critical value (e.g., $\alpha=0$) [S1, unofficial].
5.  **Compute SMT Euler Characteristic ($\chi_v$):**
    *   For each critical point $p$, determine its stratum (Vertex, Edge, Face, or Cell) [S1, unofficial].
    *   Compute the lower link $L^-(p)$ and upper link $L^+(p)$ Euler characteristics [S1, unofficial].
    *   Calculate the change $\Delta\chi(p) = \chi(L^+(p)) - \chi(L^-(p))$ [S1, unofficial].
    *   Sum the changes: $\chi_v = \sum_{p \in CPs} \Delta\chi(p)$ [S1, unofficial].
6.  **Compute Mesh Euler Characteristic ($\chi_k$):**
    *   Extract the triangle mesh $K$ from the isosurfacing algorithm [S1, unofficial].
    *   Compute $\chi_k = V(K) - E(K) + F(K)$, where $V, E, F$ are the counts of vertices, edges, and triangles in $K$ [S1, unofficial].
7.  **Compare:** If $\chi_v \neq \chi_k$, the verification fails [S1, unofficial].

### Formulas
The core of SMT is computing $\chi(L^-(p))$ and $\chi(L^+(p))$ for critical points in non-smooth strata [S1, unofficial].

**General Recursive Formula:**
For a critical point $p$ in a $d$-dimensional stratum, let $T_{L^-}(p)$ be the lower tangential link and $N_{L^-}(p)$ be the lower normal link [S1, unofficial]. The Euler characteristic of the lower link is:
$$ \chi(L^-(p)) = \chi(N_{L^-}(p)) + \chi(T_{L^-}(p)) - \chi(N_{L^-}(p))\chi(T_{L^-}(p)) $$
This formula is applied recursively. If $T_{L^-}(p)$ or $T_{L^+}(p)$ are not 1-disks, the equation is applied to lower-dimensional strata until 1-disks are reached, where $\chi$ is simply the cardinality of the set (0, 1, or 2 points) [S1, unofficial].

**Vertex Critical Points:**
For a critical point $p$ located at a vertex, let $nl_i$ be the number of adjacent vertices in Cartesian direction $i$ with scalar values less than $p$'s value [S1, unofficial].
$$ \chi(L^-(p)) = nl_1 + nl_2 + nl_3 - nl_1(nl_2 + nl_3) $$
$\chi(L^+(p))$ is computed similarly using neighbors with higher values [S1, unofficial].

**Face Critical Points:**
Critical points on faces are necessarily saddles [S1, unofficial]. Let $r$ be the face containing $p$, and $r^\perp$ be the line segment orthogonal to $r$ passing through $p$ [S1, unofficial]. Let $nl$ be the number of ends of $r^\perp$ with scalar values less than $p$, and $nu$ be the number of ends with values higher than $p$ [S1, unofficial].
$$ \chi(L^-(p)) = nl + 2 - 2nl $$
$$ \chi(L^+(p)) = nu + 2 - 2nu $$
Solutions for face critical points are found by setting coordinates to 0 or 1 and solving the linear equation; solutions outside the unit cube are ignored [S1, unofficial].

**Cell Critical Points:**
For critical points in the interior of cubic cells, standard smooth Morse theory applies, as the normal Morse data is 0-dimensional [S1, unofficial].

**Total Euler Characteristic:**
$$ \chi_\alpha = \sum_{p \in C_\alpha} \Delta\chi(p) $$
where $C_\alpha$ is the set of critical points $p_i$ such that $f(p_i) = \alpha$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def verify_isosurface_topology(grid, isosurfacing_func, num_tests):
    """
    Verifies an isosurfacing algorithm using Stratified Morse Theory.
    """
    for i in range(num_tests):
        # 1. Randomly sample the grid with values in [-1, 1]
        scalar_field = {v: random.uniform(-1.0, 1.0) for v in grid.vertices}
        
        # 2. Compute Critical Points (CPs)
        cps = []
        for v in grid.vertices:
            cps.append({'point': v, 'stratum': 'vertex'})
        for e in grid.edges:
            cps.extend(find_stratum_critical_points(e, 'edge', scalar_field))
        for f in grid.faces:
            cps.extend(find_stratum_critical_points(f, 'face', scalar_field))
        for c in grid.cells:
            cps.extend(find_stratum_critical_points(c, 'cell', scalar_field))
            
        # 3. Check for degeneracy
        if any(is_degenerate(p) or is_internal_saddle_close_to_boundary(p) for p in cps):
            continue # Restart test
            
        # 4. Extract Isosurface
        mesh_k = isosurfacing_func(scalar_field, isovalue=0.0)
        
        # 5. Compute Euler Characteristic from Critical Points (SMT)
        chi_v = 0.0
        for p in cps:
            stratum = p['stratum']
            if stratum == 'vertex':
                chi_v += compute_vertex_delta_chi(p, scalar_field)
            elif stratum == 'face':
                chi_v += compute_face_delta_chi(p, scalar_field)
            elif stratum == 'cell':
                chi_v += compute_cell_delta_chi(p, scalar_field)
            elif stratum == 'edge':
                chi_v += compute_edge_delta_chi(p, scalar_field)
                
        # 6. Compute Euler Characteristic from Mesh
        v_count = len(mesh_k.vertices)
        e_count = len(mesh_k.edges)
        f_count = len(mesh_k.triangles)
        chi_k = v_count - e_count + f_count
        
        # 7. Compare
        if abs(chi_v - chi_k) > 1e-9:
            return False
            
    return True

# [S1, unofficial] Sources do not provide explicit code for solving gradient=0 on strata.
def find_stratum_critical_points(stratum, stratum_type, scalar_field):
    return []

# [S1, unofficial] Sources do not provide explicit code for degeneracy checks.
def is_degenerate(p):
    return False

def is_internal_saddle_close_to_boundary(p):
    return False

def compute_vertex_delta_chi(p, scalar_field):
    # [S1, unofficial] nl_i: number of adjacent vertices in direction i with value < p.value
    # [S1, unofficial] Sources do not provide explicit code for counting adjacent vertices.
    nl1, nl2, nl3 = 0, 0, 0
    chi_L_minus = nl1 + nl2 + nl3 - nl1 * (nl2 + nl3)
    
    nu1, nu2, nu3 = 0, 0, 0
    chi_L_plus = nu1 + nu2 + nu3 - nu1 * (nu2 + nu3)
    
    return chi_L_plus - chi_L_minus

def compute_face_delta_chi(p, scalar_field):
    # [S1, unofficial] r_perp: line segment orthogonal to face through p
    # [S1, unofficial] Sources do not provide explicit code for segment operations.
    r_perp = []
    nl = 0
    nu = 0
    
    chi_L_minus = nl + 2 - 2 * nl
    chi_L_plus = nu + 2 - 2 * nu
    
    return chi_L_plus - chi_L_minus

def compute_cell_delta_chi(p, scalar_field):
    # [S1, unofficial] For cell interiors, normal Morse data is 0-dimensional.
    # Apply standard smooth Morse theory based on Hessian eigenvalues.
    # [S1, unofficial] Sources do not provide explicit code for Hessian computation.
    index = 0
    
    if index == 0: # Minimum
        return 2 - 0
    elif index == 1: # Saddle-1
        return 0 - 2
    elif index == 2: # Saddle-2
        return 2 - 0
    else: # Maximum
        return 0 - 2

def compute_edge_delta_chi(p, scalar_field):
    # [S1, unofficial] Apply general recursive formula for d=1 stratum
    # [S1, unofficial] Sources do not provide explicit code for link computation.
    chi_L_minus = 0
    chi_L_plus = 0
    
    return chi_L_plus - chi_L_minus
```

### Parameters
*   **Grid Size ($n \times n \times n$):** The resolution of the rectilinear grid [S1, unofficial].
*   **Scalar Range:** Values are sampled in $[-1, 1]$ [S1, unofficial].
*   **Isovalue:** Typically 0, chosen to avoid critical values [S1, unofficial].
*   **#tests:** A large number of iterations to statistically verify the implementation [S1, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Critical Points:** If a critical point is degenerate or an internal saddle is too close to an edge or face, the test is restarted to avoid numerical instability [S1, unofficial].
*   **Masking Errors:** SMT may fail to detect topological errors if the algorithm introduces features that preserve the Euler characteristic. For example, a torus ($\chi=0$) plus three "cycling" triangles (each sharing two vertices but no edges, also $\chi=0$) results in a total $\chi=0$, masking the error [S1, unofficial].
*   **Face Critical Point Detection:** Critical points on faces are found by solving linear equations with coordinates set to 0 or 1; solutions outside the unit cube $[0, 1]^3$ must be ignored [S1, unofficial].

## Subtopics

### [Total Euler Characteristic (TEC)](12667-total-euler-characteristic-tec.md)
Total Euler Characteristic (TEC) is a noise level estimation metric for direct volume rendering transfer function design, defined as the L1-norm of the evolution curve of the Euler Characteristic across all possible isovalues. It solves the problem of distinguishing true topological structures (islands and holes) from noise-induced speckles in rendered images, offering a more accurate noise assessment than gradient-based measures like Total Variation (TV) or Total Second Derivatives (TSD). Unlike its siblings, TEC approximates zero for vanishing noise even in scenes with significant true gradients and curvatures, making it superior for automated transfer function optimization.

## Operating characteristics
*   **Time Complexity:** The method requires iterating over a large number of random grid samples (#tests) [S1, unofficial]. For each sample, it involves finding critical points (analytically solvable for trilinear functions), extracting the mesh, and computing topological invariants [S1, unofficial]. The complexity is dominated by the number of tests and the cost of isosurface extraction [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Handles surfaces with boundary components, which is a limitation for some other topological verification methods. It is effective when used in unison with Digital Surface (DS) tests [S1, unofficial].
*   **Limitations:** Less sensitive than Digital Surface (DS) tests because it only provides the Euler characteristic, making it harder to identify the specific source of topological failure [S1, unofficial]. It can be fooled by algorithms that preserve $\chi$ while introducing spurious features [S1, unofficial].
*   **When to Use:** When verifying isosurface extraction algorithms that promise topological correctness for piecewise trilinear interpolants, especially in the presence of ambiguous configurations [S1, unofficial].
*   **When Not to Use:** When detailed topological information (Betti numbers) is required, or when debugging specific topological errors, as SMT only provides a scalar invariant [S1, unofficial].

## Background
Stratified Morse Theory for Euler Characteristic was introduced by Etiene et al. as part of a verification pipeline for isosurface extraction algorithms [S1, unofficial]. It extends classical Morse theory to handle the non-smooth nature of piecewise trilinear interpolants used in volume rendering [S1, unofficial]. The method was developed to address the limitations of smooth Morse theory, which cannot be directly applied to functions with discontinuities in derivatives at cell boundaries [S1, unofficial]. It forms the core of the MMS-SMT verification pipeline [S1, unofficial].

## Sources
[S1] Topology Verification for Isosurface Extraction Topology Verification for Isosurface Extraction, https://hal.science/hal-01211133, fetched 2026-10-26 (unofficial)
