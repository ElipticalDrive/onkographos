# Suggestive Contour Extraction

Also called: Suggestive contours

## Overview
Suggestive contour extraction identifies surface locations where radial curvature is zero but increasing towards the viewer, complementing standard object contours to enhance shape perception in non-photorealistic volume rendering [S1]. Unlike standard contours which mark visible boundaries, suggestive contours reveal hidden or occluded geometry, such as the rims of indentations, by detecting where the surface is about to curve towards the camera [S1].

## How it works
**Inputs:** A scalar volume field $\phi$, an isovalue $\tau$ defining the isosurface $\phi - \tau = 0$, and a view vector $\mathbf{v}$ for each sample point.
**Outputs:** A set of line segments representing the suggestive contours.

**Data Structures:**
The method operates on a 3D grid of voxels. For each voxel $(i, j, k)$, the following values are required or computed:
*   $\phi(i, j, k)$: Scalar value.
*   $\mathbf{n}(i, j, k)$: Unit-length surface normal, computed as $\mathbf{n} = -\nabla \phi / \|\nabla \phi\|$.
*   $\mathbf{v}(i, j, k)$: View vector pointing from the surface point to the camera.
*   $s(i, j, k)$: Radial curvature proxy, defined below.

**Formulas:**
The radial curvature function $s$ is defined as:
$$ s(i, j, k) = \nabla \hat{\mathbf{n}}(i, j, k) \cdot \mathbf{v}(i, j, k) \cdot \mathbf{v}(i, j, k) $$
where $\hat{\mathbf{n}}$ is the unit-length normal and $\mathbf{v}$ is the view vector. The sign of $s$ corresponds to the sign of the radial curvature [S1].

The directional derivative of $s$ towards the viewer, denoted $D_s$, is computed numerically.

**Algorithm Steps:**
1.  **Sample Selection:** Identify candidate points on the isosurface. This is typically done via random probing or gradient descent from previous frame seeds [S1].
2.  **Zero-Crossing Detection:** Locate points where $s(i, j, k) \approx 0$. These are the locations where radial curvature is zero.
3.  **Derivative Test (Frontfaces):** For points where the surface normal $\mathbf{n}$ points towards the camera (frontfacing), check if the directional derivative of $s$ towards the viewer is positive. In practice, compare $D_s$ to a small positive threshold $\epsilon$:
    $$ D_s > \epsilon $$
    This filters out spurious zero crossings caused by noise [S1].
4.  **Derivative Test (Backfaces):** For points where the surface normal $\mathbf{n}$ points away from the camera (backfacing), negate the sign of the derivative test to maintain analogous lines to frontfacing contours:
    $$ D_s < -\epsilon $$
    Note: Previous work assumed opaque surfaces and did not address backfacing suggestive contours [S1].
5.  **Interior Contour Filtering (Optional):** To omit contours that are never visible from outside the surface (e.g., the bottom of an indentation), test if $s < 0$ for extracted segments. If $s < 0$, the segment is an interior contour and may be omitted [S1].
6.  **Line Generation:** Connect valid points into line segments or strips.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_suggestive_contours(volume, isovalue, view_direction, epsilon=1e-6):
    """
    Extracts suggestive contours from a volume dataset.
    
    Args:
        volume: 3D array of scalar values.
        isovalue: Threshold for isosurface extraction.
        view_direction: 3D vector pointing from object to camera.
        epsilon: Threshold for derivative test.
        
    Returns:
        List of line segments (pairs of 3D points).
    """
    segments = []
    # 1. Identify points near the isosurface
    # In practice, this involves marching cubes or ray casting to find
    # surface points p where |volume(p) - isovalue| < tolerance.
    surface_points = find_isosurface_points(volume, isovalue)
    
    for p in surface_points:
        # 2. Compute normal and view vector at point p
        # Normal is negative gradient of volume
        grad_phi = compute_gradient(volume, p)
        n_hat = normalize(-grad_phi)
        
        # View vector v points from surface to camera
        # Assuming camera is at origin or known position, v = camera_pos - p
        v = normalize(view_direction) 
        
        # 3. Compute radial curvature proxy s
        # s = (grad(n_hat) . v) . v
        # Note: grad(n_hat) is the Jacobian of the normal field.
        # In discrete implementation, this is often approximated or 
        # s is computed via finite differences of the normal field along v.
        s = compute_radial_curvature_proxy(volume, p, n_hat, v)
        
        # 4. Check for zero crossing (radial curvature ~ 0)
        if abs(s) < tolerance:
            # 5. Compute directional derivative of s towards viewer (Ds)
            Ds = compute_derivative_of_s(volume, p, v)
            
            # 6. Apply derivative test based on facing
            if dot(n_hat, v) > 0:
                # Frontfacing: Ds > epsilon
                if Ds > epsilon:
                    segments.append(p)
            else:
                # Backfacing: Ds < -epsilon
                if Ds < -epsilon:
                    segments.append(p)
                    
    # 7. Connect points into segments
    return connect_points_into_segments(segments)

