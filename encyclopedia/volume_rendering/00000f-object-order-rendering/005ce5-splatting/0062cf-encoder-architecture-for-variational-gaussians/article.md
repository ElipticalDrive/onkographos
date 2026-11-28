# Encoder Architecture for Variational Gaussians

Also called: DINO-Epipolar Gaussian Encoder, Variational Gaussian Encoder

## Overview
This method encodes two reference views into a variational 3D Gaussian representation, predicting not only geometric parameters but also distributions over spherical harmonic coefficients to model view-dependent appearance uncertainty [S1, unofficial]. It distinguishes itself from deterministic encoders by explicitly modeling uncertainty in both 3D location (via depth distributions) and local appearance (via variational parameters), enabling the generation of plausible details in occluded or empty areas through a subsequent GAN-based decoder [S1, unofficial].

## How it works
**Inputs:** Two reference views $I_1$ and $I_2$ sampled from a video sequence.
**Outputs:** A set of variational 3D Gaussians, each defined by mean position $\mathbf{x}$, opacity $o$, scale $S$, rotation $R$ (quaternion), color $c$, and variational parameters for spherical harmonic coefficients: mean $\mathbf{h}_\mu$ and variance $\mathbf{h}_\sigma$.

**Data Structures:**
*   **Feature Maps:** Pixel-aligned tensors output by the Vision Transformer.
*   **Depth Annotations:** Depth values associated with epipolar lines from the opposing view.
*   **Gaussian Properties:** For each sampled Gaussian: $\mathbf{x}, o, S, R, c, \mathbf{h}_\mu, \mathbf{h}_\sigma$.

**Steps:**

1.  **Vision Transformer Encoding:**
    Apply a pre-trained DINO ViT-B/8 [5] to both tokenized images $I_1$ and $I_2$ to obtain pixel-aligned feature maps. The ResNet backbone used in prior work (e.g., pixelSplat) is omitted [S1, unofficial].

2.  **Epipolar Annotation:**
    For each feature map, annotate outputs with depth values derived from the epipolar lines of the other view [S1, unofficial]. These depth values are positionally encoded and concatenated to the individual feature maps [S1, unofficial].

3.  **Epipolar Cross-Attention:**
    Compute keys, queries, and values for all locations in the annotated feature maps. Perform cross-attention between each pixel and samples from its corresponding epipolar line in the other view [S1, unofficial]. This communication resolves scale ambiguity between the two views [S1, unofficial].

4.  **Gaussian Sampling Head (Depth Distribution):**
    For each final feature map, predict a discrete distribution over bins along each ray, indicating the probability of a 3D Gaussian lying at a specific depth. For each bin, predict an offset to refine the 3D position [S1, unofficial].

5.  **Gaussian Property Prediction:**
    Sample multiple Gaussians per ray based on the predicted depth distribution. The probability of a sampled Gaussian is used as its opacity $o$. For each sample, predict:
    *   Scale $S$
    *   Rotation $R$ (as a quaternion)
    *   Color $c$
    *   Variational parameters: mean $\mathbf{h}_\mu$ and variance $\mathbf{h}_\sigma$ of spherical harmonic coefficients [S1, unofficial].

**Uncertainty Modeling:**
Uncertainty is modeled in two ways:
1.  **Location Uncertainty:** Gaussian locations are sampled from the predicted distributions over rays [S1, unofficial].
2.  **Appearance Uncertainty:** Local appearance uncertainty is modeled via the prediction of distribution parameters ($\mathbf{h}_\mu, \mathbf{h}_\sigma$) for the variational Gaussians [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_variational_gaussians(I1, I2):
    """
    Encodes two reference views into variational 3D Gaussians.
    I1, I2: Input images (H, W, 3)
    Returns: List of Gaussian parameters
    """
    # 1. Vision Transformer
    # Use pre-trained DINO ViT-B/8. Omit ResNet.
    F1 = DINO_ViT_B8(I1)
    F2 = DINO_ViT_B8(I2)
    
    # 2. Epipolar Annotation
    # Get depth values from epipolar lines of the other view
    # Positionally encode and concatenate
    depth_1_from_2 = get_epipolar_depth(I1, I2)
    depth_2_from_1 = get_epipolar_depth(I2, I1)
    
    F1_annotated = concatenate_positional_encoding(F1, depth_1_from_2)
    F2_annotated = concatenate_positional_encoding(F2, depth_2_from_1)
    
    # 3. Epipolar Transformer (Cross-Attention)
    # Compute K, Q, V for all locations
    K1, Q1, V1 = compute_kvq(F1_annotated)
    K2, Q2, V2 = compute_kvq(F2_annotated)
    
    # Cross-attention between each pixel and samples from epipolar line in other view
    F1_final = cross_attention(Q1, K2, V2, epipolar_samples=I2)
    F2_final = cross_attention(Q2, K1, V1, epipolar_samples=I1)
    
    # 4. Gaussian Sampling Head
    gaussians = []
    
    for F in [F1_final, F2_final]:
        # Predict discrete distribution over bins for each ray
        depth_probs, depth_offsets = predict_depth_distribution(F)
        
        # Sample multiple Gaussians per ray
        # Sample from the discrete distribution defined by depth_probs
        sampled_depth_indices = sample_from_distribution(depth_probs)
        
        for i in range(num_samples_per_ray):
            # Get bin index and offset for accurate 3D position
            bin_idx = sampled_depth_indices[i]
            offset = depth_offsets[bin_idx]
            
            # Calculate 3D position x
            x = calculate_3d_position_from_depth(bin_idx, offset, ray_origin, ray_direction)
            
            # Predict opacity o from probability of sampled Gaussian
            o = depth_probs[bin_idx]
            
            # Predict remaining properties
            S = predict_scale(F, i)
            R = predict_rotation_quaternion(F, i)
            c = predict_color(F, i)
            
            # Predict variational parameters for spherical harmonics
            h_mu = predict_mean_sh_coefficients(F, i)
            h_sigma = predict_var_sh_coefficients(F, i)
            
            gaussians.append({
                'x': x,
                'o': o,
                'S': S,
                'R': R,
                'c': c,
                'h_mu': h_mu,
                'h_sigma': h_sigma
            })
            
    return gaussians
```

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Explicitly models uncertainty in both location and appearance, allowing the decoder to generate plausible details in empty areas [S1, unofficial]. Resolves scale ambiguity through epipolar cross-attention [S1, unofficial].
*   **Limitations:** Requires two reference views. Relies on a pre-trained DINO ViT and a separate GAN-based decoder for final rendering [S1, unofficial].

## Background
Originated in the "latentSplat" paper (2024) [S1, unofficial]. It adapts the encoder from pixelSplat [10] to the setting of variational Gaussians, adding the capability to predict means and variances of spherical harmonic coefficients. It builds on the epipolar transformer [20] and vision transformer [5] architectures.

## Sources
[S1] latentSplat, https://arxiv.org/abs/2403.16292, fetched 2026-10-27 (unofficial)
