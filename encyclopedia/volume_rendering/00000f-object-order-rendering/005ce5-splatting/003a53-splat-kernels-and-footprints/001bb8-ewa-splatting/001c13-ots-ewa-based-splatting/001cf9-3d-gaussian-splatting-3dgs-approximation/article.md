# 3D Gaussian Splatting (3DGS) Approximation

Also called: 3D Gaussian Splatting (3DGS)

## Overview
3D Gaussian Splatting (3DGS) is a splatting method that uses unnormalized 2D Gaussians to represent scene primitives, preserving projected amplitude (opacity) across projections rather than preserving weight as in EWA splatting [S1]. It differs from its parent [OTS (EWA-Based Splatting)](1181-ots-ewa-based-splatting.md) by simplifying the volumetric integration: it assumes uniform color per Gaussian, ignores perspective distortion via affine projection, and approximates attenuation using a first-order Taylor expansion ($e^x \approx 1-x$) [S1]. This allows for real-time rendering but prevents the definition of a corresponding 3D extinction function in the established volume integration framework [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N$ 3D Gaussians. Each Gaussian $i$ is defined by:
    *   Mean position $\mathbf{X}_{0,i}$ in world space.
    *   Rotation matrix $\mathbf{R}_i$ and scaling matrix $\mathbf{S}_i$, defining the 3D covariance $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$ [S2].
    *   Opacity parameter $\sigma_i$ (used to derive splatting opacity $\alpha_i$) [S2].
    *   Spherical Harmonics (SH) coefficients for view-dependent color $c_i$ [S2].
*   Camera parameters: View matrix $\mathbf{V}$, Projection matrix $\mathbf{P}$, and intrinsics.
*   Image resolution (pixel coordinates $\mathbf{p}$).

**Outputs:**
*   Final pixel color $c_{\text{pixel}}$ for each pixel in the image plane.

### Data Structures
Each Gaussian $i$ stores:
1.  **Position:** $\mathbf{X}_{0,i} \in \mathbb{R}^3$.
2.  **Covariance:** Defined implicitly by $\mathbf{R}_i \in \mathbb{R}^{3\times3}$ (orthonormal) and $\mathbf{S}_i \in \mathbb{R}^{3\times3}$ (diagonal, positive scales) [S2].
3.  **Opacity:** Scalar $\sigma_i \in [0, 1]$ [S2].
4.  **Color:** SH coefficients (typically degree 3, 48 floats) [S2].

### Steps

1.  **3D to 2D Projection:**
    For each Gaussian $i$, compute the 2D mean $\boldsymbol{\mu}'_i$ and covariance $\boldsymbol{\Sigma}'_i$ in screen space.
    *   Transform $\mathbf{X}_{0,i}$ to camera space and project to screen space to get $\boldsymbol{\mu}'_i$ [S1].
    *   Compute the projected 2D covariance $\boldsymbol{\Sigma}'_i$. Since perspective projection is non-linear, an affine approximation is used (as in EWA) [S1]. The 2D Gaussian is the marginal distribution of the 3D Gaussian along the view axis [S1].
    *   The projected 2D Gaussian function is unnormalized:
        $$ o_i(\mathbf{p}) = \mathcal{G}_2^u(\mathbf{p}, a'_i, \boldsymbol{\mu}'_i, \boldsymbol{\Sigma}'_i) $$
        where $a'_i$ is the projected amplitude (opacity) [S1]. Note: Unlike EWA which preserves weight $w$, 3DGS preserves amplitude $a'$ [S1].

2.  **Splatting Opacity Estimation:**
    For a pixel $\mathbf{p}$, the splatting opacity $\alpha_i$ for Gaussian $i$ is estimated based on the distance between $\mathbf{p}$ and the Gaussian's center $\boldsymbol{\mu}'_i$ using the unnormalized Gaussian function [S2].
    $$ \alpha_i = o_i(\mathbf{p}) $$
    (Clamped to $[0, 1]$ if necessary, though the approximation handles this implicitly in the blending formula) [S1, S2].

3.  **Color Evaluation:**
    The color $c_i$ for Gaussian $i$ is evaluated using its Spherical Harmonics coefficients at the viewing direction relative to the mean $\boldsymbol{\mu}'_i$ [S1, S2]. Color is assumed uniform within the Gaussian footprint [S1].

4.  **Alpha Blending (Approximate Volume Integration):**
    Gaussians are sorted by depth (distance from camera) [S1]. The final pixel color is computed using alpha blending with the first-order Taylor expansion approximation for attenuation [S1]:
    $$ c_{\text{pixel}} = \sum_{i=1}^{N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    This approximates the integral:
    $$ I(\mathbf{p}) = \sum_{i=0}^N c_i(\mathcal{R})g_i(\mathbf{p}) \prod_{j=0}^{i-1} (1 - g_j(\mathbf{p})) + c_b \prod_{i=0}^N (1 - g_i(\mathbf{p})) $$
    where $g_i(\mathbf{p})$ is the unnormalized Gaussian value [S1]. The term $c_b$ is the background color, often assumed black or white [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_3dgs(gaussians, camera, image_width, image_height):
    """
    Render a scene using 3D Gaussian Splatting approximation.
    
    Args:
        gaussians: List of Gaussian objects. Each has:
            - pos: (3,) world position
            - R: (3,3) rotation matrix
            - S: (3,3) scaling matrix (diagonal)
            - opacity: scalar sigma
            - sh_coeffs: array for spherical harmonics
        camera: Object with view_matrix, proj_matrix, intrinsics.
        image_width, image_height: int
        
    Returns:
        image: (H, W, 3) float32 array
    """
    image = np.zeros((image_height, image_width, 3), dtype=np.float32)
    
    # Precompute camera matrices
    view_proj = camera.proj_matrix @ camera.view_matrix
    
    for i, g in enumerate(gaussians):
        # 1. Project 3D Gaussian to 2D screen space
        # Transform mean to camera space
        pos_cam = camera.view_matrix @ np.append(g.pos, 1.0)
        pos_cam = pos_cam[:3] / pos_cam[3] # Perspective divide
        
        # Project to screen
        pos_screen = camera.proj_matrix @ np.append(pos_cam, 1.0)
        pos_screen = pos_screen[:2] / pos_screen[3]
        
        # Compute 2D covariance Sigma'
        # J is the Jacobian of the projection transform at the mean
        # Sigma_3d = R @ S @ S.T @ R.T
        sigma_3d = g.R @ (g.S * g.S.T) @ g.R.T
        # J is derived from camera intrinsics and position
        # For affine approximation, J is constant for the Gaussian
        J = compute_projection_jacobian(camera, pos_cam)
        sigma_2d = J[:2, :2] @ sigma_3d @ J[:2, :2].T
        
        # 2. Determine which pixels are covered
        # Compute bounding box of the 2D Gaussian
        # Use 3-sigma rule for bounding box
        eigenvalues, eigenvectors = np.linalg.eigh(sigma_2d)
        max_extent = 3.0 * np.sqrt(np.max(eigenvalues))
        
        min_x = int(max(0, pos_screen[0] - max_extent))
        max_x = int(min(image_width, pos_screen[0] + max_extent))
        min_y = int(max(0, pos_screen[1] - max_extent))
        max_y = int(min(image_height, pos_screen[1] + max_extent))
        
        # 3. Splat to pixels
        for y in range(min_y, max_y):
            for x in range(min_x, max_x):
                # Pixel center in normalized device coordinates or screen space
                # Assuming pos_screen is in pixel coordinates for simplicity
                # Adjust if pos_screen is in NDC [-1, 1]
                pixel_pos = np.array([x, y])
                
                # Compute unnormalized Gaussian value at pixel
                diff = pixel_pos - pos_screen
                # alpha = exp(-0.5 * diff.T @ Sigma_inv @ diff)
                # But 3DGS uses unnormalized Gaussian for alpha in blending
                # alpha = exp(...) * amplitude
                # However, the formula in S1/S2 implies alpha is the value of the 
                # unnormalized Gaussian function itself, often scaled by opacity sigma
                # S2 says: alpha_i estimated based on Gaussian-to-Pixel distance
                # S1 says: o_i(p) = G_2^u(...)
                
                # Compute Mahalanobis distance
                try:
                    sigma_inv = np.linalg.inv(sigma_2d)
                    dist_sq = diff @ sigma_inv @ diff
                except np.linalg.LinAlgError:
                    continue # Singular covariance
                
                # Unnormalized Gaussian value
                # G^u(p) = exp(-0.5 * dist_sq)
                # The amplitude a'_i is related to the 3D opacity sigma_i
                # S2 uses alpha_i directly in blending. 
                # S1 uses g_i(p) in blending.
                # Typically, alpha_i = sigma_i * exp(-0.5 * dist_sq)
                # But S1 says "unnormalized Gaussians that preserves 2D amplitude"
                # Let's assume alpha_i = sigma_i * exp(-0.5 * dist_sq)
                # Note: S1 eq 13 defines o_i(p) as the unnormalized Gaussian.
                # S2 eq 2 uses alpha_i in blending.
                # We assume alpha_i = o_i(p)
                
                alpha = np.exp(-0.5 * dist_sq)
                alpha *= g.opacity # Scale by 3D opacity parameter
                
                # Clamp alpha to [0, 1] to prevent numerical issues
                alpha = np.clip(alpha, 0.0, 1.0)
                
                # Evaluate color from SH
                # View direction from camera to Gaussian mean
                view_dir = pos_cam / np.linalg.norm(pos_cam)
                color = evaluate_sh(g.sh_coeffs, view_dir)
                
                # Alpha blending
                # c_pixel = c_i * alpha_i * prod(1 - alpha_j)
                # We need to accumulate this. 
                # Since we are iterating Gaussians, we can update the image buffer
                # But standard splatting requires sorting by depth.
                # For simplicity, we assume the loop order is sorted or we use a 
                # depth-sorted list. Here, we assume gaussians are pre-sorted.
                
                # Accumulate into image
                # c_new = c_old * (1 - alpha) + c_i * alpha
                # This is equivalent to the product formula if processed in order
                image[y, x] = image[y, x] * (1 - alpha) + color * alpha
                
    return image

def compute_projection_jacobian(camera, pos_cam):
    """
    Compute the Jacobian of the projection transform at pos_cam.
    Simplified for perspective projection.
    """
    fx = camera.intrinsics[0, 0]
    fy = camera.intrinsics[1, 1]
    cx = camera.intrinsics[0, 2]
    cy = camera.intrinsics[1, 2]
    
    z = pos_cam[2]
    if z <= 0:
        return np.eye(2)
        
    # Jacobian of perspective projection (x/z, y/z)
    # dx/dx = fx/z, dx/dz = -fx*x/z^2
    # dy/dy = fy/z, dy/dz = -fy*y/z^2
    
    x = pos_cam[0]
    y = pos_cam[1]
    
    J = np.zeros((2, 3))
    J[0, 0] = fx / z
    J[0, 2] = -fx * x / (z * z)
    J[1, 1] = fy / z
    J[1, 2] = -fy * y / (z * z)
    
    return J

def evaluate_sh(sh_coeffs, view_dir):
    """
    Evaluate Spherical Harmonics color from view direction.
    Simplified for degree 0 (constant) for brevity.
    """
    # In practice, this involves projecting view_dir onto SH basis functions
    # and dotting with coefficients.
    # For degree 0, it's just the first coefficient.
    return sh_coeffs[:3] # RGB
```

### Parameters and Typical Values
*   **SH Degree:** Typically 3 (48 coefficients per Gaussian) [S2].
*   **Bounding Box:** 3-sigma rule for culling [S1, S2].
*   **Opacity Clamping:** $\alpha_i \in [0, 1]$ [S2].

### Edge Cases and Pitfalls
*   **Singular Covariance:** If $\boldsymbol{\Sigma}'_i$ is singular (e.g., flat Gaussian viewed edge-on), inversion fails. Guard against this [S1].
*   **Perspective Distortion:** The affine approximation ignores perspective skew, which can cause artifacts for large Gaussians or extreme angles [S1].
*   **No 3D Extinction:** The method does not define a 3D density function, making it incompatible with standard volume rendering integrators [S1].
*   **Sorting:** Correct alpha blending requires Gaussians to be sorted by depth for each pixel. This is computationally expensive if done per-pixel. Often, a global depth sort is used as an approximation [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** $O(N \cdot A)$ where $N$ is the number of Gaussians and $A$ is the average number of Gaussians splatting to a pixel. Sorting adds $O(N \log N)$ globally or $O(N \log N)$ per pixel if done locally.
*   **Memory:** $O(N)$ to store Gaussian parameters.

### Measured Performance
*   3DGS achieves real-time rendering performance on modern GPUs [S2].
*   High fidelity compared to neural radiance fields (NeRFs) [S2].

### Strengths and Limitations
*   **Strengths:** Fast real-time rendering, high visual fidelity, explicit scene representation [S2].
*   **Limitations:** Large number of Gaussians required, memory intensive for high-res scenes, artifacts from affine projection approximation [S1].

### When to Use
*   Real-time rendering of static or dynamic scenes with high fidelity [S2].
*   When a compact, explicit representation is needed [S2].

### When Not to Use
*   When accurate volumetric integration is required (e.g., for physics-based lighting) [S1].
*   When memory is extremely constrained [S2].

## Background

### Origin
3D Gaussian Splatting was introduced by Kerbl et al. in "A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Dynamic Scenes" (2023) [S2]. It builds on the EWA splatting framework [S1].

### History
The method was developed to address the slow rendering speeds of NeRFs while maintaining high quality. It uses Gaussians as primitives, leveraging the efficiency of splatting [S2].

### Usage
Used in real-time rendering, 3D reconstruction, and dynamic scene representation [S2].

## Sources

[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] Gaussian On-the-Fly Splatting: A Progressive Framework for Robust Near Real-Time 3DGS Optimization, https://doi.org/10.1109/lra.2025.3632729, fetched 2026-10-30
