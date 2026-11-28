# Gaussian Transformation to World Space

Also called: Camera-to-World Gaussian Transform, Gaussian Transformation to Camera Frame, World-space Gaussian Transformation

## Overview
This method transforms 3D Gaussian parameters (means, covariances, rotations, scales, and spherical harmonics coefficients) predicted or stored in camera space into world space, enabling consistent rendering across multiple views or integration with world-space scene geometry [S1]. It sets itself apart from sibling methods like [NDC Ray Transformation](905-ndc-ray-transformation.md) by operating on the Gaussian primitives themselves rather than the ray sampling coordinates, and from [Point to Camera Transformation](1660-point-to-camera-transformation.md) by handling the full anisotropic covariance and spherical harmonics rotation, not just point positions.

## How it works
**Inputs:**
*   Gaussian parameters in camera space:
    *   $\mu_{\text{cam}}$: Mean position, shape $(N, 3)$.
    *   $\Sigma_{\text{cam}}$: Covariance matrix, shape $(N, 3, 3)$ or represented via scale $\mathbf{s}_{\text{cam}}$ and rotation $q_{\text{cam}}$.
    *   $\alpha_{\text{cam}}$: Opacity, shape $(N, 1)$.
    *   $\mathbf{s}_{\text{cam}}$: Scale parameters, shape $(N, C_s)$. If $C_s=1$, Gaussians are isotropic; if $C_s>1$, they are anisotropic [S2, unofficial].
    *   $q_{\text{cam}}$: Rotation quaternion, shape $(N, 4)$, normalized [S2, unofficial].
    *   $\mathbf{c}_{\text{cam}}$: Spherical harmonics coefficients, shape $(N, C_{sh})$.
*   Camera-to-World extrinsics matrix $\mathbf{T}$, shape $(4, 4)$, composed of rotation $\mathbf{R}$ (top-left $3 \times 3$) and translation $\mathbf{t}$ (top-right $3 \times 1$) [S1].

**Outputs:**
*   Gaussian parameters in world space: $\mu_{\text{world}}$, $\Sigma_{\text{world}}$, $\alpha_{\text{world}}$, $\mathbf{s}_{\text{world}}$, $q_{\text{world}}$, $\mathbf{c}_{\text{world}}$.

**Data Structures:**
*   The Gaussian parameters are typically stored in contiguous arrays or tensors.
*   The transformation matrix $\mathbf{T}$ is a standard $4 \times 4$ affine matrix.

**Steps:**
1.  **Transform Mean:** Convert $\mu_{\text{cam}}$ to homogeneous coordinates by appending a column of ones, resulting in shape $(N, 4)$. Multiply by $\mathbf{T}$ from the left. Extract the first three coordinates to obtain $\mu_{\text{world}}$ [S1][S2, unofficial].
    $$ \mu_{\text{world}} = (\mathbf{T} \cdot [\mu_{\text{cam}}, \mathbf{1}]^T)^T_{0:3} $$
2.  **Transform Covariance:** Compute the world-space covariance $\Sigma_{\text{world}}$ using the rotation component $\mathbf{R}$ of $\mathbf{T}$ [S1].
    $$ \Sigma_{\text{world}} = \mathbf{R} \Sigma_{\text{cam}} \mathbf{R}^T $$
3.  **Transform Opacity:** Opacity is invariant under rigid body transformation [S1].
    $$ \alpha_{\text{world}} = \alpha_{\text{cam}} $$
4.  **Transform Rotation (if anisotropic):** If the Gaussians are anisotropic (determined by checking if scale parameters have more than one channel), normalize the camera-space rotation quaternions $q_{\text{cam}}$ and compose them with the camera rotation quaternion $q_{\text{cam\_rot}}$ derived from $\mathbf{R}$ [S2, unofficial]. The world-space rotation $q_{\text{world}}$ is obtained by quaternion multiplication [S2, unofficial]:
    $$ q_{\text{world}} = q_{\text{cam\_rot}} \otimes q_{\text{cam}} $$
    Note: The order of quaternion multiplication depends on the convention (global vs. local rotation). The source [S2, unofficial] implies $q_{\text{cam\_rot}}$ is applied to $q_{\text{cam}}$.
