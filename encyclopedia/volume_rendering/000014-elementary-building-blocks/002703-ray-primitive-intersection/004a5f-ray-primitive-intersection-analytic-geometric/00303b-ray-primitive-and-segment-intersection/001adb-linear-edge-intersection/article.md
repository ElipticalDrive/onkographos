# Linear Edge Intersection

Also called: Linear Edge Interpolation, Linear Interpolation, Midpoint Intersection (variant), Inverse Linear Interpolation (algebraic form)

## Overview
Linear Edge Intersection determines the precise position where an isosurface intersects a grid edge by linearly interpolating between the scalar field values at the two vertices defining that edge [S1, S2, unofficial][S4][S5, S6, S7, unofficial]. It solves the problem of locating surface geometry within a discrete voxel grid, serving as the fundamental geometric primitive for contouring algorithms like Marching Cubes and ray-marching isosurface extraction [S1, S9, unofficial]. It sets itself apart from its siblings by assuming a strictly linear variation of the scalar field along the edge, providing an exact algebraic solution for the intersection point under that assumption, whereas [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md) uses numerical approximation and [Hermite Interpolation for Isosurface Extraction](13666-hermite-interpolation-for-isosurface-extraction.md) uses gradient data for higher-order accuracy [S1, unofficial][S4, S13].

## How it works
**Inputs:**
*   Two edge endpoints: $\mathbf{p}_1$ and $\mathbf{p}_2$ (3D coordinates).
*   Scalar field values at endpoints: $v_1 = f(\mathbf{p}_1)$ and $v_2 = f(\mathbf{p}_2)$.
*   Target isovalue (threshold): $iso$ (or $k$, $T$, $Y$).

**Outputs:**
*   Intersection point $\mathbf{p}$ on the edge segment $[\mathbf{p}_1, \mathbf{p}_2]$.

**Data Structures:**
*   Points are represented as 3D vectors $(x, y, z)$.
*   Scalar values are floating-point numbers.

**Steps:**
1.  **Identify Intersection:** Check if the edge crosses the isosurface. This occurs if $(v_1 - iso)$ and $(v_2 - iso)$ have opposite signs, or equivalently, if $(v_1 - iso)(v_2 - iso) < 0$. If both values are on the same side of the isovalue, there is no intersection on this edge segment [S1, S9, unofficial].
2.  **Handle Degeneracy:** If the scalar values are nearly identical ($|v_1 - v_2| < \epsilon$), the denominator in the interpolation formula approaches zero. In this case, return $\mathbf{p}_1$ (or $\mathbf{p}_2$, as they are effectively the same point relative to the surface) to avoid division by zero [S2, S6, S7, unofficial]. Typical $\epsilon$ values are $10^{-7}$ or $10^{-5}$ [S5, S6, unofficial].
3.  **Calculate Interpolation Factor:** Compute the normalized distance $t$ (or $\mu$, $\alpha$, $k$) from $\mathbf{p}_1$ to the intersection point:
    $$ t = \frac{iso - v_1}{v_2 - v_1} $$
    Alternatively, using signed distances $d_1 = v_1 - iso$ and $d_2 = v_2 - iso$ (where $d_1$ and $d_2$ have opposite signs):
    $$ \mathbf{p} = \frac{d_1 \mathbf{p}_2 - d_2 \mathbf{p}_1}{d_1 - d_2} $$
    This form is numerically more stable when vertices are near the plane [S3, unofficial].
4.  **Compute Position:** Calculate the intersection point using vector interpolation:
    $$ \mathbf{p} = \mathbf{p}_1 + t (\mathbf{p}_2 - \mathbf{p}_1) $$
    Or in component form:
    $$ \mathbf{p} = (1 - t) \mathbf{p}_1 + t \mathbf{p}_2 $$
