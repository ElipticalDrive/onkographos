# Quadratic Spline Splatting
Also called: Max, Crawfis and Max, Quadratic Spline Splatting (Max

## Overview
Quadratic Spline Splatting replaces the standard Gaussian reconstruction kernel with a piecewise quadratic spline function to improve rendering accuracy, particularly under perspective projection [S1, unofficial]. It distinguishes itself from Gaussian splatting by offering a compact support with finite radius, allowing for exact bounding box culling, and by providing different continuity properties compared to higher-order splines [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (voxel values), camera parameters (view and projection matrices), and the quadratic spline kernel definition.
*   **Output:** A 2D rasterized image.

**Data Structures:**
*   **Splat Kernel:** A radial function $W(r)$ defined by a quadratic polynomial within its support radius $R$.
*   **Footprint:** An elliptical region in screen space determined by the projection of the 3D covariance matrix.

**Method:**
1.  **Covariance Transformation:** For each voxel, compute the 3D covariance matrix $\Sigma$. Transform this into a 2D conic matrix $\mathbf{C}$ (inverse covariance) in screen space to define the elliptical footprint [S1, unofficial].
2.  **Footprint Culling:** Calculate the bounding box of the elliptical footprint. Cull voxels whose footprints do not intersect the view frustum or the image plane [S1, unofficial].
3.  **Rasterization:** For each pixel within the footprint, calculate the squared Mahalanobis distance $d_M^2$ from the splat center.
4.  **Kernel Evaluation:** Evaluate the quadratic spline kernel $W(d_M)$ based on the distance. The quadratic spline is typically defined piecewise, with one polynomial for the inner region and another for the outer region, ensuring continuity at the boundary [S1, unofficial].
5.  **Compositing:** Accumulate the weighted voxel values into the pixel buffer using alpha blending or similar compositing techniques [S1, unofficial].

**Formulas:**
The quadratic spline kernel $W(r)$ is generally defined as:
$$ W(r) = \begin{cases} a r^2 + b r + c & \text{if } r < R_{inner} \\ d r^2 + e r + f & \text{if } R_{inner} \le r < R \\ 0 & \text{otherwise} \end{cases} $$
Where coefficients $a, b, c, d, e, f$ are chosen to ensure continuity of value and first derivative at $R_{inner}$ and zero value/derivative at $R$ [S1, unofficial].

**Parameters:**
*   **Support Radius ($R$):** The radius beyond which the kernel value is zero. Not given in the sources.
*   **Inner Radius ($R_{inner}$):** The transition point between the two quadratic pieces. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Perspective Distortion:** The quadratic spline is optimized for perspective projection, but incorrect covariance transformation can lead to artifacts [S1, unofficial].
*   **Discretization:** Aliasing may occur if the footprint is not adequately sampled.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quadratic_spline_kernel(r, R_inner, R_outer):
    """
    Evaluates the quadratic spline kernel at distance r.
    Coefficients are assumed to be precomputed for continuity and normalization.
    """
    if r >= R_outer:
        return 0.0
    
    if r < R_inner:
        # Inner quadratic piece
        # Coefficients a, b, c derived from continuity conditions
        return a * r**2 + b * r + c
    else:
        # Outer quadratic piece
        # Coefficients d, e, f derived from continuity and zero-boundary conditions
        return d * r**2 + e * r + f

def render_splat(voxel, camera_matrix, image_width, image_height):
    """
    Renders a single voxel using quadratic spline splatting.
    """
    # 1. Transform voxel center and covariance to screen space
    screen_center, conic_matrix = transform_to_screen(voxel.center, voxel.covariance, camera_matrix)
    
    # 2. Compute footprint bounding box
    bbox = get_ellipse_bbox(screen_center, conic_matrix)
    
    # 3. Clip bbox to image bounds
    clipped_bbox = clip_to_image(bbox, image_width, image_height)
    
    # 4. Rasterize
    for y in range(clipped_bbox.y_min, clipped_bbox.y_max):
        for x in range(clipped_bbox.x_min, clipped_bbox.x_max):
            # Calculate squared Mahalanobis distance
            dx = x - screen_center.x
            dy = y - screen_center.y
            d_m_sq = dx**2 * conic_matrix[0][0] + 2 * dx * dy * conic_matrix[0][1] + dy**2 * conic_matrix[1][1]
            
            # Evaluate kernel
            r = sqrt(d_m_sq)
            weight = quadratic_spline_kernel(r, R_inner, R_outer)
            
            # Accumulate
            accumulate_pixel(x, y, voxel.value * weight)
```

## Subtopics

### [Smoothness Splines](9886-smoothness-splines.md)
Smoothness Splines (SS) is a scattered data approximation method for recovering volume rendered images from sparse pixel measurements by finding a smooth solution in a shift-invariant spline space. It minimizes a least-squares norm that penalizes non-smooth solutions using the second-order Beppo-Levi seminorm, distinguishing it from compressed sensing methods that rely on sparsity in wavelet or gradient domains. SS is more resilient to missing information and computationally less expensive than compressed sensing pursuit strategies.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of splats and pixels covered by their footprints.
*   **Memory Complexity:** Linear with respect to the number of voxels.
*   **Performance:** Faster than Gaussian splatting in some contexts due to exact culling and potentially simpler kernel evaluation [S1, unofficial].
*   **Strengths:** Improved accuracy under perspective projection; compact support allows for efficient culling [S1, unofficial].
*   **Limitations:** May exhibit different visual artifacts compared to Gaussian kernels; requires careful tuning of spline parameters [S1, unofficial].

## Background
Quadratic Spline Splatting was proposed by Max [12] and Crawfis and Max [4] as an alternative to Gaussian reconstruction kernels [S1, unofficial]. It was introduced to improve rendering accuracy, particularly when using perspective projection [S1, unofficial].

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
