# Topology Verification using Digital Surfaces


## Overview
Topology Verification using Digital Surfaces (DT) is a method for validating the topological correctness of isosurface extraction algorithms by comparing extracted meshes against theoretically expected topological invariants derived from the underlying scalar field [S1, unofficial]. It solves the problem of detecting topological errors (such as incorrect hole counts or connectivity) in generated meshes by leveraging the rigorous mathematical framework of Digital Topology [S1, unofficial]. Unlike Stratified Morse Theory (SMT), which only provides the Euler characteristic $\chi$, DT computes detailed Betti numbers ($\beta_0, \beta_1, \beta_2$), allowing for precise localization of topological failures [S1, unofficial].

## How it works
**Inputs:**
1. A rectilinear grid scalar field $f$ of resolution $N_x \times N_y \times N_z$ (e.g., $5 \times 5 \times 5$) [S1, unofficial].
2. An isovalue $\tau$ for surface extraction [S1, unofficial].
3. An isosurface extraction algorithm $\mathcal{A}$ under test [S1, unofficial].
4. Ground truth topological invariants for the scalar field, computed via Majority Interpolation (MI) or theoretical calculation [S1, unofficial].

**Outputs:**
A boolean result indicating whether the extracted mesh $M$ matches the expected topology, and a report of mismatched Betti numbers if applicable [S1, unofficial].

**Data Structures:**
*   **Scalar Field:** A 3D array of scalar values indexed by $(x, y, z)$ [S1, unofficial].
*   **Mesh $M$:** A set of vertices, edges, and triangles forming a PL-manifold [S1, unofficial].
*   **Digital Surface Complex $\partial O_\alpha$:** A 2-manifold without boundary derived from the scalar field via MI [S1, unofficial].
*   **Betti Numbers:** Integers $\beta_0$ (connected components), $\beta_1$ (loops/handles), and $\beta_2$ (voids/cavities) [S1, unofficial].

**Steps:**
1.  **Generate/Load Test Data:** Create or load random scalar fields on a rectilinear grid [S1, unofficial].
2.  **Extract Mesh:** Run the isosurface extraction algorithm $\mathcal{A}$ on the scalar field with isovalue $\tau$ to produce mesh $M$ [S1, unofficial].
3.  **Compute Mesh Invariants:** Calculate the Betti numbers $\beta_0(M), \beta_1(M), \beta_2(M)$ for the extracted mesh $M$ [S1, unofficial].
4.  **Determine Expected Invariants:** Compute the expected Betti numbers $\beta_0^{exp}, \beta_1^{exp}, \beta_2^{exp}$ for the scalar field using Digital Surface theory based on the MI reconstruction [S1, unofficial].
5.  **Compare:** Check if $\beta_i(M) == \beta_i^{exp}$ for $i \in \{0, 1, 2\}$ [S1, unofficial].
6.  **Report:** If any invariant mismatches, the extraction failed the topology consistency test [S1, unofficial].