5.  **Transform Scale:** The scale parameters $\mathbf{s}_{\text{world}}$ are transformed using a block-diagonal matrix $\mathbf{D}$ composed of Wigner D-matrices corresponding to the rotation $\mathbf{R}$ [S1].
    $$ \mathbf{s}_{\text{world}} = \mathbf{D} \mathbf{R} \mathbf{s}_{\text{cam}} $$
    *Note: The formula in [S1] writes $\mathbf{s}_{\text{world}} = \mathbf{D}\mathbf{R}\mathbf{s}_{\text{cam}}$. This notation is ambiguous. Typically, scales are invariant under rotation if defined in the local frame, but spherical harmonics coefficients require rotation. The text states $\mathbf{D}$ rotates spherical harmonics coefficients. It is likely that $\mathbf{s}$ here refers to SH coefficients or the scale transformation is identity for isotropic Gaussians. Given the explicit mention of Wigner D matrices for SH, the scale $\mathbf{s}$ in the formula might be a typo for SH coefficients $\mathbf{c}$, or it implies a specific parameterization. However, following [S1] strictly:*
    If $\mathbf{s}$ represents SH coefficients, then $\mathbf{D}$ is the rotation matrix for SH. If $\mathbf{s}$ represents geometric scales, they are usually invariant. The source [S1] explicitly links $\mathbf{D}$ to Wigner D matrices for SH. We assume the formula applies to SH coefficients $\mathbf{c}$, or that $\mathbf{s}$ is a placeholder for all angular parameters. For implementation, SH coefficients are rotated by $\mathbf{D}(\mathbf{R})$. Geometric scales $\mathbf{s}_{\text{geo}}$ remain unchanged if defined in the local frame, or are transformed if defined in world space. The source [S1] formula $\mathbf{s}_{\text{world}} = \mathbf{D}\mathbf{R}\mathbf{s}_{\text{cam}}$ is likely referring to SH coefficients $\mathbf{c}$, as $\mathbf{D}$ is defined for SH. We will treat $\mathbf{s}$ in the formula as SH coefficients $\mathbf{c}$ for correctness, or note the ambiguity. Let's stick to the source's variable name but clarify the operation.
    Actually, re-reading [S1]: "$\mathbf{s}_{\text{world}} = \mathbf{D}\mathbf{R}\mathbf{s}_{\text{cam}}$. Here, DR is a block-diagonal matrix consisting of Wigner D matrices, which rotates the spherical harmonics coefficients". This strongly suggests $\mathbf{s}$ in the formula is a typo for $\mathbf{c}$ (SH coefficients). We will implement SH rotation separately.
