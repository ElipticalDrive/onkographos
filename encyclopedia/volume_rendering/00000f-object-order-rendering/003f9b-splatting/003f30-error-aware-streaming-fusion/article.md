# Error-aware Streaming Fusion

Also called: Adaptive Fusion Weighting, Error-based Splat Fusion

## Overview
Error-aware Streaming Fusion is a temporal stabilization technique for dynamic 3D Gaussian Splatting that mitigates artifacts caused by warping inaccuracies and appearance variations [S1]. It operates by adaptively weighting the fusion of historical splats against current frame predictions based on per-pixel rendering error [S1]. Unlike standard streaming methods that may accumulate warping errors or flicker, this method explicitly suppresses unreliable historical contributions and erroneous current-frame predictions to ensure temporal consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of current-frame splats $\mathcal{S}_t$ predicted by a multi-view transformer, a set of historical splats $\mathcal{S}_{t-1}$ from past keyframes, and the camera parameters for input views [S1].
*   **Outputs:** A fused set of splats $\mathcal{S}_{fused}$ with updated weights and attributes for rendering [S1].

**Data Structures:**
*   **Splat:** Defined by position $\mathbf{p}$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, and color $c$ [S1, implied by Gaussian Splatting context].
*   **Keyframe Buffer:** Maintains primitives from a set of past keyframes. In practice, two keyframes are maintained, placed 5 frames apart, each lasting 10 frames [S1].

**Steps:**
1.  **Render Current Frame:** Render the current frame splats $\mathcal{S}_t$ to the input views to obtain a reference depth map $D_{ref}$ and color image $I_{ref}$ [S1].
2.  **Project Historical Splats:** For each historical splat $s_h \in \mathcal{S}_{t-1}$, project it onto the input views [S1].
3.  **Error Assessment:** For each historical splat $s_h$:
    *   Render $s_h$ individually or in a batch to the input views to obtain a rendered image $I_{hist}$ and depth $D_{hist}$ [S1].
    *   Calculate the per-pixel error between $I_{hist}$ and $I_{ref}$ (and potentially depth consistency) [S1].
    *   Determine if $s_h$ appears in front of the rendered depth of the current frame splats ($D_{hist} < D_{ref}$) [S1].
4.  **Weight Adjustment (Historical):**
    *   Count the number of input views where $s_h$ results in increased rendering error compared to the baseline [S1].
    *   If the error increases in more than half of the input views, set the fusion weight of $s_h$ to zero ($w_h = 0$) [S1].
    *   Otherwise, retain its contribution [S1].
5.  **Weight Adjustment (Current):**
    *   Evaluate current frame splats $s_c \in \mathcal{S}_t$ predicted by the transformer [S1].
    *   If a current splat $s_c$ increases rendering error (compared to a baseline or expected consistency), assign it zero weight ($w_c = 0$) [S1].
6.  **Static Area Identification:**
    *   During the Splat-to-voxel transfer process, identify static areas [S1].
    *   Historical splats undergoing deformation mark their neighboring voxels as regions of motion [S1].
    *   Voxels not marked as motion are considered static [S1].
