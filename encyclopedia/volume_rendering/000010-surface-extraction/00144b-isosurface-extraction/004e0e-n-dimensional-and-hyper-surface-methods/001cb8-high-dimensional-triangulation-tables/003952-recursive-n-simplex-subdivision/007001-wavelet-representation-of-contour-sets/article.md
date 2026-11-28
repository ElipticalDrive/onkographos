# Wavelet Representation of Contour Sets

Also called: Adaptive Wavelet Representation of Contour Sets, Multiresolution contour representation, Subdivision-surface wavelets, Wavelet-based contour set compression

## Overview
This method represents scalar field contours as a hierarchy of meshes using subdivision surfaces and wavelets, allowing for compression via thresholding or encoding of wavelet coefficients [S1]. Unlike standard wavelet schemes that quantize field values sampled on a regular grid—risking topological errors such as spurious components enclosing local extrema—this approach compresses a parametrization of the scalar field induced by its contours [S1]. It preserves the prescribed contour topology at every resolution level by explicitly storing the geometry of base contours and interpolating intermediate levels through subdivision [S1].

## How it works
**Inputs:** A scalar field function defined on a regular grid.
**Outputs:** A compressed hierarchy of wavelet coefficients representing the contour set, or a reconstructed mesh at a desired resolution.

**Data Structures:**
*   **Base Mesh:** A hierarchical polygon mesh with subdivision connectivity.
    *   **Vertices:** Store 2D coordinates $(x, y)$ and an associated isovalue.
    *   **Edges:** Store connectivity flags for traversal.
    *   **Polygons:** Store lists of vertex indices.
*   **Wavelet Coefficients:** Sparse set of coefficients representing geometric displacements from the subdivision surface [S1].

**Parameters:**
*   $n_s$: Number of dyadic refinement levels to be computed [S1].
*   $\delta$: Finest sampling distance, slightly smaller than the edge length of the regular grid defining the field function [S1].
*   $J$: Maximal distance for vertex movement during Newton iteration to avoid divergence due to shallow gradients [S1].
*   $\Delta$: Maximal displacement during link relaxation to avoid coincidence of adjacent base vertices [S1].

**Step 1: Extract Base Contours**
Extract a finite set of base contours using uniformly distributed isolevels or denser sampling in regions of interest [S1]. This set defines the topology of all intermediate contours [S1]. The density of base contours should not be too high, as they represent the coarsest level of detail [S1].

**Step 2: Sample Base Vertices**
Sample base vertices on the extracted base contours at approximately equidistant intervals of arc length $A$ [S1].
The interval $A$ is calculated as:
$$ A = 2^{n_s} \cdot \delta $$
where $n_s$ is the number of dyadic refinement levels and $\delta$ is the finest sampling distance [S1].
*   All boundary points of contours must be base vertices [S1].
*   Every contour component must have at least three base vertices to avoid degenerate base polygons [S1].

**Step 3: Construct Links Between Base Contours**
Construct links between base vertices on adjacent base contours $C_a$ and $C_b$ [S1].
1.  Propagate vertices of $C_a$ onto $C_b$ using Newton iteration [S1].
2.  Restrict movement to a maximal distance $J$ to avoid divergence [S1].
3.  If a vertex does not converge (e.g., stuck at local extrema or zero-gradient areas), link it to the closest base point on $C_b$ and restore its initial position on $C_a$ [S1].
4.  Repeat the process with vertices of $C_b$ iterating towards $C_a$ [S1].
5.  Base vertices on the boundary of the data set are connected by traversing the boundary [S1].

**Step 4: Relax Links**
Relax links by moving base vertices along their corresponding contours to minimize link length [S1].
*   Restrict maximal displacement to value $\Delta$ to avoid coincidence of adjacent vertices [S1].
*   Connect isolated components ('islands') by one additional link to the closest base vertex on the surrounding contour [S1].

**Step 5: Construct Base Mesh**
Fill the space between adjacent base contours with convex polygons [S1].
1.  Traverse polygons in counter-clockwise orientation, recording base vertices [S1].
2.  Recursively split non-convex polygons or those with more than five edges by connecting close, non-adjacent vertices, avoiding self-intersections [S1].
3.  The result is a convex tessellation (base mesh) of the domain [S1].

