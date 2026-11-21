# Gaussian Splitting for Bent Shapes

Also called: Bent Gaussian Splitting

## Overview
A procedure to split bent Gaussians into multiple well-formed ellipsoids to approximate non-rigid warping shapes that a single Gaussian cannot represent, preventing visual artifacts [S1, unofficial]. It addresses the limitation where a single transformed Gaussian fails to properly represent bent shapes caused by non-rigid warping, which leads to visual artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians, each represented by a proxy point cloud $AP_d$ after initial deformation (e.g., cage-based deformation). The proxy points include the deformed center $c'_d$ and deformed points on the principal axes (e.g., $x'_1, x'_2$ for the x-axis).
*   **Output:** A refined set of Gaussians where bent primitives have been split into multiple smaller, well-formed ellipsoids, ready for MVC (Mean Value Coordinates) transformation.

**Data Structures:**
*   **Proxy Point Cloud ($AP_d$):** A set of points representing the Gaussian ellipsoid. For a Gaussian with semi-principal axes $a, b, c$, the proxy points typically include the center and points along the axes at distances proportional to the axis lengths.
*   **Gaussian Parameters:** Mean position, covariance matrix (or semi-principal axes $a, b, c$ and orientation), opacity, and color.

**Parameters:**
*   **Splitting Threshold:** $175^\circ$. Splitting is required if the calculated angle falls below this value [S1, unofficial].
*   **Scaling Factor ($k$):** $1/2$. Although theoretically derived as $(1/2)^{1/3}$, setting $k=1/2$ produces marginally better results [S1, unofficial].
*   **Minimum Axis Length:** $1e-2$. Axes with lengths below this threshold are not split for numerical stability [S1, unofficial].

**Steps:**
1.  **Identify Bent Gaussians:** For each Gaussian, calculate the angle between vectors from the deformed center $c'_d$ to two deformed points $x'_1$ and $x'_2$ on the same axis (e.g., x-axis) in the deformed point cloud $AP_d$.
    *   Vector 1: $\vec{v}_1 = x'_1 - c'_d$
    *   Vector 2: $\vec{v}_2 = x'_2 - c'_d$
    *   Angle $\theta = \arccos\left(\frac{\vec{v}_1 \cdot \vec{v}_2}{\|\vec{v}_1\| \|\vec{v}_2\|}\right)$
    *   If $\theta < 175^\circ$, the Gaussian is considered bent and requires splitting on this axis [S1, unofficial].

2.  **Split the Ellipsoid:** Divide the pre-MVC Gaussian ellipsoid into two smaller ellipsoids along the chosen axis.
    *   Each new ellipsoid has axis lengths reduced by factor $k$.
    *   Both new ellipsoids are positioned to meet at the center of the original ellipsoid along the splitting axis.
    *   Specifically, if splitting along the x-axis with original semi-axis $a$, the new semi-axes are $ka, b, c$. The centers are shifted by $\pm (a - ka)/2$ along the x-axis relative to the original center [S1, unofficial].

3.  **Apply MVC Transformation:** Apply the MVC transformation again to the resulting two smaller ellipsoids for further processing [S1, unofficial].

4.  **Repeat for All Axes:** Run the identification and splitting process across all axes (x, y, z) to ensure Gaussians requiring splitting on multiple axes are split more than once [S1, unofficial].

**Formulas:**
*   **Volume Preservation Constraint:** The combined volume of two smaller ellipsoids approximates the original volume.
    $$ L_V = \frac{2}{3}\pi abc - 2 \frac{2}{3}\pi k_1 a k_2 b k_3 c \approx 0 $$
    Simplified: $k_1 k_2 k_3 = 1/2$ [S1, unofficial].
*   **Shape Preservation Constraint:** Each smaller ellipsoid maintains shape similar to original.
    $$ \frac{k_1}{a} \approx \frac{k_2}{b} \quad \text{and} \quad \frac{k_1}{a} \approx \frac{k_3}{c} $$
    (Note: The source text implies $k_1/a \approx k_2/b$ and $k_1/a \approx k_3/c$ are constraints to maintain shape similarity, leading to $k_1=k_2=k_3$) [S1, unofficial].