def compute_radial_curvature_proxy(volume, p, n_hat, v):
    """
    Computes s = grad(n_hat) . v . v.
    This requires the gradient of the normal vector field.
    """
    # Numerical approximation of grad(n_hat)
    # This is computationally expensive if done naively.
    # Often approximated using finite differences of n_hat.
    # [G] Implementation detail: Compute n_hat at p and neighbors along v.
    # s approx (n_hat(p + delta*v) - n_hat(p - delta*v)) / (2*delta) . v
    delta = 1.0
    n_plus = compute_normal(volume, p + delta * v)
    n_minus = compute_normal(volume, p - delta * v)
    grad_n_dot_v = (n_plus - n_minus) / (2 * delta)
    return dot(grad_n_dot_v, v)

def compute_derivative_of_s(volume, p, v):
    """
    Computes the directional derivative of s towards the viewer.
    """
    # Similar to above, compute s at p and neighbors along v
    delta = 1.0
    s_plus = compute_radial_curvature_proxy(volume, p + delta * v, 
                                            compute_normal(volume, p + delta * v), v)
    s_minus = compute_radial_curvature_proxy(volume, p - delta * v, 
                                             compute_normal(volume, p - delta * v), v)
    return (s_plus - s_minus) / (2 * delta)
```

**Parameters:**
*   $\epsilon$: Small positive threshold for derivative test. Typical value not specified in sources, but described as "small" [S1].
*   Tolerance: For zero-crossing detection of $s$. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noise:** Spurious zero crossings in $s$ due to noise are filtered by the $D_s > \epsilon$ test [S1].
*   **Backfaces:** Standard contour methods often ignore backfaces. Suggestive contours require explicit handling of backfacing normals by negating the derivative test [S1].
*   **Interior Contours:** Contours at the bottom of indentations have negative radial curvature ($s < 0$) and are never visible from outside. These can be detected and omitted if desired [S1].
*   **Performance:** Computing normals and gradients via finite differences on demand is less efficient than precomputing, which would increase memory usage by 4x [S1].

## Subtopics

### Interior Contour Detection
A method to detect and omit 'interior' contours (those never visible from outside the surface, e.g., bottom of an indentation) by testing for negative radial curvature [S1]. It is chosen when the goal is to simplify the line drawing by removing hidden geometry details. The test involves checking if $s < 0$ for an extracted segment [S1]. A pitfall is that normals and gradients are computed numerically on demand using finite differences, which is less efficient than precomputing (which would cost 4x memory) [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the sampling strategy. Random probing and gradient descent are used to locate seeds [S1]. The cost of computing normals and gradients via finite differences is significant; precomputing would improve runtime but increase memory.
*   **Memory Complexity:** If normals are computed on demand, memory usage is low. Precomputing normals increases memory usage by 4x [S1].
*   **Performance:** In tests on a skull dataset, checking $64n$ points took 416 msec/frame, while checking $n$ points took 110 msec/frame, showing that fewer probes reduce cost but may miss contours [S1]. Temporal coherence (retaining previous frame contours) accelerates detection [S1].
*   **Strengths:** Complements standard contours by revealing hidden shape features, improving perceptual clarity of complex shapes [S1].
*   **Limitations:** Requires careful handling of backfaces and noise filtering. Numerical computation of derivatives can be noisy and expensive [S1].

## Background
Suggestive contours were introduced by DeCarlo et al. [2003] to complement standard contours in line drawings, producing more effective renderings of shape [S1]. The original work assumed opaque surfaces and did not address backfacing suggestive contours [S1]. The method is based on the observation that radial curvature is zero at the boundary between convex and concave regions, and a positive derivative towards the viewer indicates a region curving towards the camera [S1]. Interior contours, which have negative radial curvature, were later discussed as a refinement to omit hidden details [S1].

## Sources
[S1] Line drawings from volume data, https://doi.org/10.1145/1073204.1073222, fetched 2026-10-28