**Step 6: Regular Subdivision**
Apply regular subdivision to the base mesh using Catmull-Clark refinement connectivity [S1].
1.  Insert vertices at centroids of polygons and midpoints of edges [S1].
2.  This generates quadrilaterals in the first step, which are regularly refined subsequently [S1].
3.  Assign isovales to new vertices:
    *   Vertices on base contours retain their isovalue.
    *   Vertices between two base contours are assigned the average of the minimum and maximum isovales of the quadrilateral's four corner vertices [S1].
    *   Vertices inside a quadrilateral are assigned the average of the minimum and maximum isovales of the quadrilateral's four corner vertices [S1].

**Step 7: Project Vertices onto Contours**
Project each vertex onto the contour with its assigned isovalue using constrained Newton iteration coupled with Laplacian smoothing [S1].
1.  Propagate along the gradient of the field [S1].
2.  Relax orthogonal to the gradient by projecting Laplacian displacement onto a vector/plane orthogonal to the gradient [S1].
3.  Limit maximal displacement by distance $J$ [S1].
4.  Terminate after a finite number of steps [S1].
    *   *Pitfall:* If a vertex cannot be projected because a nearby contour component does not exist (due to topology simplification), Laplacian relaxation prevents the mesh from entangling [S1].

**Step 8: Wavelet Compression**
Terminate subdivision when resolution is slightly finer than the grid resolution of the underlying field function [S1]. Compress the mesh hierarchy using wavelets [S1].
*   The geometric displacements from subdivision are represented as sparse wavelet coefficients [S1].
*   The contours produced by the subdivision process are either linear or cubic polynomials [S1].

**Pseudocode for Wavelet Reconstruction (Lifting Scheme):**
The inverse wavelet transform is computed by repeated reconstruction starting with a coarse base polygon. The reconstruction procedures for dyadic refinement of linear and cubic B-splines are defined as follows [S1]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def s_lift(v, e, a, b):
    """
    Scaling function lift operation.
    v_i <- v_i + a * e_{i-1} + a * e_j
    e_i <- e_i + b * v_i + v_{i+1}
    """
    # Implementation depends on specific mesh topology indexing.
    # For a regular grid or structured subdivision:
    # Update vertices based on adjacent edges
    # Update edges based on adjacent vertices
    pass

def w_lift(v, e, a, b):
    """
    Wavelet lift operation.
    Standard lifting step for detail coefficients.
    """
    pass

def reconstruct_linear_b_spline(mesh):
    """
    Linear B-spline wavelet reconstruction sequence:
    S-lift(-1/2, 1); w-lift(1/2, 1).
    """
    # Apply S-lift with a=-0.5, b=1
    # Apply W-lift with a=0.5, b=1
    pass

def reconstruct_cubic_b_spline(mesh):
    """
    Cubic B-spline wavelet reconstruction sequence:
    S-lift(-1/2, 1); w-lift(1/2, 1); S-lift(1/2, 1/2).
    """
    # Apply S-lift with a=-0.5, b=1
    # Apply W-lift with a=0.5, b=1
    # Apply S-lift with a=0.5, b=0.5
    pass
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The method stores a sparse set of wavelet coefficients, which compactly stores the parametrization [S1]. However, it requires additional space for storing contour topology compared to standard field compression [S1].
*   **Performance:** The method allows for efficient access to geometry and multiresolution modeling of large-scale isosurfaces [S1].
*   **Strengths:** Preserves contour topology at every resolution level, even with high quantization errors [S1]. Avoids spurious contour components enclosing local extrema of reconstruction error [S1].
*   **Limitations:** Requires additional space for storing contour topology [S1]. Geometric error is bounded by half the sampling distance for base isovalues [S1]. Mesh collapse or stretching can occur near critical points (saddle points) or long "headlands" [S1].
*   **When to use:** Multiresolution modeling of large-scale isosurfaces and compression via thresholding or encoding of quantized wavelet coefficients [S1].
*   **When not to use:** When topological correctness of the scalar field reconstruction (not just contours) is required, or when storage for topology is prohibitive.

## Background
**Origin:** The method was presented by Martin Bertram, Daniel E. Laney, Mark A. Duchaineau, and Kenneth I. Joy [S1].
**History:** It is a wavelet compression and multiresolution modeling approach for sets of contours (level sets) [S1]. It generalizes previous wavelet schemes by creating a parametrization of a scalar field induced by its contours rather than compressing function values sampled on a regular grid [S1].
**Usage:** Used for multiresolution modeling of large-scale isosurfaces and compression [S1]. It applies to scalar fields defined on planar domains, and generalizes to volumetric domains, time-varying contours, and level sets of vector fields [S1].

## Sources
[S1] Wavelet Representation of Contour Sets, https://digital.library.unt.edu/ark:/67531/metadc1405239/?q=volume data compression wavelet, fetched 2026-10-28
