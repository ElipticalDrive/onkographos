# Bounding Box Culling

Also called: 2D Gaussian Bounding Box Culling, Bounding Box Culling and Pre-processing, Viewport Culling via AABB

## Overview
Bounding Box Culling is a view-dependent pre-processing step that filters out 3D Gaussians whose projected 2D footprint does not intersect the camera viewport [S1, unofficial]. It differs from general [AABB Culling](1789-aabb-culling.md) by operating on the projected 2D ellipse of a Gaussian rather than a static 3D axis-aligned box, thereby enabling early rejection of primitives that are off-screen in screen space [S1, unofficial].

## How it works
**Inputs:**
- A list of 3D Gaussians, each defined by a 3D center position $\mathbf{c}$ and a $3 \times 3$ covariance matrix $\Sigma$ [S1, unofficial].

**Outputs:**
- A buffer of alive primitives containing packed major/minor axes, depth, and color/opacity data [S1, unofficial].

**Data Structures:**
1.  **Alive Primitive Buffer:**
    -   **Major/Minor Axes:** Packed into two 32-bit unsigned integers. Each 32-bit integer stores a 2D vector component pair using 16-bit floats (half-precision) per component [S1, unofficial].
    -   **Depth:** Stored as a 32-bit floating-point value in a dedicated buffer [S1, unofficial].
    -   **Color/Opacity:** Packed into a single 32-bit unsigned integer using RGBA8 format (8 bits per channel) [S1, unofficial].

**Steps:**
1.  **Projection:** For each Gaussian, project its 3D center $\mathbf{c}$ and covariance matrix $\Sigma$ into 2D screen space to yield a 2D ellipse [S1, unofficial].
2.  **Axis Calculation:** Calculate the major and minor axes of the projected 2D ellipse [S1, unofficial].
3.  **AABB Calculation:** Compute an axis-aligned bounding box (AABB) for the 2D ellipse [S1, unofficial].
4.  **Culling Test:** Drop the Gaussian if its bounding box does not intersect the camera viewport [S1, unofficial].
5.  **Depth Storage:** For alive primitives, calculate the depth and store it as a 32-bit floating point in a dedicated buffer [S1, unofficial].
6.  **Color Evaluation:** Evaluate Spherical Harmonics (SH) coefficients based on the current viewing direction to compute the final RGB color [S1, unofficial].
7.  **Axis Packing:** Compress and pack the major and minor axis vectors into a pair of 32-bit unsigned integers, using 16-bit floats for each vector component [S1, unofficial].
8.  **Color Packing:** Compress and pack the resulting floating-point color and the splat’s opacity into a single 32-bit unsigned integer using an RGBA8 format [S1, unofficial].
9.  **Write:** Write the alive primitives to the next stage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_box_culling(gaussians, view_proj_matrix, viewport_min, viewport_max):
    alive_primitives = []
    depth_buffer = []

    for g in gaussians:
        # 1. Project 3D center and covariance to 2D screen space
        w = view_proj_matrix[2][0]*g.center[0] + view_proj_matrix[2][1]*g.center[1] + view_proj_matrix[2][2]*g.center[2] + view_proj_matrix[2][3]
        clip_c = [
            view_proj_matrix[0][0]*g.center[0] + view_proj_matrix[0][1]*g.center[1] + view_proj_matrix[0][2]*g.center[2] + view_proj_matrix[0][3],
            view_proj_matrix[1][0]*g.center[0] + view_proj_matrix[1][1]*g.center[1] + view_proj_matrix[1][2]*g.center[2] + view_proj_matrix[1][3],
            w
        ]
        screen_c = [clip_c[0] / w, clip_c[1] / w]

        # 2. Compute projection Jacobian and screen covariance [G]
        f = 1.0 / w
        f2 = f * f
        J = [
            [view_proj_matrix[0][0]*f - view_proj_matrix[0][2]*g.center[0]*f2, view_proj_matrix[0][1]*f - view_proj_matrix[0][2]*g.center[1]*f2, -view_proj_matrix[0][2]*g.center[2]*f2],
            [view_proj_matrix[1][0]*f - view_proj_matrix[1][2]*g.center[0]*f2, view_proj_matrix[1][1]*f - view_proj_matrix[1][2]*g.center[1]*f2, -view_proj_matrix[1][2]*g.center[2]*f2]
        ]
        screen_cov = mat_mul(mat_mul(J, g.covariance), transpose(J))

        # 3. Calculate major and minor axes of the projected 2D ellipse [G]
        eigenvalues, eigenvectors = eigh_2x2(screen_cov)
        major_axis = [eigenvectors[0][0] * eigenvalues[0], eigenvectors[1][0] * eigenvalues[0]]
        minor_axis = [eigenvectors[0][1] * eigenvalues[1], eigenvectors[1][1] * eigenvalues[1]]

        # 4. Compute AABB for the 2D ellipse
        extent_x = abs(major_axis[0]) + abs(minor_axis[0])
        extent_y = abs(major_axis[1]) + abs(minor_axis[1])
        aabb_min = [screen_c[0] - extent_x, screen_c[1] - extent_y]
        aabb_max = [screen_c[0] + extent_x, screen_c[1] + extent_y]

        # 5. Culling Test: Check intersection with camera viewport
        if aabb_min[0] > viewport_max[0] or aabb_max[0] < viewport_min[0] or \
           aabb_min[1] > viewport_max[1] or aabb_max[1] < viewport_min[1]:
            continue

        # 6. Calculate depth and store as 32-bit float
        depth = clip_c[2]
        depth_buffer.append(depth)

        # 7. Evaluate SH coefficients for RGB color [G]
        view_dir = normalize([view_proj_matrix[0][3] - g.center[0], view_proj_matrix[1][3] - g.center[1], view_proj_matrix[2][3] - g.center[2]])
        rgb = evaluate_sh(g.sh_coeffs, view_dir)

        # 8. Pack major/minor axes into pair of 32-bit uints (16-bit floats per component)
        packed_axes = pack_16bit_vectors(major_axis, minor_axis)

        # 9. Pack color and opacity into single 32-bit uint (RGBA8)
        packed_color = pack_rgba8(rgb, g.opacity)

        alive_primitives.append({
            'axes': packed_axes,
            'depth': depth,
            'color': packed_color,
            'screen_center': screen_c
        })

    return alive_primitives, depth_buffer

