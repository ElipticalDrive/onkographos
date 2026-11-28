# Scene Contraction

Also called: Mip-NeRF 360 Scene Contraction, Mip-NeRF 360 Contraction, Contract Function, Mip-NeRF 360 contract function, Contracted space parameterization, Space Contraction, Coordinate contraction

## Overview
Scene Contraction is a coordinate transformation method that maps unbounded 3D space into a bounded region, enabling the use of hash-based encodings or voxel grids for large-scale neural radiance fields [S1][S2, unofficial][S4]. It solves the problem of infinite coordinate values breaking hash tables or MLPs by compressing distant background geometry into a finite ball or cube [S2, unofficial][S6, unofficial]. It differs from standard normalization by preserving angular direction while non-linearly compressing radial distance, ensuring points at infinity map to the boundary of the bounded space [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D point coordinate $x \in \mathbb{R}^3$ in the original unbounded scene space (often denoted as $t$-space) [S8].
*   Output: A 3D point coordinate $x' \in \mathbb{R}^3$ in the contracted bounded space (often denoted as $s$-space) [S8].

**Data Structures:**
No specific data structures are required for the contraction function itself; it operates on individual coordinates [G]. The output is typically used to index into a multi-resolution hash grid or voxel grid defined in the bounded space [S1][S2, unofficial].

**Parameters:**
*   $r$: The radius of the inner sphere (or half-width of the inner cube) where no contraction occurs. In the standard Mip-NeRF 360 implementation, the inner radius threshold is $r=1$ [S1][S2, unofficial]. In other variants, $r$ may be set to the scene bound (e.g., $r=10$) [S3, unofficial].
*   $b$: A scaling parameter for the contraction rate outside the inner region. In the standard Mip-NeRF 360 implementation, $b=1$ [S3, unofficial].

**Formulas:**
The most common formulation, used in Mip-NeRF 360 and Instant-NGP, uses the L2 norm to map space into a ball of radius 2 [S1][S2, unofficial][S4]:

$$
\text{contract}(x) = 
\begin{cases} 
x & \text{if } \|x\| \le 1 \\
\left(2 - \frac{1}{\|x\|}\right) \frac{x}{\|x\|} & \text{if } \|x\| > 1 
\end{cases}
$$

This can be algebraically simplified for implementation as:

$$
\text{contract}(x) = 
\begin{cases} 
x & \text{if } \|x\| \le 1 \\
x \cdot \frac{2\|x\| - 1}{\|x\|^2} & \text{if } \|x\| > 1 
\end{cases}
$$

An alternative formulation uses the infinity norm ($\|\cdot\|_\infty$) to map space into a unit cube [S9]:

$$
\chi(x) = 
\begin{cases} 
x & \text{if } \|x\|_\infty \le 1 \\
\frac{x}{2 - \|x\|_\infty} & \text{if } \|x\|_\infty > 1 
\end{cases}
$$

A more general parameterized formulation allows for arbitrary inner radius $r$ and boundary parameter $b$ [S3, unofficial]:

$$
\text{contract}(x) = 
\begin{cases} 
\frac{x}{r} & \text{if } \|x\| \le r \\
\frac{x}{b(1 + b - \frac{\|x\|}{r})} & \text{otherwise}
\end{cases}
$$

**Steps:**
1.  **Normalization:** Normalize the scene coordinates such that the region of interest (foreground) falls within a unit sphere or cube [S10]. This is often done by scaling the scene frames with a factor [S4].
2.  **Compute Norm:** Calculate the norm of the input point $x$ [G]. For L2 contraction, compute $\|x\|_2 = \sqrt{x_x^2 + x_y^2 + x_z^2}$ [G]. For L-infinity contraction, compute $\|x\|_\infty = \max(|x_x|, |x_y|, |x_z|)$ [G].
3.  **Condition Check:** Compare the norm against the threshold (1 for standard Mip-NeRF 360, or $r$ for the general case) [S1][S3, unofficial].
4.  **Transform:** If the norm is less than or equal to the threshold, return $x$ unchanged [S1]. If the norm is greater than the threshold, apply the contraction formula to compress the point into the bounded region [S1].
5.  **Usage:** Use the resulting contracted coordinate $x'$ to query the hash grid or voxel grid [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def contract_l2(x, threshold=1.0):
    """
    Contracts 3D point x into a bounded ball of radius 2 * threshold.
    Uses the Mip-NeRF 360 contraction function.
    
    Args:
        x: list or tuple of 3 floats (x, y, z)
        threshold: float, radius of the inner sphere where no contraction occurs (default 1.0)
        
    Returns:
        list of 3 floats: contracted coordinates
    """
    # Compute L2 norm
    norm = math.sqrt(x[0]**2 + x[1]**2 + x[2]**2)
    
    # If inside the inner sphere, return unchanged
    if norm <= threshold:
        return x
    
    # If outside, apply contraction
    # Formula: (2 - 1/||x||) * (x / ||x||)
    # Simplified factor: (2 * norm - 1) / norm^2
    factor = (2.0 * norm - 1.0) / (norm * norm)
    return [x[0] * factor, x[1] * factor, x[2] * factor]

def contract_linf(x, threshold=1.0):
    """
    Contracts 3D point x into a bounded cube of size 2 * threshold.
    Uses the infinity norm contraction.
    
    Args:
        x: list or tuple of 3 floats (x, y, z)
        threshold: float, half-width of the inner cube (default 1.0)
        
    Returns:
        list of 3 floats: contracted coordinates
    """
    # Compute L-infinity norm
    norm_inf = max(abs(x[0]), abs(x[1]), abs(x[2]))
    
    # If inside the inner cube, return unchanged
    if norm_inf <= threshold:
        return x
    
    # If outside, apply contraction
    # Formula from [S9]: x / (2 - ||x||_inf)
    factor = 1.0 / (2.0 - norm_inf)
    return [x[0] * factor, x[1] * factor, x[2] * factor]
```

**Edge Cases and Pitfalls:**
*   **Zero Vector:** If $x = (0,0,0)$, the norm is 0 [G]. The condition $\|x\| \le 1$ holds, so it returns $x$ unchanged [G]. No division by zero occurs [G].
*   **Precision:** For points very close to the boundary ($\|x\| \approx 1$), the function is continuous [G]. Floating-point precision errors near the boundary should be handled by ensuring the condition check is robust [G].
*   **Unbounded Background:** Without contraction, points at infinity would cause hash table collisions or MLP overflow [S2, unofficial]. Contraction ensures all points map to a finite range [S2, unofficial].
*   **Mesh Watertightness:** When contracting mesh points for surface reconstruction, the non-linearity of the contraction can break the watertight manifold property if applied directly to the mesh geometry in the original space [S6, unofficial]. It is recommended to apply marching cubes or other mesh extraction in the original space ($t$-space) and then contract the resulting mesh points to the contracted space ($s$-space) for color computation [S6, unofficial].

## Subtopics

### [L2 Norm Sphere Contraction](12197-l2-norm-sphere-contraction.md)
Uses the L2 norm to map unbounded space into a sphere. Preserves angular direction. Choose when spherical symmetry is desired or when using hash grids designed for spherical bounding volumes.

### [L-infinity Norm Cube Contraction](12198-l-infinity-norm-cube-contraction.md)
Uses the L-infinity norm to map unbounded space into a cube. Choose when using hash grids or voxel grids with axis-aligned bounding boxes, as it aligns with the grid structure.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per point [G]. The computation involves a few arithmetic operations and a square root (for L2) or max (for L-infinity) [G].
*   **Memory Complexity:** $O(1)$ additional memory [G]. No auxiliary data structures are needed for the transformation itself [G].
*   **Strengths:** Enables the use of hash-based encodings (like Instant-NGP) for unbounded scenes [S2, unofficial]. Improves training stability for large scenes by bounding coordinate values [S4].
*   **When to use:** Always use for unbounded scenes when employing hash-based or voxel-based neural radiance fields [S2, unofficial]. Not needed for bounded scenes or methods that do not rely on coordinate-based lookups [S2, unofficial].

## Background
Scene Contraction was introduced in the Mip-NeRF 360 paper by Barron et al. [S4][S6, unofficial] to handle large, unbounded 360-degree scenes. It was adopted by Instant-NGP [S2, unofficial] and subsequent works to enable efficient training on unbounded datasets. The method maps the infinite background into a bounded region, allowing the use of multi-resolution hash grids which require finite coordinate ranges [S1][S2, unofficial].

## Sources
[S1] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
[S2] Factor Fields: A Unified Framework for Neural Fields and Beyond, https://doi.org/10.48550/arxiv.2302.01226, fetched 2026-10-26 (unofficial)
[S3] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
[S4] LiDAR-NeRF: Novel LiDAR View Synthesis via Neural Radiance Fields, https://arxiv.org/abs/2304.10406v2, fetched 2026-10-26
[S5] Strata-NeRF : Neural Radiance Fields for Stratified Scenes, https://arxiv.org/abs/2308.10337v1, fetched 2026-10-26
[S6] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
[S7] NeRF — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-NeRF_SOTA_Updated.html, fetched 2026-10-29 (unofficial)
[S8] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
[S9] Multi-Level Neural Scene Graphs for Dynamic Urban Environments, https://doi.org/10.1109/cvpr52733.2024.01996, fetched 2026-10-29
[S10] HyRF: Hybrid Radiance Fields for Memory-efficient and High-quality Novel View Synthesis, https://doi.org/10.52202/085713-0617, fetched 2026-10-30
[S11] Compact 3D Scene Representation via Self-Organizing Gaussian Grids, https://doi.org/10.1007/978-3-031-73013-9_2, fetched 2026-10-26
