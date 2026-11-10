# Digital Surface (DS) Verification

Also called: Betti Numbers from Digital Surface, INVARIANT FROM DS, MMS-DS, Method of Manufactured Solutions (MMS) using Digital Topology

## Overview
Digital Surface (DS) Verification is a topology verification pipeline that refines a grid to remove ambiguous cells, computes topological invariants (Betti numbers) from the resulting digital surface, and compares them to the invariants of the extracted mesh [S1, unofficial]. It solves the problem of detecting topological bugs in isosurface extraction algorithms by providing detailed information about connected components ($\beta_0$), tunnels ($\beta_1$), and voids ($\beta_2$) [S1, unofficial]. Unlike its sibling method, Stratified Morse Theory (SMT) Verification, which computes only the Euler characteristic ($\chi$), DS Verification computes individual Betti numbers, allowing it to detect spurious topological features (such as cycling triangles) that SMT might miss because they preserve the global Euler characteristic [S1, unofficial].

## How it works
**Inputs:**
*   A rectilinear grid $G$ of size $n \times n \times n$ [S1, unofficial].
*   An isosurface extraction algorithm (the method under verification) [S1, unofficial].
*   The number of tests to perform, denoted as `#tests` [S1, unofficial].

**Outputs:**
*   A boolean result: Pass (if all Betti numbers match) or Fail (if any mismatch occurs) [S1, unofficial].

**Data Structures:**
*   **Grid $G$:** A 3D array of vertices. Each vertex holds a scalar value.
*   **Grid $\tilde{G}$:** The refined grid, which must contain no ambiguous cells.
*   **Digital Surface $\partial O$:** A set of faces or voxels defining the boundary of the object $O$ within $\tilde{G}$, computed via Majority Interpolation.
*   **Mesh $K$:** The isosurface mesh extracted from the original grid $G$.
*   **Betti Numbers:** Tuples $(\beta_0, \beta_1, \beta_2)$ representing the topology.

**Definitions:**
*   **Trilinear Interpolation:** Within a grid cell, the scalar field is represented as:
    $$f(x, y, z) = axyz + bxy + cxz + dyz + ex + fy + gz + h$$
    where $a, b, c, d, e, f, g, h$ are coefficients derived from the vertex values [S1, unofficial].
*   **Ambiguous Cell:** A cell where the topology of the isosurface cannot be uniquely determined from the vertex signs alone (e.g., the saddle case in Marching Cubes) [S1, unofficial].
*   **Majority Interpolation:** A scheme to determine the digital surface $\partial O$ from the refined grid $\tilde{G}$ [S1, unofficial].

**Parameters:**
*   `#tests`: The number of random scalar fields to generate. Not given in the sources.
*   `max_refinement`: The maximum depth of grid subdivision allowed before restarting. Not given in the sources.
*   Boundary Scalar Value: Vertices on the boundary of $G$ are assigned a scalar value of 1 to ensure the surface does not have a boundary [S1, unofficial].

