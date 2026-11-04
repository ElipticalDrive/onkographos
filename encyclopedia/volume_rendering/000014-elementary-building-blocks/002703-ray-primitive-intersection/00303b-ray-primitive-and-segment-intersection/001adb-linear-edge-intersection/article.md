# Linear Edge Intersection

Also called: Linear Edge Interpolation, Edge Crossing Linear Interpolation, Edge Intersection Vertex Interpolation, Linear Interpolation for Edge Intersection

## Overview
Linear Edge Intersection calculates the exact position where an isosurface (defined by a scalar field threshold) intersects a straight edge connecting two grid vertices, assuming the scalar field varies linearly between those vertices [S2, S3, S4, unofficial]. It solves the geometric problem of locating zero-crossings (or threshold crossings) on discrete grid structures, which is a fundamental step in algorithms like Marching Cubes, Dual Contouring, and ray-marching volume rendering [S3, S6, unofficial]. Unlike [Voxel selection via pick ray intersection](893-voxel-selection-via-pick-ray-intersection.md), which determines *which* voxel a ray enters, this method determines *where* on a specific segment the surface lies [S6, unofficial]. It is distinguished from [Linear Intersection Approximation](1232-linear-intersection-approximation.md) by using the actual scalar values at the endpoints rather than assuming the midpoint [S2, S3, unofficial].

## How it works
**Inputs:**
*   $\mathbf{p}_1, \mathbf{p}_2$: 3D coordinates of the two edge endpoints (vertices).
*   $v_1, v_2$: Scalar field values (e.g., density, SDF, intensity) at $\mathbf{p}_1$ and $\mathbf{p}_2$.
*   $isoLevel$: The threshold value defining the isosurface.
*   $eps$: A small tolerance for numerical stability checks (typically $10^{-5}$ to $10^{-7}$) [S2, S3, unofficial].

**Outputs:**
*   $\mathbf{p}$: The 3D coordinate of the intersection point on the edge.

**Data Structures:**
*   Vectors/Points: 3-element arrays or structs containing $(x, y, z)$ coordinates.
*   Scalars: Floating-point numbers for values and thresholds.

**Steps:**
1.  **Check for Degeneracy:** Calculate the difference $\Delta v = v_2 - v_1$. If $|\Delta v| < eps$, the scalar values are effectively identical. Return $\mathbf{p}_1$ as the intersection point [S2, S3, S4, unofficial]. This prevents division by zero.
2.  **Calculate Interpolation Factor:** Compute the normalized distance $\mu$ along the edge where the scalar value equals $isoLevel$. The standard formula is:
    $$ \mu = \frac{isoLevel - v_1}{v_2 - v_1} $$
    Some implementations clamp $\mu$ to the range $[0.0, 1.0]$ to handle cases where the threshold is slightly outside the sampled range due to noise or precision errors [S3, unofficial].
3.  **Compute Intersection Point:** Interpolate the position using $\mu$:
    $$ \mathbf{p} = \mathbf{p}_1 + \mu (\mathbf{p}_2 - \mathbf{p}_1) $$
    Alternatively, this can be expressed as a weighted average:
    $$ \mathbf{p} = (1 - \mu)\mathbf{p}_1 + \mu\mathbf{p}_2 $$
4.  **Alternative Formulation (Signed Distance/Plane):** In clipping contexts (e.g., convex clipping), if $d_0$ and $d_1$ are the signed distances of $\mathbf{p}_1$ and $\mathbf{p}_2$ to a clipping plane (where $d_0 > 0$ and $d_1 \le 0$), the intersection $\mathbf{x}_p$ is calculated as [S1, unofficial]:
    $$ \mathbf{x}_p = \frac{d_0 \mathbf{p}_2 - d_1 \mathbf{p}_1}{d_0 - d_1} $$
    This form is numerically more stable when vertices lie very near the plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_edge_intersection(p1, p2, val1, val2, iso_level, eps=1e-7):
    """
    Calculates the intersection point of an isosurface with the edge defined by p1 and p2.
    
    Args:
        p1: Tuple/Array (x, y, z) of the first vertex.
        p2: Tuple/Array (x, y, z) of the second vertex.
        val1: Scalar value at p1.
        val2: Scalar value at p2.
        iso_level: The threshold value for the isosurface.
        eps: Tolerance for degenerate cases.
        
    Returns:
        Tuple/Array (x, y, z) of the intersection point.
    """
    # Step 1: Check for degeneracy (division by zero)
    diff = val2 - val1
    if abs(diff) < eps:
        return p1
    
    # Step 2: Calculate interpolation factor
    # Standard linear interpolation formula
    mu = (iso_level - val1) / diff
    
    # Optional: Clamp to [0, 1] if the threshold is outside the range [val1, val2]
    # This handles cases where the surface does not strictly cross the edge 
    # but the threshold is slightly out of bounds due to noise.
    # mu = max(0.0, min(1.0, mu)) 
    
    # Step 3: Compute intersection point
    # p = p1 + mu * (p2 - p1)
    px = p1[0] + mu * (p2[0] - p1[0])
    py = p1[1] + mu * (p2[1] - p1[1])
    pz = p1[2] + mu * (p2[2] - p1[2])
    
    return (px, py, pz)
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $v_1 = v_2$, the denominator is zero. The method must check $|v_2 - v_1| < eps$ and return an endpoint (typically $p_1$) [S2, S3, S4, unofficial].
*   **Threshold Outside Range:** If $isoLevel$ is not between $v_1$ and $v_2$, the mathematical intersection lies outside the segment $[p_1, p_2]$. Some implementations return $p_1$ or clamp $\mu$ to $[0,1]$ [S3, S4, unofficial]. Others may return "No Hit" if strict geometric validity is required [S6, unofficial].
*   **Multiple Roots:** In ray-marching contexts, if the scalar field has two roots within a voxel, linear interpolation between entry and exit points may fail to detect the intersection because the signs at the endpoints might be identical [S6, unofficial]. This method assumes a single crossing per edge segment.
*   **Numerical Stability:** When vertices are very close to the plane (in clipping applications), using signed distances directly in the weighted average formula is more stable than solving parametric equations [S1, unofficial].