# Standard linear algebra helpers [G]
def transpose(M): return [[M[j][i] for j in range(len(M))] for i in range(len(M[0]))]
def mat_mul(A, B):
    C = [[0.0]*len(B[0]) for _ in range(len(A))]
    for i in range(len(A)):
        for j in range(len(B[0])):
            for k in range(len(B)): C[i][j] += A[i][k]*B[k][j]
    return C
def eigh_2x2(M):
    # Symmetric 2x2 eigen decomposition [G]
    trace = M[0][0] + M[1][1]
    det = M[0][0]*M[1][1] - M[0][1]*M[1][0]
    disc = trace*trace - 4*det
    if disc < 0: disc = 0
    sqrt_disc = disc**0.5
    lam1 = (trace + sqrt_disc) / 2.0
    lam2 = (trace - sqrt_disc) / 2.0
    if abs(M[0][1]) > 1e-8:
        v1 = [M[0][1], lam1 - M[0][0]]
        v2 = [M[0][1], lam2 - M[0][0]]
    else:
        v1 = [1.0, 0.0]
        v2 = [0.0, 1.0]
    norm1 = (v1[0]**2 + v1[1]**2)**0.5
    norm2 = (v2[0]**2 + v2[1]**2)**0.5
    v1 = [x/norm1 for x in v1]
    v2 = [x/norm2 for x in v2]
    return [lam1, lam2], [[v1[0], v2[0]], [v1[1], v2[1]]]
def normalize(v):
    n = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return [x/n for x in v]
def evaluate_sh(coeffs, direction):
    # Standard SH evaluation [G]
    return [max(0.0, min(1.0, sum(c*d for c,d in zip(coeffs, direction))))]
def pack_16bit_vectors(major, minor):
    def pack_2d(v):
        return (int(v[0]*65535) & 0xFFFF) | ((int(v[1]*65535) & 0xFFFF) << 16)
    return (pack_2d(major), pack_2d(minor))
def pack_rgba8(rgb, opacity):
    r = int(max(0.0, min(1.0, rgb[0])) * 255)
    g = int(max(0.0, min(1.0, rgb[1])) * 255)
    b = int(max(0.0, min(1.0, rgb[2])) * 255)
    a = int(max(0.0, min(1.0, opacity)) * 255)
    return (r << 24) | (g << 16) | (b << 8) | a
```

**Edge Cases and Pitfalls:**
-   **Precision:** The use of 16-bit floats for axis packing may introduce quantization errors for very small or very large splats [G].
-   **Degenerate Input:** If a Gaussian is projected to a point or line (degenerate ellipse), the AABB calculation must handle zero-area cases gracefully [G].
-   **Boundary Conditions:** The intersection test must correctly handle Gaussians that are partially off-screen but still contribute to the visible area [G].

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of Gaussians $O(N)$, as each Gaussian is processed independently [S1, unofficial].
-   **Memory Complexity:** Requires additional buffers for depth, packed axes, and packed color/opacity, which compresses data to significantly reduce memory bandwidth during the final rendering stage [S1, unofficial].
-   **Performance:** Significantly reduces the number of primitives passed to the Sort and Rasterization stages, improving overall rendering efficiency [S1, unofficial].
-   **Strengths:** Effective early rejection of off-screen primitives; reduces memory bandwidth for subsequent stages [S1, unofficial].
-   **Limitations:** Requires projection and ellipse axis calculation per primitive during the culling pass [S1, unofficial].

## Background
Bounding Box Culling is a key component of the pre-processing stage in WebSplatter, a system designed for efficient Gaussian Splatting in web browsers via WebGPU [S1, unofficial]. It was introduced to address the computational bottleneck of sorting and rendering large numbers of Gaussians by filtering out invisible ones early [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