*   **Scaling Factor:**
    $$ k_1 = k_2 = k_3 = \sqrt[3]{\frac{1}{2}} $$
    In practice, $k = 1/2$ is used [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def split_bent_gaussians(gaussians, threshold=175.0, k=0.5, min_axis=1e-2):
    """
    Splits bent Gaussians into multiple well-formed ellipsoids.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - center: 3D vector
            - axes: List of 3D vectors representing semi-principal axes (a, b, c)
            - proxy_points: List of 3D points representing the deformed ellipsoid
        threshold: Angle threshold in degrees for splitting (default 175)
        k: Scaling factor for splitting (default 0.5)
        min_axis: Minimum axis length to avoid splitting (default 1e-2)
        
    Returns:
        List of Gaussian objects (split Gaussians replace originals)
    """
    new_gaussians = []
    
    for g in gaussians:
        # Check if splitting is needed for any axis
        split_needed = False
        split_axes = []
        
        for i, axis in enumerate(g.axes):
            axis_len = np.linalg.norm(axis)
            if axis_len < min_axis:
                continue
                
            # Get deformed points for this axis from proxy_points
            # Assuming proxy_points contains center and two points along each axis
            # This is a simplification; actual implementation depends on proxy point structure
            center = g.center
            # Find points along the current axis direction
            # In practice, these are pre-computed deformed points x'_1, x'_2
            p1, p2 = get_axis_points(g.proxy_points, i)
            
            if p1 is None or p2 is None:
                continue
                
            # Calculate angle
            v1 = p1 - center
            v2 = p2 - center
            cos_angle = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
            # Clamp for numerical stability
            cos_angle = np.clip(cos_angle, -1.0, 1.0)
            angle = np.degrees(np.arccos(cos_angle))
            
            if angle < threshold:
                split_needed = True
                split_axes.append(i)
        
        if not split_needed:
            new_gaussians.append(g)
            continue
            
        # Split the Gaussian
        # Create two new Gaussians
        g1 = copy_gaussian(g)
        g2 = copy_gaussian(g)
        
        for axis_idx in split_axes:
            # Reduce axis length by factor k
            g1.axes[axis_idx] = g1.axes[axis_idx] * k
            g2.axes[axis_idx] = g2.axes[axis_idx] * k
            
            # Shift centers to meet at original center
            # Original center is midpoint between new centers
            shift = (1 - k) / 2 * g.axes[axis_idx] # Original axis vector
            g1.center = g.center + shift
            g2.center = g.center - shift
            
        # Add split Gaussians to new list
        new_gaussians.append(g1)
        new_gaussians.append(g2)
        
    return new_gaussians
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians and axes checked. Splitting increases the number of Gaussians, potentially doubling the count for heavily bent regions.
*   **Memory Complexity:** Increases proportionally to the number of splits.
*   **Strengths:** Prevents visual artifacts from non-rigid warping; no architectural changes or retraining needed; can work in real-time with caching [S1, unofficial].
*   **Limitations:** Increases the number of primitives, which may impact rendering performance if many splits occur.
*   **When to use:** When deforming 3DGS Gaussians using cage-based deformation or other methods that cause non-rigid warping, leading to bent Gaussians.
*   **When not to use:** When deformation is rigid or minimal, as splitting adds unnecessary complexity.

## Background
Originated in GSDeformer: Direct Cage-based Deformation for 3D Gaussian Splatting [S1, unofficial]. The method was developed to address artifacts in 3DGS deformation caused by non-rigid warping, where a single Gaussian cannot represent bent shapes. It operates directly on the primitives of 3DGS, allowing integration with other 3DGS variants without retraining [S1, unofficial].

## Sources
[S1] GSDeformer: Direct Cage-based Deformation for 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2405.15491, fetched 2026-10-30 (unofficial)
