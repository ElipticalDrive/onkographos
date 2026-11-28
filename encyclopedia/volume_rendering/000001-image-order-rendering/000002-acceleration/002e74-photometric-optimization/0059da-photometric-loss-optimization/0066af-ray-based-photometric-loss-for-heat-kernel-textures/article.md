# Ray-based Photometric Loss for Heat Kernel Textures

Also called: HKTex Ray Training Loss, Multi-view Image Fitting Loss

## Overview
This method defines the differentiable rendering objective for training Heat Kernel Textures (HKTex) in a multi-view setting. It minimizes the difference between predicted and target ray colors to reconstruct diffuse albedo. It is distinguished from sibling methods by its exclusive use of synthetic rendered images as ground truth to prevent the model from baking scene lighting, shadows, and view-dependent reflections into the diffuse albedo, which is the only material property currently parametrized [S1].

## How it works
**Inputs:**
*   A mini-batch of intersecting rays $R = \{r_i\}_{i=1}^B$.
*   For each ray $r_i$, a set of $N_{\text{spp}}$ Monte Carlo radiance samples $\{\mathbf{L}_{i,s}\}_{s=1}^{N_{\text{spp}}}$ produced by the HKTex renderer.
*   Target colors $\mathbf{c}'_i$ corresponding to each ray.

**Outputs:**
*   Scalar loss value $\mathcal{L}_{\text{ray}}$.

**Data Structures:**
*   Ray batch size $B$: Number of rays in the mini-batch.
*   Samples per ray $N_{\text{spp}}$: Number of Monte Carlo samples per ray.

**Parameters:**
*   $B$: Typical value is 1024 [S1].
*   $N_{\text{spp}}$: Number of Monte Carlo radiance samples. Value not given in the sources.

**Steps:**
1.  **Average Radiance:** For each ray $r_i$, compute the predicted linear RGB color $\hat{\mathbf{c}}_i$ by averaging the Monte Carlo radiance samples:
    $$
    \hat{\mathbf{c}}_i = \frac{1}{N_{\text{spp}}} \sum_{s=1}^{N_{\text{spp}}} \mathbf{L}_{i,s}
    $$
2.  **Preprocessing:** Apply image-space preprocessing to both the predicted color $\hat{\mathbf{c}}_i$ and the target color $\mathbf{c}'_i$. This involves two sub-steps:
    *   **Clipping:** Clip each channel of the color vector to the range $[0, 1]$.
    *   **Gamma Conversion:** Apply a linear-to-gamma color transformation $\Gamma(\cdot)$.
    
    The preprocessed colors are:
    $$
    \tilde{\mathbf{c}}_i = \Gamma \big (\operatorname {clip}(\hat{\mathbf{c}}_i,0,1)\big ), \qquad \tilde{\mathbf{c}}'_i = \Gamma \big (\operatorname {clip}(\mathbf{c}'_i,0,1)\big )
    $$
    where $\operatorname {clip}(\cdot, 0, 1)$ is applied channel-wise [S1].
3.  **Loss Computation:** Compute the ray-training loss as the L1 norm of the difference between the preprocessed predicted and target colors, averaged over the batch $B$:
    $$
    \mathcal{L}_{\text{ray}} = \frac{1}{B} \sum_{i=1}^{B} \left \| \tilde{\mathbf{c}}_i - \tilde{\mathbf{c}}'_i \right \|_1
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hktex_ray_loss(predicted_samples, target_colors, gamma_func, batch_size, use_hdr=False):
    """
    Computes the ray-based photometric loss for HKTex.
    
    Args:
        predicted_samples: Tensor of shape (batch_size, Nspp, 3) containing 
                           Monte Carlo radiance samples for each ray.
        target_colors: Tensor of shape (batch_size, 3) containing target RGB colors.
        gamma_func: Function applying linear-to-gamma transformation.
        batch_size: Integer B, number of rays in the batch.
        use_hdr: Boolean. If True, skip clipping to [0, 1].
        
    Returns:
        Scalar loss value.
    """
    # Step 1: Average radiance samples to get predicted linear RGB color
    # Shape: (batch_size, 3)
    predicted_linear = predicted_samples.mean(dim=1)
    
    # Step 2: Preprocessing
    if not use_hdr:
        # Clip channel-wise to [0, 1]
        predicted_clipped = torch.clamp(predicted_linear, min=0.0, max=1.0)
        target_clipped = torch.clamp(target_colors, min=0.0, max=1.0)
    else:
        # Omit clipping for HDR targets to preserve dynamic range
        predicted_clipped = predicted_linear
        target_clipped = target_colors
        
    # Apply linear-to-gamma transformation
    predicted_gamma = gamma_func(predicted_clipped)
    target_gamma = gamma_func(target_clipped)
    
    # Step 3: Compute L1 loss averaged over the batch
    # Shape: (batch_size, 3) -> scalar
    loss = torch.mean(torch.abs(predicted_gamma - target_gamma))
    
    return loss
```

**Edge Cases and Pitfalls:**
*   **HDR Targets:** If the target images use High Dynamic Range (HDR), the clipping step to $[0, 1]$ must be omitted to preserve dynamic range information [S1].
*   **Lighting Baking:** Using real photographs as ground truth directly is discouraged because the current HKTex implementation parametrizes only the albedo component of the Principled BSDF. Optimizing against real photos would cause the model to incorrectly bake scene lighting, shadows, and view-dependent reflections into the diffuse color [S1].

## Operating characteristics
*   **Time Complexity:** The loss computation is $O(B \cdot N_{\text{spp}})$ for averaging samples and $O(B)$ for the L1 norm. The dominant cost is the rendering of $N_{\text{spp}}$ samples per ray.
*   **Memory Complexity:** Requires storing $B \cdot N_{\text{spp}} \cdot 3$ samples for the forward pass if not streamed, plus gradients. The text notes that heat-kernel texture queries are evaluated in chunks (typically 1024 surface points) and image tiling (e.g., $32 \times 32$ crops) is used to limit GPU memory [S1].
*   **Strengths:** Prevents baking of non-albedo effects (lighting, shadows, reflections) into the material texture by using synthetic ground truth [S1].
*   **Limitations:** Requires a differentiable renderer capable of producing Monte Carlo samples. The current formulation only optimizes albedo; extending to roughness, metallic, or normal maps is feasible but not implemented in this specific loss configuration [S1].
*   **When to use:** Multi-view image fitting experiments for HKTex where physically accurate diffuse albedo reconstruction is required [S1].
*   **When not to use:** When real photographs are the only available data and lighting/disentangling effects are not handled by extending the material parameterization [S1].

## Background
This method is used in the multi-view image fitting experiments for Heat Kernel Textures (HKTex) [S1]. The approach was introduced in the paper "Heat Kernel Textures the Geodesic Gaussians That Do Not Splat" [S1]. The authors note that while the ground truth could naturally be replaced with screen-space image pixels from real photographs, they isolate albedo fitting using synthetic ground truth to ensure physically accurate material reconstruction, as their current implementation parametrizes only the albedo component of the Principled BSDF [S1].

## Sources
[S1] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
