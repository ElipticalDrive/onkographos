# MAC-Splat

Also called: Multi-Attribute Consistency Splatting

## Overview
MAC-Splat is a training framework for sparse-view 3D Gaussian Splatting that resolves depth and correspondence ambiguities by enforcing direct 3D consistency supervision. Unlike methods relying on 2D photometric losses or proxy geometry, it uses semantically informed 2D correspondences to anchor and jointly regularize the position, shape, and appearance of matched Gaussians in a shared world coordinate frame [S1].

## How it works
**Inputs:** Sparse-view image set; initial 3D Gaussian primitives (centroids, covariances, opacities, colors); ground-truth camera poses (training) or estimated poses (testing).
**Outputs:** Optimized 3D Gaussian scene representation.

**Data Structures:**
*   **Gaussian Primitives:** Standard 3DGS attributes: centroid $\mathbf{c}$, covariance matrix $\mathbf{\Sigma}$ (or rotation $\mathbf{R}$ and scale $\mathbf{s}$), opacity $\alpha$, and spherical harmonic coefficients for color $\mathbf{c}_{sh}$.
*   **Correspondence Anchors:** Pairs of 2D pixel coordinates $(u_1, v_1)$ and $(u_2, v_2)$ from different views, associated with a confidence score and semantic feature vector.

**Steps:**

1.  **Geometric Correspondence Extraction:**
    Use the MASt3R geometric backbone to extract sparse 2D correspondences between image pairs [S1].

2.  **Semantic Enrichment:**
    Lift the geometric matches with part-level features from a frozen DINOv3 encoder. This is performed via a lightweight Residual Semantic Fusion module to improve correspondence reliability and confidence estimation [S1].

3.  **3D Lifting:**
    Lift the 2D anchor pairs into 3D space. During training, use ground-truth camera poses. At test time, use MASt3R-estimated poses [S1].