7.  **Temporal Blending:**
    *   In static areas, blend the current frame splats with historical splats [S1].
    *   Retain 50% of the voxel attributes from the past keyframe to enhance temporal continuity [S1].
    *   The fusion is smooth as older keyframes are replaced by newer ones [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_aware_streaming_fusion(current_splats, historical_splats, camera_params, views):
    """
    Fuses current and historical splats based on rendering error.
    
    Args:
        current_splats: List of splats for the current frame t.
        historical_splats: List of splats from past keyframes (t-1).
        camera_params: Camera intrinsics and extrinsics for input views.
        views: List of input view indices or IDs.
        
    Returns:
        fused_splats: List of splats with adjusted weights.
    """
    fused_splats = []
    
    # Step 1: Render current frame to get reference
    # Note: Implementation detail for rendering is assumed from parent [Splatting]
    ref_color, ref_depth = render_splats(current_splats, camera_params, views)
    
    # Step 2 & 3: Assess historical splats
    for s_h in historical_splats:
        # Project historical splat to views
        hist_color, hist_depth = render_splats([s_h], camera_params, views)
        
        # Calculate error and check depth
        error_count = 0
        for view in views:
            # Per-pixel error calculation (e.g., L1 or L2 norm)
            pixel_error = calculate_error(hist_color[view], ref_color[view])
            
            # Check if historical splat is in front of current depth
            # Note: Depth comparison logic depends on specific depth buffer implementation
            is_in_front = check_depth_consistency(hist_depth[view], ref_depth[view])
            
            # If error is high AND it is in front (occluding current), it's unreliable
            # The source says "increased error in more than half of the input views"
            # and "appears in front of the rendered depth"
            if pixel_error > threshold and is_in_front:
                error_count += 1
        
        # Step 4: Weight Adjustment
        if error_count > len(views) / 2:
            s_h.weight = 0.0
        else:
            s_h.weight = 1.0 # Or retain original weight, source implies binary suppression
            
        fused_splats.append(s_h)

    # Step 5: Weight Adjustment for Current Splats
    for s_c in current_splats:
        # Evaluate if current splat increases error
        # This requires a baseline or consistency check against the scene
        # Source: "current frame splats predicted... that increase rendering error are assigned zero weight"
        current_error = evaluate_current_splat_error(s_c, ref_color, ref_depth)
        
        if current_error > threshold:
            s_c.weight = 0.0
        else:
            s_c.weight = 1.0
            
        fused_splats.append(s_c)

    # Step 6 & 7: Static Area Blending
    # Identify static areas from Splat-to-voxel transfer
    static_voxels = identify_static_voxels(historical_splats, current_splats)
    
    for s in fused_splats:
        if s.position in static_voxels:
            # Retain 50% of voxel attributes from past keyframe
            # Note: This implies a blending of attributes, not just weight
            # Implementation: s.attributes = 0.5 * s.attributes + 0.5 * past_attributes
            blend_static_attributes(s)
            
    return fused_splats

def calculate_error(img1, img2):
    # Placeholder for specific error metric (e.g., MSE, SSIM)
    # Not given in the sources. [G]
    return mean_squared_error(img1, img2)

def check_depth_consistency(depth1, depth2):
    # Placeholder for depth comparison logic
    # Not given in the sources. [G]
    return depth1 < depth2

def evaluate_current_splat_error(splat, ref_color, ref_depth):
    # Placeholder for error evaluation of a single splat
    # Not given in the sources. [G]
    return 0.0

def identify_static_voxels(historical_splats, current_splats):
    # Identify static areas: historical splats undergoing deformation mark neighbors as motion
    # Voxels not marked as motion are static
    # Not given in the sources. [G]
    return set()

def blend_static_attributes(splat):
    # Retain 50% of voxel attributes from the past keyframe
    # Not given in the sources. [G]
    pass
```

**Parameters:**
*   **Keyframe Interval:** 5 frames apart [S1].
*   **Keyframe Duration:** 10 frames each [S1].
*   **Static Blend Factor:** 50% (0.5) of voxel attributes from the past keyframe [S1].
*   **Error Threshold:** Not given in the sources. [G]

**Edge Cases and Pitfalls:**
*   **Warping Inaccuracies:** The method specifically targets artifacts from warping errors, appearance variation, and new objects [S1].
*   **Motion Regions:** Historical splats undergoing deformation mark neighboring voxels as motion, preventing incorrect static blending in dynamic areas [S1].
*   **Transformer Errors:** Current frame splats predicted by the multi-view transformer can be erroneous; the error check prevents these from corrupting the scene [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Adds overhead for rendering historical splats and current splats to assess error [S1]. The cost depends on the number of historical splats and views [S1].
*   **Memory Complexity:** Maintains primitives from a set of past keyframes. In practice, two keyframes are stored [S1].
*   **Strengths:** Mitigates warping inaccuracies, reduces temporal flickering, and improves reconstruction quality in dynamic scenes [S1].
*   **Limitations:** Requires rendering multiple passes for error assessment, which increases computational cost [S1]. Relies on the accuracy of the multi-view transformer for current frame prediction [S1].
*   **When to Use:** Dynamic scenes where temporal stability is critical and warping artifacts are prevalent [S1].
*   **When Not to Use:** Static scenes where temporal fusion is unnecessary, or real-time performance is severely constrained [S1].

## Background
*   **Origin:** Introduced in "Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting" [S1].
*   **History:** Developed to address temporal flickering and artifacts in feed-forward 3D Gaussian Splatting for video [S1]. It extends static-scene-trained models to video by merging historical and current frame information [S1].
*   **Usage:** Used in history-aware streaming reconstruction for dynamic 3D Gaussian Splatting [S1].

## Sources
[S1] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
