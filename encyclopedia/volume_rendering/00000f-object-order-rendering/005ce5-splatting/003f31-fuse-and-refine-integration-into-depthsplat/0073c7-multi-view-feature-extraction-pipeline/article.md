# Multi-View Feature Extraction Pipeline

Also called: Cross-View U-Net Fusion, Multi-View Branch

## Overview
The Multi-View Feature Extraction Pipeline aggregates geometry-aware features from multiple input views to produce a unified low-resolution feature tensor. It distinguishes itself by combining multi-view transformer features, monocular depth priors, and cost-volume matching evidence, outputting a fixed $H/4 \times W/4$ resolution tensor suitable for downstream compression or rendering [S1, unofficial].

## How it works
**Inputs:**
- $N$ input views with images, camera parameters (intrinsics and extrinsics), and near/far depth ranges.
- Monocular depth prior model (DepthAnything V2).

**Outputs:**
- Fused feature tensor $F_1 \in \mathbb{R}^{N \times C \times \frac{H}{4} \times \frac{W}{4}}$, where $N$ is the number of views, $C$ is the feature dimension, and $H, W$ are the input image dimensions [S1, unofficial].

**Data Structures:**
- **Multi-view features ($F_{mv}$):** Tensor of shape $N \times C \times \frac{H}{4} \times \frac{W}{4}$.
- **Monocular features ($F_{mono}$):** Tensor of shape $N \times C_{mono} \times \frac{H}{4} \times \frac{W}{4}$.
- **Cost volumes ($V_{cv}$):** Tensor of shape $N \times D \times \frac{H}{4} \times \frac{W}{4}$, where $D$ is the number of depth candidates.
- **Fused features ($F_1$):** Tensor of shape $N \times C \times \frac{H}{4} \times \frac{W}{4}$.

**Steps:**

1. **Per-View Feature Extraction:**
   Extract per-view features using a lightweight ResNet-style backbone [S1, unofficial].

2. **Multi-View Transformer Aggregation:**
   Process the per-view features with a multi-view Transformer composed of alternating self-attention and cross-view attention layers.
   - For $N > 2$ views, restrict cross-view interaction to nearby source views to maintain computational efficiency [S1, unofficial].
   - Resize the resulting features to a common spatial resolution of $\frac{H}{4} \times \frac{W}{4}$, yielding $F_{mv} \in \mathbb{R}^{N \times C \times \frac{H}{4} \times \frac{W}{4}}$ [S1, unofficial].

3. **Cost-Volume Construction:**
   Construct per-view cost volumes using plane-sweep stereo [S1, unofficial].
   - For each reference view, sample depth candidates within the near–far range.
   - Warp source view features to the reference camera at each candidate depth using known camera parameters.
   - Compute correlations between warped source features and reference features.
   - Aggregate correlations from selected neighboring views [S1, unofficial].
   - **Dataset-Specific Handling:**
     - For RealEstate10K and ACID: Construct the cost volume directly from the $\frac{1}{4}$-resolution multi-view features [S1, unofficial].
     - For DL3DV: Due to high computational overhead from large resolution and many views, bilinearly downsample the multi-view features to a lower resolution, compute the cost volume, then bilinearly upsample the result back to $\frac{H}{4} \times \frac{W}{4}$ [S1, unofficial].

4. **Monocular Geometric Prior Extraction:**
   Extract per-view geometric priors independently using a monocular depth branch initialized from DepthAnything V2 [S1, unofficial].
   - Resize these features to $\frac{H}{4} \times \frac{W}{4}$ [S1, unofficial].