## Subtopics

### [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md)
Iterative Bisection Intersection Refinement is an iterative root-finding method used to refine the intersection position after a sign change has been detected during ray marching. It narrows down the interval $[t_{prev}, t_{curr}]$ containing the isosurface by iteratively applying linear interpolation (bisection) within the segment. This approach solves the problem of geometric inaccuracy inherent in discrete sampling, providing a balance between computational speed and visual fidelity. Unlike closed-form algebraic solutions or higher-order root isolation techniques, it relies on repeated linear approximation. It builds on [Linear Edge Intersection](868-linear-edge-intersection.md) by applying similar linear interpolation principles within an iterative refinement loop over a ray segment rather than a single edge.

### [Linear Intersection Approximation](1232-linear-intersection-approximation.md)
Linear Intersection Approximation (also known as Midpoint Intersection) is a method for estimating the intersection point of an isosurface with a grid edge. It solves the problem of determining where a scalar field crosses a threshold value by calculating the geometric midpoint of the edge, rather than performing linear interpolation based on the scalar values at the endpoints. It is distinguished from its parent, [Linear Edge Intersection](868-linear-edge-intersection.md), by ignoring the scalar field values entirely, and from its sibling, [Iterative Bisection Intersection Refinement](880-iterative-bisection-intersection-refinement.md), by using a single fixed step rather than recursive refinement.

### Linear Density Minimization for Zero-Crossing
Approximates the zero-crossing position along an edge by sampling discrete points and selecting the one with minimum absolute density [S7, unofficial].
**Steps:**
1.  Initialize `minValue` to a large number (e.g., 100000.0), `t` to 0.0, `currentT` to 0.0, `steps` to 8, and `increment` to $1.0 / steps$ [S7, unofficial].
2.  Loop while `currentT` $\le 1.0$:
    *   Compute point $p = p_0 + (p_1 - p_0) \times currentT$ [S7, unofficial].
    *   Compute density $= |Density\_Func(p)|$ [S7, unofficial].
    *   If density $< minValue$, update `minValue` $=$ density and `t` $=$ `currentT` [S7, unofficial].
    *   Increment `currentT` $+=$ `increment` [S7, unofficial].
3.  Return $p_0 + (p_1 - p_0) \times t$ [S7, unofficial].

### Triangle Point Generation (Linear Interpolation with Snap)
Generates intersection points with snapping behavior for vertices close to the grid edge point [S8, unofficial].
**Steps:**
1.  If $|v_1| < snap$, return $p_1$ [S8, unofficial].
2.  If $|v_2| < snap$, return $p_2$ [S8, unofficial].
3.  If $|v_2 - v_1| < snap$, return $p_1$ [S8, unofficial].
**Formula:**
$$ \mu = \frac{v_2}{v_2 - v_1} $$
$$ point = \mu \mathbf{p}_1 + (1 - \mu) \mathbf{p}_2 $$
**Parameter:**
*   $snap$: Threshold to snap vertices when close to grid edge point. Default value is 0.01 [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$. The calculation involves a constant number of arithmetic operations (subtraction, division, multiplication, addition) regardless of grid size.
*   **Memory Complexity:** $O(1)$. No additional data structures are required beyond the input variables.
*   **Performance:** Extremely fast. It is the dominant operation in Marching Cubes and similar surface extraction algorithms, often executed millions of times per frame.
*   **Strengths:** Exact solution under the linear assumption; numerically stable with proper epsilon checks; simple to implement.
*   **Limitations:** Assumes linear variation of the scalar field, which is an approximation. It cannot resolve multiple intersections within a single edge segment [S6, unofficial]. Accuracy is limited by the grid resolution; finer grids yield more accurate surface representations.
*   **When to use:** When extracting surfaces from voxel data (Marching Cubes, Dual Contouring) or performing ray-marching with linear interpolation.
*   **When not to use:** When the scalar field is known to be highly non-linear within the edge segment and higher-order interpolation (e.g., quadratic or cubic) is required for visual fidelity, or when multiple roots are expected within the segment.

## Background
Linear Edge Intersection is a foundational technique in computer graphics, primarily associated with the Marching Cubes algorithm introduced by Lorensen and Cline in 1987 [S3, unofficial]. It is also used in Dual Contouring [S7, unofficial], ray-marching volume rendering [S6, unofficial], and various mesh reconstruction techniques [S8, unofficial]. The method relies on the assumption that scalar fields vary linearly between grid points, which is a standard approximation in discrete field processing. Variations of this method, such as using signed distances for clipping [S1, unofficial] or snapping for robustness [S8, unofficial], have been developed to address specific numerical stability and geometric consistency issues.

## Sources
[S1] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
[S2] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
[S3] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S4] samsfisher/3d-model-xray, https://github.com/samsfisher/3d-model-xray, fetched 2026-10-27 (unofficial)
[S5] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S6] PDF Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray ..., https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29 (unofficial)
[S7] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S8] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
