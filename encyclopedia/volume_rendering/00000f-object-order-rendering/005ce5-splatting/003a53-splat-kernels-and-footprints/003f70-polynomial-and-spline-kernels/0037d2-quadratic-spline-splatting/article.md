# Quadratic Spline Splatting
Also called: Max, Crawfis and Max, Quadratic Spline Splatting (Max

## Overview
Quadratic Spline Splatting replaces the standard Gaussian reconstruction kernel with a piecewise quadratic spline function to improve rendering accuracy under perspective projection [S1, unofficial]. It belongs to the polynomial and spline kernel family, sharing compact support for exact bounding-box culling and piecewise continuity properties [G].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (voxel values and positions), camera parameters (view and projection matrices), and kernel parameters [G].
*   **Output:** A 2D rasterized image [G].

**Data Structures:**
*   **Splat Kernel:** A radial function $W(r)$ defined by a quadratic polynomial within a finite support radius $R$ [G].
*   **Footprint:** An elliptical region in screen space determined by the projected 2D covariance (conic) matrix [G].

**Method:**
1.  **Covariance Transformation:** For each voxel, compute the 3D covariance matrix $\Sigma$ and transform it into a 2D conic matrix $\mathbf{C}$ in screen space to define the elliptical footprint [G].
2.  **Footprint Culling:** Calculate the bounding box of the elliptical footprint and cull voxels whose footprints do not contribute to the final image [S1, unofficial].
3.  **Rasterization:** For each pixel within the footprint, compute the squared Mahalanobis distance $d_M^2$ from the splat center using the conic matrix [G].
4.  **Kernel Evaluation:** Evaluate the quadratic spline kernel $W(d_M)$ based on the computed distance [G].
5.  **Compositing:** Accumulate the weighted voxel values into the pixel buffer using alpha blending or similar compositing techniques [G].

**Formulas:**
The quadratic spline kernel $W(r)$ is defined piecewise:
$$ W(r) = \begin{cases} P_1(r) & \text{if } r < R_{\text{inner}} \\ P_2(r) & \text{if } R_{\text{inner}} \le r < R \\ 0 & \text{otherwise} \end{cases} $$
where $P_1$ and $P_2$ are quadratic polynomials optimized for certain conditions to ensure continuity and proper normalization [S1, unofficial].

**Parameters:**
*   **Support Radius ($R$):** The radius beyond which the kernel value is zero [G].
*   **Inner Radius ($R_{\text{inner}}$):** The transition point between the two quadratic pieces [G].

**Edge Cases and Pitfalls:**
*   **Perspective Distortion:** The kernel is optimized for perspective projection, requiring accurate covariance transformation to avoid artifacts [G].
*   **Discretization:** Aliasing may occur if the footprint is undersampled [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def project_to_screen(center, cam):
    return cam * center  # Standard projection [G]

def transform_covariance_to_screen(cov, cam):
    return cam * cov * cam.T  # Standard covariance transform [G]

def compute_ellipse_bbox(center, C):
    a, b, c = C[0][0], C[0][1], C[1][1]
    trace = a + c
    det = a * c - b * b
    disc = math.sqrt(max(0, trace * trace - 4 * det))
    l1 = (trace + disc) / 2
    l2 = (trace - disc) / 2
    s1 = 1.0 / math.sqrt(l1)
    s2 = 1.0 / math.sqrt(l2)
    v1x, v1y = b, l1 - a
    v2x, v2y = b, l2 - a
    len1 = math.sqrt(v1x**2 + v1y**2)
    len2 = math.sqrt(v2x**2 + v2y**2)
    if len1 > 0: v1x /= len1; v1y /= len1
    if len2 > 0: v2x /= len2; v2y /= len2
    dx = abs(v1x * s1) + abs(v2x * s2)
    dy = abs(v1y * s1) + abs(v2y * s2)
    return type('BBox', (), {'x_min': center.x - dx, 'x_max': center.x + dx, 'y_min': center.y - dy, 'y_max': center.y + dy})()

def quadratic_spline_kernel(r, R_inner, R_outer, a, b, c, d, e, f):
    if r >= R_outer:
        return 0.0
    if r < R_inner:
        return a * r**2 + b * r + c
    return d * r**2 + e * r + f

def render_splat(voxel, camera_matrix, image_width, image_height, R_inner, R_outer, a, b, c, d, e, f):
    screen_center = project_to_screen(voxel.center, camera_matrix)
    conic_matrix = transform_covariance_to_screen(voxel.covariance, camera_matrix)
    bbox = compute_ellipse_bbox(screen_center, conic_matrix)
    y_min = max(0, bbox.y_min)
    y_max = min(image_height, bbox.y_max)
    x_min = max(0, bbox.x_min)
    x_max = min(image_width, bbox.x_max)
    if x_min >= x_max or y_min >= y_max:
```