5. **Cross-View U-Net Fusion:**
   Fuse monocular geometric features, multi-view matching features, and cost-volume information using a cross-view U-Net [S1, unofficial].
   - Retain the low-resolution U-Net representation at $\frac{1}{4}$ input resolution as the final output $F_1$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_view_feature_extraction_pipeline(images, cameras, near_far_ranges, dataset_name):
    """
    Extracts and fuses multi-view features.
    
    Args:
        images: List of N images, shape (H, W, 3).
        cameras: List of N camera parameters (intrinsics, extrinsics).
        near_far_ranges: List of N tuples (near, far).
        dataset_name: String, one of 'RealEstate10K', 'ACID', 'DL3DV'.
        
    Returns:
        F1: Fused feature tensor, shape (N, C, H/4, W/4).
    """
    N = len(images)
    
    # Step 1: Per-View Feature Extraction
    # ResNet-style backbone
    per_view_features = []
    for img in images:
        # Backbone returns features at original resolution
        feat = resnet_backbone(img) 
        per_view_features.append(feat)
        
    # Step 2: Multi-View Transformer Aggregation
    # Transformer with alternating self- and cross-view attention
    # Cross-view interaction restricted to nearby views if N > 2
    mv_transformer = MultiViewTransformer()
    F_mv_raw = mv_transformer(per_view_features, cameras)
    
    # Resize to H/4 x W/4
    F_mv = resize_bilinear(F_mv_raw, target_size=(images[0].shape[0] // 4, images[0].shape[1] // 4))
    
    # Step 3: Cost-Volume Construction
    cost_volumes = []
    for i in range(N):
        ref_feat = F_mv[i]
        ref_cam = cameras[i]
        near, far = near_far_ranges[i]
        
        # Sample depth candidates
        depth_candidates = sample_depth_candidates(near, far)
        
        # Warp source features and compute correlations
        # Select neighboring views based on dataset or proximity
        source_indices = get_neighboring_views(i, N)
        
        cv = plane_sweep_stereo(
            ref_feat=ref_feat,
            source_feats=[F_mv[j] for j in source_indices],
            source_cams=[cameras[j] for j in source_indices],
            ref_cam=ref_cam,
            depth_candidates=depth_candidates
        )
        
        # Dataset-specific resolution handling
        if dataset_name == 'DL3DV':
            # Downsample before computation, upsample after
            # Note: F_mv is already at 1/4 resolution. 
            # The source says "bilinearly downsample the multi-view features 
            # to a lower spatial resolution for cost-volume computation".
            # We assume a further downsampling factor 'k' for DL3DV.
            # Since 'k' is not specified, we note the operation.
            cv_low_res = resize_bilinear(cv, target_size=(cv.shape[2] // 2, cv.shape[3] // 2)) # Placeholder factor
            # Recompute or interpolate? Source says "compute cost volume" at lower res.
            # This implies warping and correlation at lower res.
            # For implementation clarity, we assume the warping/correlation 
            # is performed on the downsampled features.
            cv_low_res = plane_sweep_stereo(
                ref_feat=resize_bilinear(ref_feat, target_size=(ref_feat.shape[2] // 2, ref_feat.shape[3] // 2)),
                source_feats=[resize_bilinear(F_mv[j], target_size=(F_mv[j].shape[2] // 2, F_mv[j].shape[3] // 2)) for j in source_indices],
                source_cams=[cameras[j] for j in source_indices],
                ref_cam=ref_cam,
                depth_candidates=depth_candidates
            )
            cv = resize_bilinear(cv_low_res, target_size=(ref_feat.shape[2], ref_feat.shape[3]))
        else:
            # RealEstate10K and ACID: Direct construction from 1/4 resolution
            pass
            
        cost_volumes.append(cv)
        
    # Step 4: Monocular Geometric Prior Extraction
    monocular_branch = DepthAnythingV2()
    F_mono_raw = []
    for img in images:
        # Extract geometric priors independently
        mono_feat = monocular_branch(img)
        F_mono_raw.append(mono_feat)
        
    # Resize to H/4 x W/4
    F_mono = [resize_bilinear(f, target_size=(images[0].shape[0] // 4, images[0].shape[1] // 4)) for f in F_mono_raw]
    
    # Step 5: Cross-View U-Net Fusion
    # Fuse F_mv, F_mono, and cost_volumes
    cross_view_unet = CrossViewUNet()
    F1 = cross_view_unet(F_mv, F_mono, cost_volumes)
    
    return F1
```

**Parameters:**
- Spatial resolution scaling factor: $1/4$ of the input resolution [S1, unofficial].
- Feature dimension $C$: Not given in the sources.
- Number of depth candidates $D$: Not given in the sources.

**Edge Cases and Pitfalls:**
- **DL3DV Overhead:** Large spatial resolution and many input views cause high computational and memory overhead during cost-volume construction. Mitigated by temporary bilinear downsampling before computation and upsampling back to $1/4$ resolution [S1, unofficial].
- **Near/Far Range:** Incorrect near/far ranges will lead to incorrect depth sampling and warped features [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the multi-view transformer and cost-volume construction. Cost-volume construction is $O(N \cdot D \cdot H \cdot W)$ per view, where $D$ is the number of depth candidates.
- **Memory Complexity:** Dominated by the cost volumes, which require storing correlations for each depth candidate.
- **Strengths:** Combines explicit geometric matching (cost volume) with learned features (transformer) and monocular priors.
- **Limitations:** High memory usage for high-resolution inputs or many views (mitigated for DL3DV).
- **When to use:** When geometry-aware multi-view feature fusion is required for downstream tasks like compression or rendering.
- **When not to use:** When computational resources are extremely limited and monocular features alone are sufficient.

## Background
The method is part of the FeCoSplat framework, which builds on DepthSplat [S1, unofficial]. It was introduced in "FeCoSplat: Feedback-Guided Compression for Feed-Forward 3D Gaussian Splatting" [S1, unofficial].

## Sources
[S1] FeCoSplat: Feedback-Guided Compression for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2609.33330, fetched 2026-10-29 (unofficial)
