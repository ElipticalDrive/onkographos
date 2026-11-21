# Optimal Gaussian Splatting Projection

Also called: Central Radial Projection, Optimal Projection, Optimal Projection Strategy, Tangent Plane Projection

## Overview
Optimal Gaussian Splatting Projection is a rendering method that minimizes projection error by projecting each 3D Gaussian onto a plane tangent to the line connecting the Gaussian mean to the camera center, rather than a fixed image plane [S1]. It sets itself apart from standard 3D-GS by using a unique tangent plane for each Gaussian based on its position relative to the camera, significantly reducing artifacts like blurriness and needle-like structures, particularly in wide-field-of-view or short focal length scenarios [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians, each defined by a mean position $\boldsymbol{\mu} \in \mathbb{R}^3$, a covariance matrix $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$, and other attributes (opacity, color) [S1]. The camera parameters (center $\mathbf{C}$, orientation, and projection model) are also required [S1].
*   **Output:** A 2D screen-space covariance matrix $\boldsymbol{\Sigma}_{2D}$ and a 2D mean position $\mathbf{m}_{2D}$ for each Gaussian, ready for rasterization [S1].

**Data Structures:**
The method requires only minor code modifications to the existing 3D-GS framework and does not introduce complex new data structures [S1]. Temporary storage is allocated for tangent plane basis vectors and projected covariances during the transformation pipeline.

**Steps:**
1.  **Compute Direction Vector:** For each Gaussian with mean $\boldsymbol{\mu}$ and camera center $\mathbf{C}$, compute the direction vector $\mathbf{d}$ from the Gaussian to the camera:
    $$ \mathbf{d} = \mathbf{C} - \boldsymbol{\mu} $$
2.  **Define Tangent Plane:** Construct a projection plane that is perpendicular to $\mathbf{d}$ and passes through $\boldsymbol{\mu}$ [S1]. This plane is tangent to the line connecting $\boldsymbol{\mu}$ and $\mathbf{C}$ [S1].
3.  **Project to Tangent Plane:** Project the 3D Gaussian onto this specific tangent plane instead of the standard z=1 plane [S1]. The covariance is rotated into the tangent plane's local orthonormal basis, and the top-left $2 \times 2$ block yields the 2D covariance [G].
4.  **Transform to Image/Camera Space:** Transform the projected Gaussian from the tangent plane back to the standard image or camera coordinate system for rasterization [S1]. This requires computing the Jacobian of the perspective projection from the tangent plane to the image plane [G].
5.  **Stabilize Covariance:** Apply covariance stabilization (e.g., adding a small epsilon $\epsilon$ to the diagonal) to ensure positive definiteness, as described in the parent method [1198-2d-gaussian-projection-with-covariance-stabilization.md] [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimal_gaussian_projection(gaussian_means, gaussian_covariances, camera_center, camera_matrix):
    """
    Projects 3D Gaussians onto tangent planes to minimize projection error.
    
    Args:
        gaussian_means: List of (N, 3) arrays, positions of N Gaussians in camera space.
        gaussian_covariances: List of (N, 3, 3) arrays, covariances of N Gaussians.
        camera_center: List of (3,) arrays, camera center position C.
        camera_matrix: List of (3, 3) arrays, intrinsic camera matrix K.
        
    Returns:
        means_2d: List of (N, 2) arrays, projected 2D means.
        covs_2d: List of (N, 2, 2) arrays, projected 2D covariances.
    """
    N = len(gaussian_means)
    means_2d = [[0.0, 0.0] for _ in range(N)]
    covs_2d = [[[0.0, 0.0], [0.0, 0.0]] for _ in range(N)]

    for i in range(N):
        mu = gaussian_means[i]
        Sigma = gaussian_covariances[i]
        C = camera_center[i]

        # Step 1: Direction from Gaussian mean to camera center
        d = [C[0]-mu[0], C[1]-mu[1], C[2]-mu[2]]
        norm_d = (d[0]**2 + d[1]**2 + d[2]**2)**0.5
        if norm_d < 1e-8: norm_d = 1e-8
        n = [d[0]/norm_d, d[1]/norm_d, d[2]/norm_d]

        # Step 2: Construct orthonormal basis (u, v) for tangent plane perpendicular to n
        ref = [0.0, 0.0, 1.0]
        if abs(n[2]) > 0.9: ref = [0.0, 1.0, 0.0]
        # Cross product n x ref
        u = [n[1]*ref[2]-n[2]*ref[1], n[2]*ref[0]-n[0]*ref[2], n[0]*ref[1]-n[1]*ref[0]]
        norm_u = (u[0]**2 + u[1]**2 + u[2]**2)**0.5
        if norm_u < 1e-8: norm_u = 1e-8
        u = [u[0]/norm_u, u[1]/norm_u, u[2]/norm_u]
        # Cross product n x u yields v (automatically normalized)
        v = [n[1]*u[2]-n[2]*u[1], n[2]*u[0]-n[0]*u[2], n[0]*u[1]-n[1]*u[0]]

        # Step 3: Project covariance to tangent plane
        # Rotate Sigma into basis [u, v, n] and extract top-left 2x2 block
        def mat_vec(M, vec):
            return [M[0][0]*vec[0] + M[0][1]*vec[1] + M[0][2]*vec[2],
                    M[1][0]*vec[0] + M[1][1]*vec[1] + M[1][2]*vec[2],
                    M[2][0]*vec[0] + M[2][1]*vec[1] + M[2][2]*vec[2]]
        def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

        Sigma_u = mat_vec(Sigma, u)
        Sigma_v = mat_vec(Sigma, v)
        cov_tangent = [[dot(u, Sigma_u), dot(u, Sigma_v)],
                       [dot(v, Sigma_u), dot(v, Sigma_v)]]

        # Step 4: Transform from tangent plane to image plane
        # Project mean mu to image plane (standard perspective projection)
        z = mu[2]
        z_safe = max(z, 1e-5)
        means_2d[i][0] = camera_matrix[i][0][0] * (mu[0] / z_safe) + camera_matrix[i][0][2]
        means_2d[i][1] = camera_matrix[i][1][1] * (mu[1] / z_safe) + camera_matrix[i][1][2]

        # Jacobian of perspective projection from tangent plane to image plane
        x, y = mu[0], mu[1]
        # J_proj is (2,3): [[1/z, 0, -x/z^2], [0, 1/z, -y/z^2]]
        J_proj = [[1/z_safe, 0, -x/(z_safe**2)],
                  [0, 1/z_safe, -y/(z_safe**2)]]
        # Basis vectors u, v form a (3,2) matrix
        # J_tangent = J_proj @ [u, v] -> (2,2)
        J_t = [[0.0, 0.0], [0.0, 0.0]]
        for r in range(2):
            for c in range(2):
                for k in range(3):
                    J_t[r][c] += J_proj[r][k] * ([u, v][k][c])

        # Final 2D covariance: Sigma_2D = J_t @ cov_tangent @ J_t^T
        def mat_mul_2x2(A, B):
            return [[A[0][0]*B[0][0] + A[0][1]*B[1][0], A[0][0]*B[0][1] + A[0][1]*B[1][1]],
                    [A[1][0]*B[0][0] + A[1][1]*B[1][0], A[1][0]*B[0][1] + A[1][1]*B[1][1]]]
        def mat_T_2x2(A):
            return [[A[0][0], A[1][0]], [A[0][1], A[1][1]]]

        covs_2d[i] = mat_mul_2x2(mat_mul_2x2(J_t, cov_tangent), mat_T_2x2(J_t))

        # Step 5: Stabilize covariance (add epsilon to diagonal)
        covs_2d[i][0][0] += 1e-3
        covs_2d[i][1][1] += 1e-3

    return means_2d, covs_2d
```

**Parameters:**
*   **Focal Length:** The method is tested with focal lengths scaled to 0.2 and 0.3 times the original focal length to simulate wide-field-of-view scenarios [S1].
*   **Epsilon:** A small value added to the diagonal of the 2D covariance for numerical stability (value not specified in sources) [G].

**Edge Cases and Pitfalls:**
*   **Training Time:** The method incurs a slight increase in training time due to the additional transformation step from the tangent plane to the image plane [S1].
*   **Parallel Normals:** When constructing the tangent plane basis, if the direction vector $\mathbf{d}$ is parallel to the reference vector (e.g., z-axis), the cross product becomes zero [G]. The pseudocode handles this by switching the reference vector.
*   **Division by Zero:** The perspective projection involves division by the z-coordinate [G]. The pseudocode clamps $z$ to a minimum safe value.

## Operating characteristics
*   **Time Complexity:** The algorithm's time complexity remains unchanged compared to 3D-GS because it avoids dense point sampling [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method significantly reduces blurriness and artifacts compared to the original projection method, particularly in settings with short focal lengths [S1]. It outperforms original 3D-GS across various scenes in terms of PSNR, SSIM, and LPIPS metrics when tested with focal length reduction ratios of $\times 0.2$ and $\times 0.3$ [S1].
*   **Strengths:** Reduces projection error, minimizes artifacts like needle-like structures and cloud-like large Gaussians at image edges, and is adaptable to various camera models (pinhole, fisheye, panorama) [S1].
*   **Limitations:** Slight increase in training time due to the additional transformation step [S1].

## Background
*   **Origin:** The method was proposed in the paper "On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy" [S1].
*   **History:** It was developed to address the issue of projection errors in 3D Gaussian Splatting (3D-GS) [17], which tend to escalate with the expansion of the field of view and reduction in focal length [S1].
*   **Usage:** Implemented in PyTorch based on 3D-GS [17] with custom CUDA kernels for rasterization [S1]. Tested on 13 real scenes from the 3D-GS dataset, including Mip-NeRF360 [2], Tanks & Temples [18], and others [S1].

## Subtopics

### Adaptive Bounding Box Computation
Computes the bounding box for each Gaussian splat to optimize rasterization. [Adaptive Bounding Box Computation](1206-adaptive-bounding-box-computation.md)

### Render Means
Renders the final image using the projected Gaussians. [Render Means](2070-render-means.md)

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
[S2] Fast and Robust 3D Gaussian Splatting for Virtual Reality, https://doi.org/10.1145/3681756.3697947, fetched 2026-10-30