4.  **Multi-Attribute Consistency (MAC) Loss Calculation:**
    Identify the Gaussians in the scene that project to the corresponding 2D anchors. Enforce direct, confidence-weighted consistency on these matched Gaussians in a common world coordinate frame. The loss jointly regularizes:
    *   **Positional Consistency:** Distance between centroids of matched Gaussians.
    *   **Shape Consistency:** Agreement between covariance parameters (respecting covariance geometry).
    *   **Appearance Consistency:** Agreement between opacity and color attributes.

    The formulation is robust to outliers [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mac_loss(gaussians, view1, view2, poses, confidences):
    """
    Computes the Multi-Attribute Consistency loss for matched Gaussians.
    
    Args:
        gaussians: List of Gaussian primitives with attributes (centroid, cov, alpha, color).
        view1, view2: Image views (indices or identifiers).
        poses: Camera poses (ground-truth for training, estimated for testing).
        confidences: Confidence scores for the 2D correspondences.
        
    Returns:
        loss: Scalar MAC loss value.
    """
    # 1. Extract 2D correspondences using MASt3R
    # Note: In implementation, this is a pre-processing step or external call.
    # We assume 'matches' contains pairs of (pixel_idx_1, pixel_idx_2) and their confidence.
    matches = extract_correspondences_ma3tr(view1, view2)
    
    # 2. Enrich with DINOv3 features (Semantic Fusion)
    # This step refines confidence and reliability.
    enriched_matches = semantic_fusion(matches, view1, view2)
    
    # 3. Initialize Loss Accumulators
    pos_loss = 0.0
    shape_loss = 0.0
    app_loss = 0.0
    total_weight = 0.0
    
    # 4. Iterate over matches to compute 3D consistency
    for match in enriched_matches:
        u1, v1 = match.pixel_1
        u2, v2 = match.pixel_2
        conf = match.confidence
        
        # Project 2D points to 3D rays using poses
        ray1 = ray_from_pixel(u1, v1, poses[view1])
        ray2 = ray_from_pixel(u2, v2, poses[view2])
        
        # Find Gaussians intersecting these rays or closest to the projection centers
        # In a rasterization-based splatting context, this often involves
        # querying the Gaussians that contributed to the pixel or are spatially closest.
        g1 = find_matching_gaussian(gaussians, ray1, view1)
        g2 = find_matching_gaussian(gaussians, ray2, view2)
        
        if g1 is None or g2 is None:
            continue
            
        # Positional Consistency: Distance between centroids
        # Ensure centroids are in the same world frame
        delta_pos = g1.centroid - g2.centroid
        pos_term = dot(delta_pos, delta_pos)
        
        # Shape Consistency: Covariance alignment
        # Use a metric that respects covariance geometry (e.g., Frobenius norm of difference
        # or log-det based divergence)
        delta_cov = g1.covariance - g2.covariance
        shape_term = frobenius_norm(delta_cov)
        
        # Appearance Consistency: Opacity and Color
        delta_alpha = g1.opacity - g2.opacity
        delta_color = g1.color - g2.color
        app_term = dot(delta_alpha, delta_alpha) + frobenius_norm(delta_color)
        
        # Weight by confidence
        weight = conf
        pos_loss += weight * pos_term
        shape_loss += weight * shape_term
        app_loss += weight * app_term
        total_weight += weight
        
    # Normalize by total confidence weight
    if total_weight > 0:
        pos_loss /= total_weight
        shape_loss /= total_weight
        app_loss /= total_weight
        
    # Combine into final MAC loss
    # Note: Weights for each attribute term are hyperparameters not specified in sources.
    # [G] Assuming equal weighting or standard L2 combination for implementation.
    loss = pos_loss + shape_loss + app_loss
    
    return loss
```

**Parameters:**
*   **Encoder:** DINOv3, kept frozen during training [S1].
*   **Backbone:** MASt3R, used for geometric matching [S1].
*   **Camera Poses:** Ground-truth during training; MASt3R-estimated at test time [S1].

**Edge Cases and Pitfalls:**
*   **Outliers:** The MAC loss formulation is designed to be robust to outliers in correspondences [S1].
*   **Pose Accuracy:** Performance depends on the quality of camera poses. The method uses ground-truth poses during training but must rely on MASt3R-estimated poses at test time, which may introduce errors if the geometric backbone fails [S1].
*   **Sparse View Ambiguity:** The method specifically targets sparse-view settings where 2D photometric losses are insufficient [S1].

## Operating characteristics
*   **Performance:** On ScanNet++, MAC-Splat improves average PSNR over Splatt3R by more than 4.5 dB [S1]. It also reduces LPIPS and maintains performance as the camera pose gap increases [S1].
*   **Strengths:** Resolves geometric ambiguities in sparse-view reconstruction; produces geometrically coherent and semantically plausible reconstructions; robust to outliers [S1].
*   **Limitations:** Relies on external geometric backbones (MASt3R) and semantic encoders (DINOv3), increasing computational overhead during the correspondence extraction phase [S1].
*   **When to use:** Sparse-view 3D reconstruction where standard 3DGS fails due to depth ambiguity.
*   **When not to use:** Dense-view settings where photometric losses are sufficient; real-time applications where the overhead of MASt3R/DINOv3 inference is prohibitive.

## Background
*   **Origin:** MAC-Splat was introduced to address geometric artifacts in sparse-view 3D Gaussian Splatting caused by the inability of 2D photometric losses to resolve depth and correspondence ambiguities [S1].
*   **Development:** It builds on the MASt3R geometric backbone and a frozen DINOv3 encoder to obtain semantically informed 2D correspondences [S1]. The core contribution is the Multi-Attribute Consistency (MAC) loss, which enforces holistic 3D consistency [S1].
*   **Usage:** Applied to sparse-view reconstruction tasks, with experiments conducted on ScanNet++ [S1].

## Sources
[S1] MAC-Splat: Multi-Attribute Consistency for High-Fidelity Sparse-View Reconstruction, https://doi.org/10.1007/978-3-032-37023-5_36, fetched 2026-10-26