**Steps:**
1.  Initialize $G$ as an $n \times n \times n$ rectilinear grid [S1, unofficial].
2.  For $i$ from 1 to `#tests`:
    a.  Randomly sample scalar values for the vertices of $G$.
    b.  Refine and resample $G$ to generate a new grid $\tilde{G}$ without ambiguous cells.
        *   Refinement can be uniform (doubling resolution in each dimension) or non-uniform (using random node points or critical points) [S1, unofficial].
        *   If $\tilde{G}$ still contains ambiguous cubes after reaching `max_refinement`, restart the test from step 2a [S1, unofficial].
    c.  Compute the digital surface $\partial O$ from $\tilde{G}$ using Majority Interpolation [S1, unofficial].
    d.  Extract the isosurface mesh $K$ from the original grid $G$ using the isosurfacing technique under verification [S1, unofficial].
    e.  Compute the Betti numbers $(\beta_{0v}, \beta_{1v}, \beta_{2v})_i$ from the digital surface $\partial O$ using the `INVARIANT FROM DS` algorithm [S1, unofficial].
    f.  Compute the Betti numbers $(\beta_{0k}, \beta_{1k}, \beta_{2k})_i$ from the mesh $K$ using the `INVARIANT FROM MESH` algorithm [S1, unofficial].
    g.  Compare $(\beta_{0v}, \beta_{1v}, \beta_{2v})_i$ and $(\beta_{0k}, \beta_{1k}, \beta_{2k})_i$.
        *   If they do not match, the verification fails [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def digital_surface_verification(n, num_tests, max_refinement, extract_isosurface):
    """
    Verifies an isosurface extraction algorithm using Digital Topology.
    
    Args:
        n: Grid dimension (n x n x n).
        num_tests: Number of random tests to perform.
        max_refinement: Maximum number of refinement steps allowed.
        extract_isosurface: Function that takes a grid G and returns a mesh K.
        
    Returns:
        True if all tests pass, False otherwise.
    """
    for i in range(num_tests):
        # 1. Randomly sample G
        G = random_scalar_grid(n)
        
        # 2. Refine and resample G to generate G_tilde without ambiguous cells
        G_tilde = None
        refinement_steps = 0
        while refinement_steps <= max_refinement:
            # Attempt to refine G to G_tilde
            # Refinement can be uniform or non-uniform.
            # Here we assume a generic refine function that handles ambiguity removal.
            G_tilde = refine_and_resample(G)
            
            # Check for ambiguous cubes in G_tilde
            if not has_ambiguous_cubes(G_tilde):
                break
            refinement_steps += 1
        
        # If max refinement reached and ambiguous cells still remain, restart test
        if refinement_steps > max_refinement:
            continue # Go to next test (restart)
            
        # 3. Compute digital surface partial O from G_tilde using Majority Interpolation
        # Boundary vertices of G are assigned scalar value 1 to avoid surfaces with boundary.
        # This assignment should be done before or during refinement.
        set_boundary_scalars_to_one(G)
        partial_O = majority_interpolation(G_tilde)
        
        # 4. Extract the isosurface K from G using the isosurfacing technique under verification
        K = extract_isosurface(G)
        
        # 5. Compute Betti numbers from the digital surface
        beta_v = invariant_from_ds(partial_O)
        
        # 6. Compute Betti numbers from the mesh
        beta_k = invariant_from_mesh(K)
        
        # 7. Compare Betti numbers
        if beta_v != beta_k:
            return False
            
    return True

def random_scalar_grid(n):
    """Generates an n x n x n grid with random scalar values."""
    # Implementation detail: values typically in [-1, 1] or similar range.
    # Boundary vertices are assigned 1.
    grid = np.random.uniform(-1, 1, (n, n, n))
    # Set boundary to 1
    grid[0, :, :] = 1
    grid[-1, :, :] = 1
    grid[:, 0, :] = 1
    grid[:, -1, :] = 1
    grid[:, :, 0] = 1
    grid[:, :, -1] = 1
    return grid

def refine_and_resample(G):
    """
    Refines grid G to remove ambiguous cells.
    Returns a finer grid G_tilde.
    """
    # Implementation depends on specific refinement strategy (uniform or non-uniform).
    # Uniform: double resolution.
    # Non-uniform: insert nodes at critical points or random locations.
    # Resample scalar values to new vertices.
    # This function must ensure no ambiguous cells remain if called with sufficient depth.
    pass

def has_ambiguous_cubes(G):
    """Checks if any cell in grid G is ambiguous."""
    # An ambiguous cell is one where the topology cannot be uniquely determined.
    # For trilinear interpolation, this involves checking the signs of vertices
    # and potentially solving for critical points.
    pass

def majority_interpolation(G_tilde):
    """
    Computes the digital surface partial O from G_tilde.
    Uses Majority Interpolation scheme.
    """
    # Determine which voxels/faces belong to the object O.
    # Typically involves checking the sign of the scalar field at cell centers
    # or using a specific digital topology rule.
    pass

def invariant_from_ds(partial_O):
    """
    Computes Betti numbers (beta0, beta1, beta2) from the digital surface.
    """
    # Construct a simplicial complex or cell complex from partial_O.
    # Compute homology groups.
    pass

def invariant_from_mesh(K):
    """
    Computes Betti numbers (beta0, beta1, beta2) from the mesh K.
    """
    # Construct a simplicial complex from K.
    # Compute homology groups.
    pass
```

## Subtopics

### Contour Tree Verification
A verification structure using contour trees to describe the evolution of level-sets of simply connected domains. It naturally gives $\beta_0$ and can be extended to report $\beta_1$ and $\beta_2$, even for surfaces with boundaries [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the cost of grid refinement, digital surface extraction, and Betti number computation. Refinement may require multiple passes. Betti number computation is generally polynomial in the size of the complex.
*   **Memory Complexity:** Requires storing the original grid $G$, the refined grid $\tilde{G}$, the digital surface $\partial O$, and the mesh $K$. Memory usage increases with grid resolution and refinement depth.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method is valid for surfaces without boundary. Verifying isosurfaces with boundaries would require additional theoretical results [S1, unofficial].
*   **Strengths:** Provides detailed topological information ($\beta_0, \beta_1, \beta_2$) compared to SMT which only provides $\chi$. Can detect spurious features like cycling triangles that preserve $\chi$ but change $\beta_0$ [S1, unofficial].
*   **Limitations:** Only valid for surfaces without boundary. If maximum refinement is reached and ambiguous cells remain, the test must be restarted, which can be inefficient [S1, unofficial].
*   **When to use:** When detailed topological verification is required, especially to detect specific types of topological bugs.
*   **When not to use:** When the isosurface has boundaries, or when computational resources are extremely limited and only a coarse check (like SMT) is acceptable.

## Background
The method originates from the work on Topology Verification for Isosurface Extraction, specifically the section on Verification using Digital Topology [S1, unofficial]. It is part of the Method of Manufactured Solutions (MMS) framework applied to isosurface extraction. The approach uses digital topology to generate expected topological invariants from a refined grid and compares them to those extracted from the mesh. The paper notes that while SMT is less sensitive in theory (only computing $\chi$), it revealed problems in unison with digital surface tests, suggesting both are effective [S1, unofficial].

## Sources
[S1] Topology Verification for Isosurface Extraction Topology Verification for Isosurface Extraction, https://hal.science/hal-01211133, fetched 2026-10-26 (unofficial)