6.  **Transform Spherical Harmonics (SH):** Rotate the SH coefficients $\mathbf{c}_{\text{cam}}$ using the Wigner D-matrix $\mathbf{D}(\mathbf{R})$ [S1].
    $$ \mathbf{c}_{\text{world}} = \mathbf{D}(\mathbf{R}) \mathbf{c}_{\text{cam}} $$
    The library `e3nn` is used to compute $\mathbf{D}(\mathbf{R})$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def gaussian_transform_to_world(mu_cam, cov_cam, alpha_cam, s_cam, q_cam, c_cam, T):
    """
    Transforms Gaussian parameters from camera space to world space.
    
    Args:
        mu_cam: Tensor of shape (N, 3), Gaussian means in camera space.
        cov_cam: Tensor of shape (N, 3, 3), Gaussian covariances in camera space.
        alpha_cam: Tensor of shape (N, 1), Gaussian opacities.
        s_cam: Tensor of shape (N, C_s), Gaussian scales. C_s=1 for isotropic.
        q_cam: Tensor of shape (N, 4), Gaussian rotation quaternions (w, x, y, z).
        c_cam: Tensor of shape (N, C_sh), Spherical Harmonics coefficients.
        T: Tensor of shape (4, 4), Camera-to-World transformation matrix.
        
    Returns:
        mu_world, cov_world, alpha_world, s_world, q_world, c_world
    """
    R = T[:3, :3]  # 3x3 Rotation matrix
    t = T[:3, 3]   # 3x1 Translation vector
    
    # 1. Transform Mean
    # Append ones to mu_cam to make it (N, 4)
    mu_cam_homo = torch.cat([mu_cam, torch.ones(mu_cam.shape[0], 1, device=mu_cam.device)], dim=1)
    # Multiply by T: (N, 4) @ (4, 4)^T or T @ (N, 4)^T
    # T is (4, 4). mu_cam_homo is (N, 4).
    # We want T @ mu_cam_homo^T for each point, resulting in (4, N).
    # Then transpose to (N, 4).
    mu_world_homo = (T @ mu_cam_homo.T).T
    mu_world = mu_world_homo[:, :3]
    
    # 2. Transform Covariance
    # Sigma_world = R * Sigma_cam * R^T
    cov_world = R @ cov_cam @ R.transpose(-1, -2)
    
    # 3. Transform Opacity
    alpha_world = alpha_cam
    
    # 4. Transform Rotation
    # Get camera rotation quaternion from R
    # Convert rotation matrix to quaternion
    q_cam_rot = matrix_to_quaternion(R)
    
    # Normalize input quaternions
    q_cam_norm = q_cam / torch.norm(q_cam, dim=1, keepdim=True)
    
    # Compose quaternions: q_world = q_cam_rot @ q_cam
    # Quaternion multiplication function needed
    q_world = quaternion_multiply(q_cam_rot, q_cam_norm)
    
    # 5. Transform Scales (Isotropic/Anisotropic)
    # If isotropic (C_s == 1), scales are invariant
    if s_cam.shape[1] == 1:
        s_world = s_cam
    else:
        # Anisotropic: Scales are defined in local frame, so they are invariant under rotation
        # unless the parameterization is different. Standard 3DGS treats scales as local.
        # However, source [S1] mentions D*R*s. This is likely for SH.
        # We assume geometric scales are invariant.
        s_world = s_cam
        
    # 6. Transform Spherical Harmonics
    # Compute Wigner D-matrix for rotation R
    # Using e3nn or similar library
    # D = wigner_D(R, degree)
    # c_world = D @ c_cam
    # For implementation, we assume a function get_wigner_D(R, degree) exists
    degree = get_sh_degree_from_coefficients(c_cam.shape[1])
    D = get_wigner_D(R, degree) # Shape (C_sh, C_sh) or block diagonal
    
    # Apply rotation to SH coefficients
    # c_cam shape (N, C_sh), D shape (C_sh, C_sh)
    c_world = c_cam @ D.T
    
    return mu_world, cov_world, alpha_world, s_world, q_world, c_world

def matrix_to_quaternion(R):
    """Convert 3x3 rotation matrix to quaternion (w, x, y, z)."""
    # Standard conversion algorithm
    tr = R.trace()
    if tr > 0:
        S = torch.sqrt(tr + 1.0) * 2
        w = 0.25 * S
        x = (R[2, 1] - R[1, 2]) / S
        y = (R[0, 2] - R[2, 0]) / S
        z = (R[1, 0] - R[0, 1]) / S
    elif (R[0, 0] > R[1, 1]) and (R[0, 0] > R[2, 2]):
        S = torch.sqrt(1.0 + R[0, 0] - R[1, 1] - R[2, 2]) * 2
        w = (R[2, 1] - R[1, 2]) / S
        x = 0.25 * S
        y = (R[0, 1] + R[1, 0]) / S
        z = (R[0, 2] + R[2, 0]) / S
    elif R[1, 1] > R[2, 2]:
        S = torch.sqrt(1.0 + R[1, 1] - R[0, 0] - R[2, 2]) * 2
        w = (R[0, 2] - R[2, 0]) / S
        x = (R[0, 1] + R[1, 0]) / S
        y = 0.25 * S
        z = (R[1, 2] + R[2, 1]) / S
    else:
        S = torch.sqrt(1.0 + R[2, 2] - R[0, 0] - R[1, 1]) * 2
        w = (R[1, 0] - R[0, 1]) / S
        x = (R[0, 2] + R[2, 0]) / S
        y = (R[1, 2] + R[2, 1]) / S
        z = 0.25 * S
    return torch.stack([w, x, y, z], dim=1)

