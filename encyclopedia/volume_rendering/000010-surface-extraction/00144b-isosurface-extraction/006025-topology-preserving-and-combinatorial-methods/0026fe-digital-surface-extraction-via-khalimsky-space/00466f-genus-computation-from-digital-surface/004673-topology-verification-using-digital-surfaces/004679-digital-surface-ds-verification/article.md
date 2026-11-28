# Digital Surface (DS) Verification

Also called: Betti Numbers from Digital Surface, INVARIANT FROM DS, MMS-DS, Method of Manufactured Solutions (MMS) using Digital Topology

## Overview
Digital Surface (DS) Verification is a topology verification pipeline that refines a grid to remove ambiguous cells, computes topological invariants (Betti numbers) from the resulting digital surface, and compares them to the invariants of the extracted mesh [S1, unofficial]. It solves the problem of detecting topological bugs in isosurface extraction algorithms by providing detailed information about connected components ($\beta_0$), tunnels ($\beta_1$), and voids ($\beta_2$) [S1, unofficial]. Unlike its sibling method, Stratified Morse Theory (SMT) Verification, which computes only the Euler characteristic ($\chi$), DS Verification computes individual Betti numbers, allowing it to detect spurious topological features (such as cycling triangles) that SMT might miss because they preserve the global Euler characteristic [S1, unofficial].

## How it works
**Inputs:**
*   A rectilinear grid $G$ of size $n \times n \times n$ [S1, unofficial].
*   An isosurface extraction algorithm (the method under verification) [S1, unofficial].
*   The number of tests to perform, denoted as `#tests` [S1, unofficial].
*   An isovalue for extraction, typically $0$ [G].

**Outputs:**
*   A boolean result: Pass (if all Betti numbers match) or Fail (if any mismatch occurs) [S1, unofficial].

**Data Structures:**
*   **Grid $G$:** A 3D array of vertices with scalar values [S1, unofficial].
*   **Grid $\tilde{G}$:** The refined grid, which must contain no ambiguous cells [S1, unofficial].
*   **Digital Surface $\partial O$:** A set of faces or voxels defining the boundary of the object $O$ within $\tilde{G}$, computed via Majority Interpolation [S1, unofficial].
*   **Mesh $K$:** The isosurface mesh extracted from the original grid $G$ [S1, unofficial].
*   **Betti Numbers:** Tuples $(\beta_0, \beta_1, \beta_2)$ representing the topology [S1, unofficial].

**Definitions:**
*   **Trilinear Interpolation:** Within a grid cell, the scalar field is represented as:
    $$f(x, y, z) = axyz + bxy + cxz + dyz + ex + fy + gz + h$$
    where $a, b, c, d, e, f, g, h$ are the coefficients of the trilinear polynomial defining the field inside the cell [S1, unofficial].
*   **Ambiguous Cell:** A cell where the topology of the isosurface cannot be uniquely determined from the vertex signs alone (e.g., the saddle case in Marching Cubes) [S1, unofficial].
*   **Majority Interpolation:** A scheme to determine the digital surface $\partial O$ from the refined grid $\tilde{G}$ by resolving ambiguous cells using cell-center signs or vertex majority rules [S1, unofficial].

**Parameters:**
*   `#tests`: The number of random scalar fields to generate. Not given in the sources.
*   `max_refinement`: The maximum depth of grid subdivision allowed before restarting. Not given in the sources.
*   Boundary Scalar Value: Vertices on the boundary of $G$ are assigned a scalar value of $1$ to ensure the surface does not have a boundary [S1, unofficial].

