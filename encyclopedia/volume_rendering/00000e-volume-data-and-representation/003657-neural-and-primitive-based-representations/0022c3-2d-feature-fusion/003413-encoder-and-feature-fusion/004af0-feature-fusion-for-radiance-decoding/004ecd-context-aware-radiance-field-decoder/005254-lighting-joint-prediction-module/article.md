# Lighting Joint Prediction Module
Also called: Lighting Decoding Network
## Overview
The Lighting Joint Prediction Module predicts a volumetric lighting representation by fusing global scene features and local visible surface features, processed by a 3D UNet [S1]. It differs from sibling methods that use sequences of 3D transpose convolutions for global feature decoding by employing an MLP decoder, which is more flexible and naturally extensible to multi-view input [S1].
## How it works
**Inputs:**
1.  Input image $I$.
2.  Global feature vector $\tilde{f}_L$ (encoded from the input image by a Direct Prediction Module).
3.  Predicted visible surface properties at pixel locations: depth $\tilde{D}$, normal $\tilde{N}$, and albedo $\tilde{A}$.
4.  Camera intrinsics.

**Outputs:**
*   Lighting Volume $\hat{L} \in \mathbb{R}^{8 \times X \times Y \times Z}$, representing Volumetric Spherical Gaussian (VSG) lighting containing HDR intensity [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D grid of resolution $X \times Y \times Z$. The standard resolution used is $128^3$ [S1].
*   **Global Feature Volume:** $z_g \in \mathbb{R}^{C \times X_g \times Y_g \times Z_g}$. The source does not specify the channel dimension $C$ or the resolution dimensions $X_g, Y_g, Z_g$ relative to the output volume $X, Y, Z$ [S1].
*   **Local Feature Volume:** $z_l \in \mathbb{R}^{9 \times X \times Y \times Z}$ [S1].

**Parameters:**
*   $\sigma_d$: Hyper-parameter for Gaussian distance decay, set to 0.15 meters [S1].

**Steps:**

1.  **Compute Global Feature Volume:**
    For each voxel center $(x, y, z)$, compute the global feature $z_g$ using an MLP decoder $h_{GFD}$ taking the voxel coordinates and the global feature $\tilde{f}_L$ [S1]:
    $$ z_g = h_{GFD} (x, y, z, \tilde{f}_L) $$
    This is repeated for all voxels in the global feature grid [S1].

2.  **Compute Local Features:**
    For each voxel in the output volume grid $(X \times Y \times Z)$:
    a.  Unproject the voxel center to find its projection $(u_p, v_p)$ onto the input image plane, assuming a depth $d_p$ corresponding to the voxel's depth along the ray [S1].
    b.  Retrieve the predicted depth $\tilde{D}_p$, normal $\tilde{N}_p$, albedo $\tilde{A}_p$, and input image RGB $I_p$ at pixel $(u_p, v_p)$ [S1].
    c.  Compute the Gaussian distance factor $k$:
        $$ k = e^{-\frac{(d - \tilde{D}_p)^2}{2\sigma_d^2}} $$
        where $d$ is the voxel's depth along the ray [S1].
    d.  Compute local features $z_l$:
        $$ z_l = [k I_p, k \tilde{N}_p, k \tilde{A}_p] $$
        This results in a 9-channel feature vector per voxel (3 for RGB, 3 for Normal, 3 for Albedo) [S1].

3.  **Fuse and Decode:**
    Concatenate global features $z_g$ and local features $z_l$ [S1]. Process the concatenated volume through a 3D CNN $h_{JP}$ (specifically a 3D UNet) to output the lighting volume $\hat{L}$ [S1]:
    $$ \hat{L} = h_{JP} (z_g, z_l; \Theta_{JP}) $$
    where $\Theta_{JP}$ are the parameters of the 3D CNN [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lighting_joint_prediction_module(
    global_feature_vector: Tensor,
    predicted_depth: Tensor,
    predicted_normal: Tensor,
    predicted_albedo: Tensor,
    input_image: Tensor,
    camera_intrinsics: Tensor,
    voxel_centers: Tensor,
    sigma_d: float = 0.15
) -> Tensor:
    """
    Predicts volumetric lighting by fusing global and local features.
    
    Args:
        global_feature_vector: Global feature vector f_L from Direct Prediction Module.
        predicted_depth: Predicted depth map D_tilde.
        predicted_normal: Predicted normal map N_tilde.
        predicted_albedo: Predicted albedo map A_tilde.
        input_image: Input RGB image I.
        camera_intrinsics: Camera intrinsic matrix.
        voxel_centers: Tensor of shape (N, 3) containing (x, y, z) for all voxels.
        sigma_d: Gaussian decay parameter (default 0.15m).
        
    Returns:
        Lighting volume L_hat of shape (8, X, Y, Z).
    """
    # 1. Compute Global Feature Volume
    # h_GFD is an MLP decoder. 
    # Note: The source does not specify the output resolution (X_g, Y_g, Z_g) 
    # or channel count C for the global feature volume.
    # We assume h_GFD maps (x,y,z, f_L) to a feature vector.
    # In practice, this is often implemented by iterating or using a grid sampler.
    
    # Placeholder for the specific implementation of h_GFD as the source 
    # refers to an Appendix for details not provided here [S1].
    global_feature_volume = compute_global_feature_volume(
        voxel_centers, global_feature_vector
    )
    
    # 2. Compute Local Features
    # Unproject voxel centers to image coordinates
    # This requires projecting 3D voxel centers to 2D pixel coordinates (u_p, v_p)
    # and determining the depth d along the ray.
    pixel_coords, ray_depths = unproject_voxels_to_image(
        voxel_centers, camera_intrinsics
    )
    
    # Retrieve properties at pixel coordinates
    # Interpolate from predicted maps and input image
    local_features_list = []
    
    for i in range(len(voxel_centers)):
        u, v = pixel_coords[i]
        d = ray_depths[i]
        
        # Check bounds
        if not is_valid_pixel(u, v, input_image.shape):
            local_features_list.append(torch.zeros(9))
            continue
            
        # Retrieve values at (u, v)
        I_p = input_image[:, u, v] # RGB
        D_p = predicted_depth[u, v]
        N_p = predicted_normal[:, u, v] # Normal is 3D
        A_p = predicted_albedo[:, u, v] # Albedo is 3D (RGB)
        
        # Compute Gaussian distance factor
        k = torch.exp(-((d - D_p)**2) / (2 * sigma_d**2))
        
        # Concatenate features: [k*I_p, k*N_p, k*A_p]
        # I_p is 3, N_p is 3, A_p is 3 -> Total 9
        feature_vec = torch.cat([
            k * I_p,
            k * N_p,
            k * A_p
        ])
        local_features_list.append(feature_vec)
        
    local_feature_volume = torch.stack(local_features_list) # Shape (N, 9)
    
    # Reshape to 3D grid (X, Y, Z, 9) then permute to (9, X, Y, Z)
    local_feature_volume = reshape_to_grid(local_feature_volume, voxel_centers)
    
    # 3. Fuse and Decode
    # Concatenate global and local features along channel dimension
    # Note: Source does not specify if global feature volume is upsampled/aligned 
    # to the output resolution (X, Y, Z). Assuming alignment or that h_JP handles it.
    fused_volume = torch.cat([global_feature_volume, local_feature_volume], dim=0)
    
    # Process through 3D CNN (3D UNet)
    lighting_volume = h_JP(fused_volume)
    
    return lighting_volume
```

**Edge Cases and Pitfalls:**
*   **Voxels outside FOV:** Voxels that do not project onto the input image must be handled, likely by zeroing out local features or skipping them [S1].
*   **Depth Mismatch:** The Gaussian factor $k$ decays rapidly with distance. Voxels far from the predicted surface manifold will have near-zero local features, relying entirely on global features [S1].
*   **Precision:** The source does not specify numerical precision requirements [S1].

## Operating characteristics
*   **Memory:** The model consumes 7.5GB of memory with a batch size of 1 [S1].
*   **Resolution:** The standard resolution for the initialized visible surface volume and predicted VSG lighting is $128^3$ [S1].
*   **Comparison:** Compared to methods using sequences of 3D transpose convolutions for global feature decoding, this MLP-based approach is more flexible and naturally extensible to multi-view input [S1].
*   **Performance:** On the InteriorNet dataset, the method achieves a Normal Angular Error of $22.95^\circ$ (with full training losses) compared to $23.94^\circ$ for NIR [S1].

## Background
The Lighting Joint Prediction Module is part of a pipeline for learning indoor inverse rendering from a single image, introduced in "Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting" [S1]. The pipeline includes a Direct Prediction Module, the Joint Prediction Module (this module), a Differentiable Re-rendering Module, and a Joint Re-prediction Module [S1].

## Sources
[S1] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