def quaternion_multiply(q1, q2):
    """Multiply two quaternions q1 and q2 (w, x, y, z)."""
    w1, x1, y1, z1 = q1.unbind(dim=1)
    w2, x2, y2, z2 = q2.unbind(dim=1)
    w = w1*w2 - x1*x2 - y1*y2 - z1*z2
    x = w1*x2 + x1*w2 + y1*z2 - z1*y2
    y = w1*y2 - x1*z2 + y1*w2 + z1*x2
    z = w1*z2 + x1*y2 - y1*x2 + z1*w2
    return torch.stack([w, x, y, z], dim=1)

def get_sh_degree_from_coefficients(num_coeffs):
    """Determine SH degree from number of coefficients."""
    # Degree 0: 1 coeff
    # Degree 1: 4 coeffs
    # Degree 2: 9 coeffs
    # Degree d: (d+1)^2 coeffs
    # Solve (d+1)^2 = num_coeffs
    d = int(np.sqrt(num_coeffs)) - 1
    return d

def get_wigner_D(R, degree):
    """
    Compute Wigner D-matrix for rotation R and SH degree.
    This is a placeholder for e3nn or similar library call.
    In practice, use e3nn.o3.spherical_harmonics or e3nn.nn.wigner_D.
    """
    # Placeholder implementation
    # The actual computation involves Wigner D-matrices which are complex.
    # For real SH, we use real Wigner D-matrices.
    # This function should return a tensor of shape (num_coeffs, num_coeffs)
    raise NotImplementedError("Use e3nn library for Wigner D-matrix computation")
```

**Parameters:**
*   $\mathbf{T}$: Camera-to-World matrix. Typical values depend on the scene scale.
*   `e3nn`: Library for computing Wigner D-matrices [S1].

**Edge Cases and Pitfalls:**
*   **Isotropic vs Anisotropic:** Check scale channel count to determine if rotation composition is needed for the orientation [S2, unofficial].
*   **Quaternion Normalization:** Quaternions must be normalized before multiplication to avoid drift [S2, unofficial].
*   **SH Degree:** The Wigner D-matrix size depends on the SH degree. Ensure the correct degree is used [S1].
*   **Precision:** Use double precision for matrix operations if high accuracy is required, especially for large scenes [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for mean and covariance transformation, where $N$ is the number of Gaussians. SH transformation is $O(N \cdot C_{sh}^2)$ or $O(N \cdot C_{sh})$ if optimized.
*   **Memory Complexity:** $O(N)$ to store the transformed parameters.
*   **Performance:** The transformation is computationally lightweight compared to rendering. The bottleneck is typically the rendering step itself.
*   **Strengths:** Enables consistent rendering across multiple views by aligning Gaussians to a common world frame.
*   **Limitations:** Requires accurate camera extrinsics. Errors in $\mathbf{T}$ lead to misaligned Gaussians.

## Background
This method is used in 3D Gaussian Splatting variants that predict Gaussian parameters in camera space, such as PixelSplat [S1]. It allows for the integration of these camera-space predictions into a global world coordinate system for consistent multi-view rendering and scene reconstruction. The use of Wigner D-matrices for SH rotation is standard in 3D graphics for handling directional data under rotation [S1].

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
[S2] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