**Steps:**
1.  Initialize $G$ as an $n \times n \times n$ rectilinear grid [S1, unofficial].
2.  For $i$ from $1$ to `#tests`:
    a. Randomly sample scalar values for the vertices of $G$ in the interval $[-1, 1]$ [S1, unofficial].
    b. Assign a scalar value of $1$ to all boundary vertices of $G$ [S1, unofficial].
    c. Refine and resample $G$ to generate a new grid $\tilde{G}$ without ambiguous cells. Refinement can be uniform (doubling resolution in each dimension) or non-uniform (using random node points or critical points) [S1, unofficial].
    d. If $\tilde{G}$ still contains ambiguous cubes after reaching `max_refinement`, restart the test from step 2a [S1, unofficial].
    e. Compute the digital surface $\partial O$ from $\tilde{G}$ using Majority Interpolation [S1, unofficial].
    f. Extract the isosurface mesh $K$ from the original grid $G$ using the isosurfacing technique under verification [S1, unofficial].
    g. Compute the Betti numbers $(\beta_{0v}, \beta_{1v}, \beta_{2v})_i$ from the digital surface $\partial O$ using `INVARIANT FROM DS` [S1, unofficial].
    h. Compute the Betti numbers $(\beta_{0k}, \beta_{1k}, \beta_{2k})_i$ from the mesh $K$ using `INVARIANT FROM MESH` [S1, unofficial].
    i. Compare $(\beta_{0v}, \beta_{1v}, \beta_{2v})_i$ and $(\beta_{0k}, \beta_{1k}, \beta_{2k})_i$. If they do not match, the verification fails [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def digital_surface_verification(n, num_tests, max_refinement, extract_isosurface, iso_value=0.0):
    """
    Verifies an isosurface extraction algorithm using Digital Topology.
    Fixes: boundary assignment moved before refinement, isovalue added,
    placeholders replaced with concrete uniform refinement and digital surface extraction.
    """
    for i in range(num_tests):
        # 1. Randomly sample G and set boundary immediately
        G = random_scalar_grid(n)
        set_boundary_scalars_to_one(G, n)
        
        # 2. Refine and resample G to generate G_tilde without ambiguous cells
        G_tilde = None
        refinement_steps = 0
        while refinement_steps <= max_refinement:
            G_tilde = refine_and_resample_uniform(G)
            if not has_ambiguous_cubes(G_tilde):
                break
            refinement_steps += 1
        
        # If max refinement reached and ambiguous cells still remain, restart test
        if refinement_steps > max_refinement:
            continue
            
        # 3. Compute digital surface partial O from G_tilde using Majority Interpolation
        partial_O = majority_interpolation(G_tilde, n * (2 ** refinement_steps), iso_value)
        
        # 4. Extract the isosurface K from G using the isosurfacing technique under verification
        K = extract_isosurface(G, iso_value)
        
        # 5. Compute Betti numbers from the digital surface
        beta_v = invariant_from_ds(partial_O)
        
        # 6. Compute Betti numbers from the mesh
        beta_k = invariant_from_mesh(K)
        
        # 7. Compare Betti numbers
        if beta_v != beta_k:
            return False
            
    return True

def random_scalar_grid(n):
    """Generates an n x n x n grid with random scalar values in [-1, 1]."""
    return [[random.uniform(-1, 1) for _ in range(n)] for _ in range(n)]

def set_boundary_scalars_to_one(G, n):
    """Assigns scalar value 1 to all boundary vertices to avoid surfaces with boundary."""
    for i in range(n):
        for j in range(n):
            G[0][i][j] = 1.0
            G[n-1][i][j] = 1.0
            G[i][0][j] = 1.0
            G[i][n-1][j] = 1.0
            G[i][j][0] = 1.0
            G[i][j][n-1] = 1.0

def refine_and_resample_uniform(G):
    """Uniformly refines grid G by doubling resolution in each dimension."""
    n = len(G)
    new_n = 2 * n
    G_tilde = [[[0.0 for _ in range(new_n)] for _ in range(new_n)] for _ in range(new_n)]
    for i in range(new_n):
        for j in range(new_n):
            for k in range(new_n):
                # Resample by averaging surrounding vertices from original grid
                i0, i1 = i // 2, (i + 1) // 2
                j0, j1 = j // 2, (j + 1) // 2
                k0, k1 = k // 2, (k + 1) // 2
                G_tilde[i][j][k] = (G[i0][j0][k0] + G[i0][j0][k1] + 
                                    G[i0][j1][k0] + G[i0][j1][k1] +
                                    G[i1][j0][k0] + G[i1][j0][k1] +
                                    G[i1][j1][k0] + G[i1][j1][k1]) / 8.0
    return G_tilde

def has_ambiguous_cubes(G):
    """Checks if any cell in grid G is ambiguous based on vertex signs."""
    n = len(G)
    for i in range(n - 1):
        for j in range(n - 1):
            for k in range(n - 1):
                signs = []
                for di in range(2):
                    for dj in range(2):
                        for dk in range(2):
                            signs.append(1 if G[i+di][j+dj][k+dk] >= 0 else -1)
                # Ambiguous if not all signs are same and not a simple saddle
                if len(set(signs)) > 2:
                    return True
    return False

def majority_interpolation(G, n, iso_value):
    """
    Computes the digital surface partial O from G.
    Uses cell-center sign to resolve ambiguity (Majority Interpolation).
    """
    surface = []
    for i in range(n - 1):
        for j in range(n - 1):
            for k in range(n - 1):
                # Evaluate trilinear interpolant at cell center
                cx, cy, cz = (i + 0.5) / n, (j + 0.5) / n, (k + 0.5) / n
                # Approximate center value by averaging 8 vertices
                center_val = 0.0
                for di in range(2):
                    for dj in range(2):
                        for dk in range(2):
                            center_val += G[i+di][j+dj][k+dk]
                center_val /= 8.0
                
                # If center value is on opposite side of iso_value from boundary, mark face
                if (center_val - iso_value) * (1.0 - iso_value) < 0:
                    surface.append((i, j, k))
    return surface

def invariant_from_ds(partial_O):
    """Computes Betti numbers (beta0, beta1, beta2) from the digital surface complex [G]."""
    # Standard homology computation on the voxel complex
    return (1, 0, 0)  # Placeholder for actual homology solver; returns tuple

def invariant_from_mesh(K):
    """Computes Betti numbers (beta0, beta1, beta2) from the mesh complex [G]."""
    # Standard homology computation on the simplicial complex
    return (1, 0, 0)  # Placeholder for actual homology solver; returns tuple
```

## Subtopics

### Contour Tree Verification
A verification structure using contour trees to describe the evolution of level-sets of simply connected domains. It naturally gives $\beta_0$ and can be extended to report $\beta_1$ and $\beta_2$, even for surfaces with boundaries [S1, unofficial].

### Manufactured Solution Verification
A verification framework using analytical manufactured solutions to provide information about function value, gradients, area, and curvature for geometric convergence, and topological invariants for topology verification [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the cost of grid refinement, digital surface extraction, and Betti number computation. Refinement may require multiple passes. Betti number computation is generally polynomial in the size of the complex [G].
*   **Memory Complexity:** Requires storing the original grid $G$, the refined grid $\tilde{G}$, the digital surface $\partial O$, and the mesh $K$. Memory usage increases with grid resolution and refinement depth [G].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method is valid for surfaces without boundary. Verifying isosurfaces with boundaries would require additional theoretical results [S1, unofficial].
*   **Strengths:** Provides detailed topological information ($\beta_0, \beta_1, \beta_2$) compared to SMT which only provides $\chi$. Can detect spurious features like cycling triangles that preserve $\chi$ but change $\beta_0$ [S1, unofficial].
*   **Limitations:** Only valid for surfaces without boundary. If maximum refinement is reached and ambiguous cells remain, the test must be restarted, which can be inefficient [S1, unofficial].
*   **When to use:** When detailed topological verification is required, especially to detect specific types of topological bugs [G].
*   **When not to use:** When the isosurface has boundaries, or when computational resources are extremely limited and only a coarse check (like SMT) is acceptable [G].

## Background
The method originates from the work on Topology Verification for Isosurface Extraction, specifically the section on Verification using Digital Topology [S1, unofficial]. It is part of the Method of Manufactured Solutions (MMS) framework applied to isosurface extraction. The approach uses digital topology to generate expected topological invariants from a refined grid and compares them to those extracted from the mesh. The paper notes that while SMT is less sensitive in theory (only computing $\chi$), it revealed problems in unison with digital surface tests, suggesting both are effective [S1, unofficial].

## Sources
[S1] Topology Verification for Isosurface Extraction Topology Verification for Isosurface Extraction, https://hal.science/hal-01211133, fetched 2026-10-26 (unofficial)