**Formulas:**
Betti numbers are computed via simplicial homology using boundary matrices [G]:
$$ \beta_0 = V - \text{rank}(B_2) $$
$$ \beta_1 = \text{rank}(B_1) - \text{rank}(B_2) $$
$$ \beta_2 = F - \text{rank}(B_1) $$
where $V$ is the number of vertices, $F$ is the number of triangles, $B_1$ is the face-to-edge boundary matrix, and $B_2$ is the edge-to-vertex boundary matrix [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def verify_topology(scalar_field, isovalue, extraction_algorithm):
    # Step 1: Extract isosurface mesh
    mesh = extraction_algorithm.run(scalar_field, isovalue)
    if mesh is None or len(mesh.triangles) == 0:
        return False, "Extraction produced no triangles"

    # Step 2: Compute Betti numbers for extracted mesh
    beta_0_m, beta_1_m, beta_2_m = compute_mesh_betti(mesh)

    # Step 3: Compute expected Betti numbers from Digital Surface
    ds_complex = build_digital_surface(scalar_field, isovalue)
    beta_0_e, beta_1_e, beta_2_e = compute_ds_betti(ds_complex)

    # Step 4: Compare invariants
    if beta_0_m != beta_0_e or beta_1_m != beta_1_e or beta_2_m != beta_2_e:
        return False, {
            "beta_0_diff": beta_0_m - beta_0_e,
            "beta_1_diff": beta_1_m - beta_1_e,
            "beta_2_diff": beta_2_m - beta_2_e
        }
    return True, {}

def compute_mesh_betti(mesh):
    # [G] Compute homology via boundary matrices
    V = len(mesh.vertices)
    F = len(mesh.triangles)
    B1 = build_boundary_matrix_faces_to_edges(mesh)
    B2 = build_boundary_matrix_edges_to_vertices(mesh)
    r1 = matrix_rank(B1)
    r2 = matrix_rank(B2)
    beta_0 = V - r2
    beta_1 = r1 - r2
    beta_2 = F - r1
    return beta_0, beta_1, beta_2

def build_digital_surface(field, isovalue):
    ds_cells = []
    for x in range(field.shape[0]):
        for y in range(field.shape[1]):
            for z in range(field.shape[2]):
                val = field[x, y, z]
                if val > isovalue:
                    ds_cells.append(('inside', x, y, z))
                elif val < isovalue:
                    ds_cells.append(('outside', x, y, z))
                else:
                    ds_cells.append(('ambiguous', x, y, z))
    # [G] Resolve ambiguous cells using Majority Interpolation to form a 2-manifold without boundary
    return resolve_ambiguous_cells(ds_cells)

def compute_ds_betti(ds):
    return compute_mesh_betti(ds.to_mesh())
```

**Edge Cases and Pitfalls:**
*   **Boundaries:** The method cannot handle surfaces with boundaries because the underlying Digital Surface theory is only valid for surfaces without boundary [S1, unofficial]. Special treatment for boundaries is not employed [S1, unofficial].
*   **Manifold Requirement:** The Betti numbers $\beta_1$ and $\beta_2$ are computed only if the output mesh is a 2-manifold without boundary [S1, unofficial].
*   **Implementation Bugs:** Incorrect traversal of grid boundaries or misconfiguration of isosurface table entries (e.g., MC configuration 13) can lead to false mismatches [S1, unofficial].
*   **Precision/Units:** The scalar field values and isovalue must share consistent units; mismatched scales cause incorrect inside/outside classification in the DS construction [G].

## Subtopics

### [Digital Surface (DS) Verification](11764-digital-surface-ds-verification.md)
A verification tool based on Digital Topology that generates detailed information about expected topology, including Betti numbers $\beta_0, \beta_1, \beta_2$. It is valid for surfaces without boundary.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** In tests on 1000 randomly generated scalar fields, DT successfully detected topological errors in algorithms like DELISO and SNAP MC [S1, unofficial].
*   **Strengths:** Provides detailed information about specific topological features ($\beta_0, \beta_1, \beta_2$) rather than just the Euler characteristic ($\chi$), making it easier to locate the source of failure [S1, unofficial].
*   **Limitations:** Cannot handle surfaces with boundaries [S1, unofficial]. If an algorithm incorrectly introduces topological features to preserve $\chi$ (as might happen with SMT), DT would still detect the error via Betti numbers, but SMT would fail to detect it [S1, unofficial].

## Background
The method is described in the context of topology consistency testing for isosurface extraction algorithms [S1, unofficial]. It is contrasted with Stratified Morse Theory (SMT) as a more informative verification tool for manifold surfaces [S1, unofficial]. The approach relies on the theoretical framework of Digital Surfaces and Majority Interpolation [S1, unofficial].

## Sources
[S1] Topology Verification for Isosurface Extraction, https://hal.science/hal-01211133, fetched 2026-10-26 (unofficial)