5.  **Clamping (Optional but Recommended):** To handle floating-point inaccuracies where $t$ might slightly exceed $[0, 1]$, clamp $t$ to $[0, 1]$ before computing $\mathbf{p}$ [S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_edge_intersection(p1, p2, v1, v2, iso, epsilon=1e-7):
    """
    Calculates the intersection point of an isosurface with an edge.
    
    Args:
        p1: 3D vector (x, y, z) of the first endpoint.
        p2: 3D vector (x, y, z) of the second endpoint.
        v1: Scalar value at p1.
        v2: Scalar value at p2.
        iso: Target isovalue.
        epsilon: Threshold for degenerate case detection.
        
    Returns:
        3D vector representing the intersection point.
    """
    # Check for degenerate case where endpoints have same scalar value
    if abs(v2 - v1) < epsilon:
        return p1
    
    # Calculate interpolation factor t
    # t represents the fraction of the distance from p1 to p2
    t = (iso - v1) / (v2 - v1)
    
    # Clamp t to [0, 1] to handle floating point errors
    t = max(0.0, min(1.0, t))
    
    # Compute intersection point
    # p = p1 + t * (p2 - p1)
    px = p1.x + t * (p2.x - p1.x)
    py = p1.y + t * (p2.y - p1.y)
    pz = p1.z + t * (p2.z - p1.z)
    
    return (px, py, pz)
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $v_1 = v_2$, the denominator is zero. This must be guarded against [S2, S6, S7, unofficial].
*   **Numerical Stability:** When vertices are very close to the isosurface, standard interpolation can suffer from cancellation errors. Using signed distances ($d_0, d_1$) as in [S3, unofficial] improves stability.
*   **Multiple Roots:** Linear interpolation assumes a single root per edge. If the scalar field has multiple roots within the voxel (e.g., a thin structure passing through), linear interpolation will fail to detect all intersections [S9, unofficial].
*   **Vertex Ordering:** In some implementations (e.g., [S2, unofficial]), consistent ordering of $P_1$ and $P_2$ (e.g., by coordinate value) is required to ensure correct triangle generation and normal orientation.

## Subtopics

### [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md)
Iterative Bisection Intersection Refinement is a post-detection technique used to accurately locate the intersection point of a viewing ray with an isosurface within a volume dataset. It solves the problem of geometric inaccuracy inherent in coarse adaptive sampling or sphere tracing by iteratively narrowing the interval known to contain the root. It sets itself apart from its siblings by relying on repeated linear interpolation (bisection) to refine the position, offering a balance of speed and moderate accuracy at the cost of potential artifacts in complex topologies.

### [Linear Intersection Approximation](1232-linear-intersection-approximation.md)
Linear Intersection Approximation (also known as Midpoint Intersection) estimates the intersection point of an isosurface with a grid edge by calculating the geometric midpoint of the edge, rather than performing linear interpolation based on scalar values. It differs from its parent, [Linear Edge Intersection](868-linear-edge-intersection.md), by ignoring endpoint scalar values entirely, and from its sibling, [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md), by using a single fixed geometric step instead of recursive refinement [G].

### [Inverse Linear Interpolation](12154-inverse-linear-interpolation.md)
Inverse Linear Interpolation calculates the non-integer position $x$ corresponding to a specific scalar value $v$ between two known values $v_1$ and $v_2$ at positions $x_1$ and $x_2$, assuming linear variation. It is primarily used for finding iso-surface intersections or specific density levels within a volume. Unlike [Linear Edge Intersection](868-linear-edge-intersection.md), which computes the exact intersection on a grid edge, this method provides the general algebraic inversion of linear interpolation.

### [Hermite Interpolation for Isosurface Extraction](13666-hermite-interpolation-for-isosurface-extraction.md)
Hermite interpolation for isosurface extraction determines the precise position of an isovertex along a zero-crossing edge by utilizing both scalar field values and gradient information (Hermite data) at the edge endpoints. Unlike [Linear Edge Intersection](868-linear-edge-intersection.md), which assumes linear variation and produces artifacts on non-linear implicit functions, this method fits higher-order polynomials to the local field behavior, significantly improving surface accuracy. It is primarily used in contouring algorithms like Marching Cubes where gradient data is available or can be approximated.

### [Edge-Crossing Error Approximation](15075-edge-crossing-error-approximation.md)
This method estimates the positional error introduced by linear interpolation when determining the intersection of a scalar field with an isovalue on a grid edge. It leverages Taylor series expansion to bound the difference between the true root and the linear approximation without solving a linear system. It distinguishes itself from [Linear Edge Intersection](868-linear-edge-intersection.md) by providing a quantitative error metric rather than just the position, and from [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md) by offering an analytical estimate rather than a numerical refinement.

### Linear Density Minimization for Zero-Crossing
Approximates the zero-crossing position along an edge by sampling discrete points and selecting the one with minimum absolute density [S10, unofficial]. It initializes a minimum value and iterates through a fixed number of steps (e.g., 8) along the edge, updating the best position if a lower absolute density is found [S10, unofficial].

### Triangle Point Generation (Linear Interpolation with Snap)
Uses linear interpolation but snaps vertices to grid points if they are within a small threshold (`snap`, typically 0.01) of the grid edge point or if the scalar values are nearly identical [S11, unofficial]. This reduces mesh complexity and potential artifacts from floating-point noise [S11, unofficial].

### Wall Edge Clipping
Clips wall edges to a screen or player boundary (e.g., $y < 1$) using linear interpolation to find the intersection point [S12, unofficial]. If an endpoint is behind the boundary, it calculates the interpolation factor $t$ and updates the coordinates to lie exactly on the boundary plane [S12, unofficial].

### Three-segment interpolation
An interpolation method between linear interpolation and midpoint selection used to calculate the intersection point of an edge and an iso-surface [S13]. It normalizes the interpolation ratio $k$ based on thresholds $m$ and $n$: if $|k| < m$, $|k|$ is set to $q$; if $|k| > n$, $|k|$ is set to $p$; otherwise, $|k|$ is set to 0.5 [S13]. This reduces artifacts in specific topological configurations [S13].

### Iterative Bisection for Zero-Crossing
A method to find the point where the SDF assumes the isovalue $c=0$ along an edge exhibiting a material change, reduced to finding a value $t \in [0,1]$ [S14]. It applies binary search with a limit of 8 iterations and a sub-interval size threshold of $10^{-6}$ [S14].

### Multiple Edge Intersection Removal
Handles cases where the surface intersects a grid edge more than once [S15]. It identifies a point where the gradient is perpendicular to the edge and deforms the surface to ensure only single edge intersections remain, using linear interpolation in the direction of the gradient [S15].

### Vertex Handling for Grid Vertices
Handles the case where the surface passes through vertices of the grid by considering $F$ to be strictly positive whenever $F \ge 0$ [S15]. This ensures consistent topology when vertices lie exactly on the isosurface.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per edge. The calculation involves a constant number of arithmetic operations [S4][S9, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are required beyond the input values [S4].
*   **Performance:** Extremely fast, making it suitable for real-time applications and large-scale volume rendering [S4][S9, unofficial]. It is significantly faster than higher-order interpolation methods [S4].
*   **Accuracy:** Provides an exact solution under the assumption of linear field variation. Accuracy degrades if the scalar field is highly non-linear within the voxel [S4].
*   **Strengths:** Simplicity, speed, and numerical stability (when implemented with signed distances) [S3, unofficial][S4].
*   **Limitations:** Cannot resolve multiple roots within a single edge/voxel [S9, unofficial]. May produce artifacts (e.g., "crinkled" surfaces) if the underlying field is not smooth or linear [S4].
*   **When to Use:** When performance is critical and the scalar field is relatively smooth or linear within voxels [S4][S9, unofficial].
*   **When Not to Use:** When high geometric accuracy is required for highly non-linear fields, or when multiple intersections per voxel are expected [S4][S9, unofficial].

## Background
Linear Edge Intersection is a fundamental technique in isosurface extraction, originating from the Marching Cubes algorithm introduced by Lorensen and Cline in 1987 [S1, S2, S6, unofficial]. It is widely used in medical imaging, scientific visualization, and computer graphics for converting volumetric data into polygonal meshes [S1, unofficial][S4][S7, unofficial]. The method has been extended and refined in various contexts, including conservative voxelization [S3, unofficial] and uncertainty estimation [S4].

## Sources
[S1] Linear interpolation on Plane (Marching Cubes), https://computergraphics.stackexchange.com/questions/2501/linear-interpolation-on-plane-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] Marching Cubes - Holes in Mesh, https://stackoverflow.com/questions/67152672/marching-cubes-holes-in-mesh, fetched 2026-10-27 (unofficial)
[S3] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
[S4] Estimation and Visualization of Isosurface Uncertainty from Linear and High-Order Interpolation Methods, https://arxiv.org/abs/2409.00043v1, fetched 2026-10-26
[S5] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
[S6] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S7] samsfisher/3d-model-xray, https://github.com/samsfisher/3d-model-xray, fetched 2026-10-27 (unofficial)
[S8] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S9] PDF Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray ..., https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29 (unofficial)
[S10] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S11] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S12] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
[S13] A marching cube algorithm based on edge growth, https://doi.org/10.1016/j.vrih.2021.08.006, fetched 2026-10-26
[S14] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S15] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
