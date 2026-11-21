# FineSplat

Also called: Detailed Splats, Fine Patches, Local Groups

## Overview
FineSplat is a feed-forward 3D Gaussian Splatting framework that predicts a set of 3D Gaussians from $N$ context views using a two-stage dual-grouping strategy to balance local detail preservation with global multi-view reasoning [S1, unofficial]. It distinguishes itself from sibling methods like [YoNoSplat](16425-yonosplat.md) and [ZipSplat](16425-zipsplat.md) by employing high-resolution 4x4 patch tokens instead of coarser 14x14 patches, combined with serialized attention for 3D spatial grouping to avoid the prohibitive cost of global multi-view attention [S1, unofficial].

## How it works
FineSplat operates in two stages: view-wise token grouping (Stage 1) and 3D spatial token grouping (Stage 2) [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** $N$ context RGB images $I \in \mathbb{R}^{N \times H \times W \times 3}$ and their corresponding camera poses (intrinsics and extrinsics). The method supports varying numbers of input views sampled from $\{6, 12, 24, 32\}$ during training [S1, unofficial].
*   **Outputs:** A set of 3D Gaussian primitives $\mathcal{G} = \{\mu, \Sigma, \alpha, c\}$, where $\mu$ is the mean, $\Sigma$ is the covariance (defined by scale and rotation), $\alpha$ is opacity, and $c$ is color [S1, unofficial].

### Data Structures
*   **Tokens:** $M = N \cdot \frac{H}{p} \cdot \frac{W}{p}$ fine-grained tokens $z^{(0)} \in \mathbb{R}^{M \times d}$, where $p=4$ is the patch size [S1, unofficial].
*   **Gaussians:** $K=4$ Gaussians predicted per token, assigned to non-overlapping 2x2 sub-patches [S1, unofficial].

### Parameters
*   $p$: Patch size, set to 4 [S1, unofficial].
*   $T$: Number of iterative refinement steps in Stage 2, set to 4 [S1, unofficial].
*   $K$: Number of Gaussians per token, set to 4 [S1, unofficial].
*   $S$: Size of groups for serialized attention in 3D spatial grouping [S1, unofficial].
*   $s$: Sub-patch stride, set to 2 [S1, unofficial].
*   $N$: Number of input context views [S1, unofficial].

### Algorithm

#### Scene Normalization
To handle scale ambiguity in feed-forward reconstruction, scene scale is normalized using context camera poses [S1, unofficial]. Poses are expressed in the mean context camera frame, and translations are rescaled so that all context camera centers lie within $[-1, 1]^3$. The same transform is applied to target poses [S1, unofficial].

#### Stage 1: View-wise Token Grouping
1.  **Feature Extraction:** Extract multi-scale features $F^l$ for $l \in \{1, 2, 3\}$ (finest to coarsest) from the input images using a shared encoder (e.g., Vision Transformer or CNN) [S1, unofficial].
2.  **Token Fusion:** Fuse features into $M$ fine-grained tokens $z^{(0)}$ using Equation (1) [S1, unofficial]:
    $$ z^{(0)} = \text{fuse}\left(\text{up}\left(\text{fuse}\left(\text{up}(F^3) + F^2\right)\right)\right) + F^1 $$
    where $\text{up}(\cdot)$ is bilinear upsampling and $\text{fuse}(\cdot)$ is a convolutional fusion block [S1, unofficial].
3.  **Group-wise Attention:** Encode tokens using view-wise token grouping based on the Multi-View Pyramid (MVP) architecture. This performs fine-grained attention within small view groups and exchanges information across all views via coarser global tokens [S1, unofficial].

#### Stage 2: 3D Spatial Token Grouping
This stage iteratively refines the Gaussians for $T=4$ steps [S1, unofficial].

1.  **Gaussian Prediction:** Decode tokens $z^{(i)}$ into an initial set of Gaussians using a lightweight Gaussian head $\mathcal{D}$ (linear decoder). Each token predicts $K=4$ Gaussians [S1, unofficial].
2.  **Patch-Aligned Sub-Pixel Gaussian Reconstruction:**
    *   For each token, assign 4 Gaussians to four non-overlapping 2x2 sub-patches [S1, unofficial].
    *   Learn a 2D image-space offset $\Delta = (\Delta_x, \Delta_y)$ for each Gaussian [S1, unofficial].
    *   Bound the offset to keep Gaussians within their sub-patch:
        $$ \delta_x = 2s \tanh(\Delta_x), \quad \delta_y = 2s \tanh(\Delta_y) $$
        where $s=2$ is the sub-patch stride, ensuring $\delta_x, \delta_y \in [-s/2, s/2]$ [S1, unofficial].
    *   Compute displaced image coordinate $(u, v) = (u_c, v_c) + \delta$, where $(u_c, v_c)$ is the sub-patch center [S1, unofficial].
    *   **Ray Unprojection:** Unproject $(u, v)$ using predicted camera intrinsics. Transform and normalize the direction using camera-to-world rotation, using camera translation as ray origin to get $\mu$ [S1, unofficial].
3.  **Error Feature Computation:**
    *   Render the predicted Gaussians into every context view to get rendered images $\hat{I}_n$ [S1, unofficial].
    *   Compute signed pixel error map $E_n = \hat{I}_n - I_n$ [S1, unofficial].
    *   Compute feature-level difference $\Delta\phi_n = \phi(\hat{I}_n) - \phi(I_n)$ using a pretrained ResNet-18 encoder $\phi(\cdot)$ [S1, unofficial].
    *   Combine with finest-scale features $F_n^1$ from Stage 1 via linear embeddings $E_{err}$ and $E_{feat}$:
        $$ F_n^{\text{err}} = [E_{err}(E_n, \Delta\phi_n); E_{\text{feat}}(F_n^1)] \in \mathbb{R}^{\frac{H}{p} \times \frac{W}{p} \times d_e} $$
        where $[\cdot; \cdot]$ represents concatenation [S1, unofficial].
4.  **Error Aggregation:**
    *   For each token, gather two error signals:
        *   $e_{\text{self}}$: Self error feature from the patch where the token is predicted.
        *   $e_{\text{agg}}$: Aggregated error feature averaged over all context views where the token is visible (projected onto error feature maps and masked by visibility/occlusion) [S1, unofficial].
5.  **3D Spatial Grouping via Serialized Attention:**
    *   Lift tokens to 3D space using their predicted $\mu$ [S1, unofficial].
    *   Order tokens along a space-filling curve [S1, unofficial].
    *   Partition tokens into groups of $S$ consecutive points along the curve [S1, unofficial].
    *   Apply self-attention within each group to exchange $e_{\text{self}}$ and $e_{\text{agg}}$ with 3D neighbors [S1, unofficial].
    *   Use four attention blocks in Stage 2, each using a different space-filling curve, so each token meets a different set of neighbors [S1, unofficial].
    *   Update tokens:
        $$ \Delta z_j^{(i)} = \text{Linear}(\text{SerialAttn}(h_j^{(i)})) $$
        $$ z_j^{(i+1)} = z_j^{(i)} + \Delta z_j^{(i)} $$
        where $h_j^{(i)}$ is the feature of token $j$ at step $i$ [S1, unofficial].
6.  **Iteration:** Repeat steps 1-5 for $T=4$ steps [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def finesplat(context_images, context_poses):
    """
    context_images: List of N RGB images, shape (H, W, 3)
    context_poses: List of N camera poses (intrinsics, extrinsics)
    Returns: List of 3D Gaussian primitives
    """
    # Scene Normalization
    mean_pose = compute_mean_pose(context_poses)
    normalized_poses = normalize_poses(context_poses, mean_pose, scale_range=(-1, 1))
    
    # Stage 1: View-wise Token Grouping
    # Extract multi-scale features F^1, F^2, F^3
    features = extract_multi_scale_features(context_images)
    
    # Fuse into fine-grained tokens z^(0)
    # z^(0) = fuse(up(fuse(up(F^3) + F^2)) + F^1)
    z = fuse_tokens(features)
    
    # MVP-based view-wise attention
    z = multi_view_pyramid_attention(z, normalized_poses)
    
    # Stage 2: 3D Spatial Token Grouping (Iterative Refinement)
    for step in range(4):
        # Decode Gaussians from tokens
        gaussians = decode_gaussians(z, normalized_poses)
        
        # Render Gaussians to context views
        rendered_images = render_gaussians(gaussians, normalized_poses)
        
        # Compute Error Features
        error_features = []
        for n in range(len(context_images)):
            # Pixel error
            E_n = rendered_images[n] - context_images[n]
            # Feature difference using ResNet-18
            phi = pretrained_resnet_18
            delta_phi_n = phi(rendered_images[n]) - phi(context_images[n])
            # Finest scale features from Stage 1 for view n
            F_n_1 = get_finest_features(features[n])
            
            # Combine errors
            # F_n_err = [E_err(E_n, delta_phi_n); E_feat(F_n_1)]
            err_emb = linear_embedding(E_n, delta_phi_n)
            feat_emb = linear_embedding(F_n_1)
            F_n_err = concatenate(err_emb, feat_emb)
            error_features.append(F_n_err)
        
        # Aggregate errors for each token
        token_errors = aggregate_token_errors(z, gaussians, error_features, normalized_poses)
        
        # 3D Spatial Grouping
        # Lift tokens to 3D
        token_positions = get_token_positions(gaussians)
        
        # Serialized Attention with 4 different space-filling curves
        for curve in range(4):
            z = serialized_attention(z, token_positions, token_errors, curve)
            
    # Final Gaussian Prediction
    final_gaussians = decode_gaussians(z, normalized_poses)
    return final_gaussians

def patch_aligned_sub_pixel_reconstruction(token, sub_patch_center, intrinsics, pose):
    """
    Predicts Gaussian mean for a specific sub-patch within a token.
    """
    # Learn offset
    delta_x, delta_y = learn_offset(token)
    
    # Bound offset
    s = 2
    delta_x_bounded = 2 * s * tanh(delta_x)
    delta_y_bounded = 2 * s * tanh(delta_y)
    
    # Displaced coordinate
    u_c, v_c = sub_patch_center
    u = u_c + delta_x_bounded
    v = v_c + delta_y_bounded
    
    # Ray Unprojection
    direction = unproject(u, v, intrinsics)
    direction = normalize(direction)
    direction = rotate(direction, pose.rotation)
    origin = pose.translation
    
    return origin + direction * depth # Depth predicted by Gaussian head
```

### Edge Cases and Pitfalls
*   **Blind Spots:** Fixed view grouping in Stage 1 leaves blind spots where fine-grained tokens in different groups cannot interact. This is mitigated by Stage 2's 3D spatial grouping, which connects tokens across view groups once lifted to 3D [S1, unofficial].
*   **Scale Ambiguity:** Feed-forward reconstruction suffers from scale ambiguity. This is handled by normalizing scene scale using context camera poses, ensuring all context camera centers lie within $[-1, 1]^3$ [S1, unofficial].
*   **Iteration Count:** Quality improves gradually with more Stage 2 steps, but going beyond 4 steps yields no further gain [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** FineSplat is efficient due to its dual-grouping strategy. Stage 1 uses local view-wise attention, and Stage 2 uses serialized attention with space-filling curves, which scales favorably with token count compared to explicit k-nearest-neighbor search [S1, unofficial]. It is faster than [ZipSplat](16425-zipsplat.md) and [YoNoSplat](16425-yonosplat.md) for 6 and 12 input views, and moderately faster than ZipSplat for 24 views [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:**
    *   **DL3DV (6 views):** 27.325 PSNR, 0.871 SSIM, 0.115 LPIPS using 62K Gaussians [S1, unofficial].
    *   **RE10K (6 views):** 27.394 PSNR, 0.886 SSIM, 0.116 LPIPS using 75K Gaussians [S1, unofficial].
    *   **Mip-NeRF 360 (128 views):** 23.463 PSNR, 0.706 SSIM, 0.214 LPIPS using 1.6M Gaussians [S1, unofficial].
*   **Strengths:** High quality with small primitive budget; efficient reconstruction time; preserves fine-grained visual information; generalizes well to unseen scenes (zero-shot on RE10K and Mip-NeRF 360) [S1, unofficial].
*   **Limitations:** Performance may degrade if the number of input views is very large and tokens observing the same surface are separated into different view groups, though Stage 2 compensates for this [S1, unofficial].
*   **When to use:** Novel view synthesis where high quality and efficiency are required, especially with a small number of input views (6-24) [S1, unofficial].
*   **When not to use:** When depth supervision is available and preferred, or when the scene scale is known and fixed without normalization [S1, unofficial].

## Background
FineSplat was introduced in 2026 by Chen et al. [S1, unofficial]. It is trained on the DL3DV dataset and evaluated on DL3DV, RealEstate10K (RE10K), and Mip-NeRF 360 [S1, unofficial]. The method builds on the principles of feed-forward 3D Gaussian Splatting, aiming to preserve fine-grained visual information while avoiding the computational cost of global multi-view attention [S1, unofficial].

## Sources
[S1] FineSplat: Fine Patches, Local Groups, Detailed Splats, https://arxiv.org/abs/2610.25011, fetched 2026-10-29 (unofficial)
