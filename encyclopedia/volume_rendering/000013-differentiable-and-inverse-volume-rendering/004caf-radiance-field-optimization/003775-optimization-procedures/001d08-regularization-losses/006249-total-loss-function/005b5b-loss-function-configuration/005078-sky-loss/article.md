# Sky Loss

Also called: Sky Transmittance Mask Loss

## Overview
Sky Loss is a regularization term used in differentiable volume rendering to prevent the invalid allocation of sparse voxels in unbounded free space. It isolates the sky region and enforces maximum global ray transmittance, ensuring that background color generation is routed exclusively through a 2D environment map rather than learned 3D primitives [S2, unofficial]. It differs from standard photometric losses by targeting the transmittance field specifically in sky-masked pixels to suppress background noise [S2, unofficial].

## How it works
The method operates by identifying pixels corresponding to the sky and penalizing any residual density or occupancy that prevents rays from passing through to the background environment map.

### Inputs and Outputs
*   **Inputs**:
    *   `sky_label`: A binary mask indicating sky pixels (1.0 for sky, 0.0 otherwise) [S1, unofficial].
    *   `density_field_weights` or `occupancy_field_weights`: The accumulated weights along rays from the volume renderer [S1, unofficial].
    *   `T_final`: The residual transmittance accumulated at the terminal boundary of the composition pass [S2, unofficial].
    *   `M_sky`: A binary mask derived from zero-depth LiDAR boundaries [S2, unofficial].
*   **Outputs**:
    *   `sky_loss`: A scalar loss value [S1, unofficial].

### Data Structures
*   **Sky Label**: A tensor of shape matching the image batch, containing float values 0.0 or 1.0 [S1, unofficial].
*   **Mask**: A binary tensor of the same spatial dimensions as the rendered image [S2, unofficial].

### Steps
1.  **Compute Sky Label**: Convert the input `batch['sky']` to float and move to the computation device. Compute `sky_label = 1.0 - sky_label` [S1, unofficial].
2.  **Clip Weights**: Clip the sum of field weights along the ray dimension (dim=1) to the range $[10^{-3}, 1 - 10^{-3}]$ to prevent gradient vanishing or explosion at boundaries [S1, unofficial].
3.  **Compute Loss**:
    *   **For Density/Occupancy Fields (SDFStudio variant)**: Compute the Binary Cross Entropy (BCE) between the clipped sum of weights and `sky_label`. Multiply the result by a coefficient of 0.01 [S1, unofficial].
    *   **For Transmittance Fields (DrivingVoxels variant)**: Compute the Mean Squared Error (MSE) between the final transmittance $T_{\text{final}}$ and 1.0 for all pixels in the sky mask [S2, unofficial].

### Formulas
**SDFStudio Variant (BCE-based):**
$$ \mathcal{L}_{\text{sky}} = 0.01 \cdot \text{BCE}(\text{clip}(\sum w, 10^{-3}, 1-10^{-3}), \text{sky\_label}) $$
Where $w$ are the field weights [S1, unofficial].

**DrivingVoxels Variant (MSE-based):**
$$ \mathcal{L}_{\text{sky}} = \frac{1}{|\mathcal{M}_{\text{sky}}|} \sum_{i \in \mathcal{M}_{\text{sky}}} (T_{\text{final}, i} - 1)^2 $$
Where $\mathcal{M}_{\text{sky}}$ is the set of pixels in the sky mask and $T_{\text{final}}$ is the residual transmittance [S2, unofficial].

### Parameters
*   **Weight Coefficient**: 0.01 (for SDFStudio variant) [S1, unofficial].
*   **Clip Range**: $[10^{-3}, 1 - 10^{-3}]$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Background Primitives**: Without this loss, background primitives in the sky dome may be incorrectly optimized. This loss completely penalizes such primitives to ensure background color is generated exclusively through the 2D environment map [S2, unofficial].
*   **Mask Derivation**: The sky mask $\mathcal{M}_{\text{sky}}$ must be derived from zero-depth LiDAR boundaries to accurately isolate unbounded free space [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sky_loss_sdfstudio(batch, field_weights, use_nerfacto=True):
    """
    Computes Sky Loss for SDFStudio-style density/occupancy fields.
    """
    # Step 1: Compute sky label
    # batch['sky'] is assumed to be 1 for sky, 0 for non-sky
    # sky_label is 1.0 for sky, 0.0 for non-sky
    sky_label = 1.0 - batch['sky'].float().to(device)
    
    # Step 2: Sum weights along ray dimension (dim=1)
    # Assuming field_weights has shape [B, N_rays, N_samples] or similar
    # Sum over the sample dimension
    sum_weights = torch.sum(field_weights, dim=1)
    
    # Step 3: Clip weights
    clipped_weights = torch.clamp(sum_weights, min=1e-3, max=1-1e-3)
    
    # Step 4: Compute BCE loss
    # BCE expects probabilities, so we treat clipped_weights as predicted probability
    # sky_label is the target
    bce_loss = torch.nn.functional.binary_cross_entropy(clipped_weights, sky_label)
    
    # Step 5: Scale by coefficient
    sky_loss = bce_loss * 0.01
    
    return sky_loss

def compute_sky_loss_drivingvoxels(T_final, M_sky):
    """
    Computes Sky Loss for DrivingVoxels-style transmittance fields.
    """
    # M_sky is a binary mask (1.0 for sky pixels, 0.0 otherwise)
    # T_final is the residual transmittance tensor
    
    # Select pixels in the sky mask
    sky_pixels_T_final = T_final[M_sky.bool()]
    
    # Compute MSE against 1.0
    # If no sky pixels, return 0
    if sky_pixels_T_final.numel() == 0:
        return torch.tensor(0.0, device=T_final.device)
        
    mse_loss = torch.mean((sky_pixels_T_final - 1.0) ** 2)
    
    return mse_loss
```

## Subtopics

### Sky Transmittance Mask Loss
A specific implementation of Sky Loss used in DrivingVoxels. It uses a Mean Squared Error objective on the final transmittance $T_{\text{final}}$ rather than Binary Cross Entropy on weights. It relies on a binary mask $\mathcal{M}_{\text{sky}}$ derived from zero-depth LiDAR boundaries. It is chosen when using sparse voxel rasterization in unbounded driving scenes to prevent invalid voxel allocation in free space [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of pixels in the sky mask. The operation is element-wise and parallelizable on GPU.
*   **Memory Complexity**: $O(N)$ to store the sky mask and intermediate transmittance/weight tensors.
*   **Strengths**: Effectively suppresses background noise and prevents the network from learning incorrect 3D structures in the sky region.
*   **Limitations**: Requires accurate sky masks (from LiDAR or other sensors). If the mask is inaccurate, it may penalize valid foreground geometry or fail to suppress background noise.
*   **When to use**: In unbounded scenes where a 2D environment map is used for background rendering and 3D primitives should not contribute to the sky region.

## Background
The concept of isolating sky regions for regularization appears in recent differentiable volume rendering literature for autonomous driving and large-scale scene reconstruction. The SDFStudio variant [S1, unofficial] uses a BCE loss on density/occupancy weights. The DrivingVoxels variant [S2, unofficial] introduces the MSE-based transmittance loss to handle sparse voxel allocation in unbounded free space.

## Sources
[S1] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S2] DrivingVoxels: Compositional Sparse Voxel Rasterization for Dynamic Driving Scene Reconstruction, https://doi.org/10.48550/arxiv.2606.23031, fetched 2026-10-26 (unofficial)
