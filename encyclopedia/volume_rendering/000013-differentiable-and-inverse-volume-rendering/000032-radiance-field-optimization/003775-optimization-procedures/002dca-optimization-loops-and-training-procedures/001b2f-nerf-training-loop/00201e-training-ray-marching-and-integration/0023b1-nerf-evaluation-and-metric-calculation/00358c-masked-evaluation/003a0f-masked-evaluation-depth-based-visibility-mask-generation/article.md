# Masked Evaluation (Depth-based Visibility Mask Generation)

Also called: Depth reprojection masking, Invisible Mask Computation, Invisible Region Detection via Depth Warping

## Overview
This method identifies pixels in a reference view that are occluded (invisible) when viewed from a new camera pose by comparing the reference depth map with the depth map warped from the new view [S1]. It is used to generate masks that restrict evaluation or inpainting to visible regions, preventing metric degradation from predicting unseen geometry [S1]. It differs from feature-level occlusion handling by operating directly on rendered depth maps to guide RGBD inpainting or consistency losses [S1][S2, unofficial].

## How it works
**Inputs:**
*   Reference view RGB image $I_{rgb}$ and depth map $I_d$ [S1].
*   New (target) view RGB image $I'_{rgb}$ and depth map $I'_d$ [S1].
*   Camera intrinsics $K$ and relative pose $R_{j \to i}$ between views [S2, unofficial].
*   Threshold $\varepsilon$ for depth difference [S1] or $\tau$ for geometric consistency [S2, unofficial].

**Outputs:**
*   Final invisible mask $m$ (binary, 1 for invisible, 0 for visible) [S1].
*   Masked inputs $I_{rgb}$ and $I_d$ for subsequent processing (e.g., inpainting) [S1].

**Data Structures:**
*   Depth maps: 2D arrays of floating-point values representing distance from camera center [S1][S2, unofficial].
*   Masks: 2D binary arrays of same dimensions as depth maps [S1][S2, unofficial].

**Steps:**
1.  **Warping:** For each pixel $p_j$ in the target view, compute its corresponding coordinate $p_{j \to i}$ in the source view using the warped depth $D_j(p_j)$, camera intrinsics $K$, and relative rotation $R_{j \to i}$ [S2, unofficial]:
    $$ p_{j \to i} \sim K R_{j \to i} D_j(p_j) K^{-1} p_j $$
    where $\sim$ indicates approximate equality due to continuous coordinate projection [S2, unofficial].
2.  **Depth Comparison:** Compare the reference depth $v_{ij}$ (or $D_i(p_{j \to i})$) with the warped depth $v'_{ij}$ (or $D_j(p_j)$) [S1][S2, unofficial].
3.  **Mask Generation:** Apply an indicator function or Iverson bracket to determine visibility [S1][S2, unofficial].
4.  **Aggregation:** Combine masks from multiple views using logical AND [S1].
5.  **Application:** Apply the final mask to RGB and depth inputs [S1].

**Formulas:**
*   **Basic Visibility Mask (Single View Pair):**
    $$ m'_{ij} = \mathbb{I}(\text{warp}(v'_{ij}) - v_{ij} > \varepsilon) $$
    where $m'_{ij}$ is the mask pixel (1 for invisible, 0 for visible), $\text{warp}(\cdot)$ is the warping function, $v_{ij}$ is the reference depth, $v'_{ij}$ is the warped depth, and $\varepsilon$ is a small error tolerance [S1].
*   **Occlusion-Aware Mask (Feature-Level):**
    $$ M(p_j) = [ \| D_j(p_j) - D_i(p_{j \to i}) \| < \tau ] $$
    where $[\cdot]$ is the Iverson bracket, $p_{j \to i}$ is the reprojected pixel, and $\tau$ is a threshold for geometric consistency [S2, unofficial].
*   **Final Invisible Mask:**
    $$ m = \bigwedge_{k} m'_k $$
    where the logic AND operation is performed across all masks $m'_k$ generated from all reference views [S1].
*   **Masked Inputs:**
    $$ \bar{I}_{rgb} = I_{rgb} \otimes (1 - m) $$
    $$ \bar{I}_d = I_d \otimes (1 - m) $$
    where $\otimes$ denotes element-wise multiplication [S1].

**Parameters:**
*   $\varepsilon$: A small value used as a threshold for depth difference to allow for small warping errors [S1]. Value not specified in sources.
*   $\tau$: Threshold to determine if two rays render depths of separate surfaces; if distance $> \tau$, pixel is masked out [S2, unofficial]. Value not specified in sources.

**Edge Cases and Pitfalls:**
*   **Warping Errors:** Erroneous geometry can cause imperfect and distorted warpings, leading to false occlusions [S2, unofficial].
*   **Self-Occlusion:** Regions with self-occlusion may not warp correctly, requiring masking [S2, unofficial].
*   **Artifacts:** Geometric artifacts can cause incorrect warping, necessitating occlusion-aware masking [S2, unofficial].
*   **Large Occlusions:** In complex 3D panoramic scenes, large-size occlusions may occur, which simple adversarial learning or semantic constraints may fail to fill [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_visibility_mask(reference_depth, target_depth, camera_intrinsics, relative_rotation, epsilon):
    """
    Generates an invisible mask by warping target depth to reference view.
    
    Args:
        reference_depth: 2D array of depth values for reference view.
        target_depth: 2D array of depth values for target view.
        camera_intrinsics: 3x3 camera intrinsic matrix K.
        relative_rotation: 3x3 rotation matrix R_j_to_i.
        epsilon: Small threshold for depth difference.
        
    Returns:
        invisible_mask: 2D binary array (1 if invisible, 0 if visible).
    """
    h, w = reference_depth.shape
    invisible_mask = np.zeros((h, w), dtype=bool)
    
    # Create meshgrid for pixel coordinates
    y, x = np.meshgrid(np.arange(h), np.arange(w), indexing='ij')
    # Homogeneous coordinates for target pixels
    target_pixels = np.stack([x, y, np.ones((h, w))], axis=-1) # Shape (h, w, 3)
    
    # Warp target depth to reference view
    # Note: This is a simplified warping step. In practice, this involves
    # projecting 3D points from target depth to reference camera coordinates.
    # 1. Convert target depth to 3D points in target camera frame
    # 2. Transform 3D points to reference camera frame using relative_rotation
    # 3. Project 3D points to reference image plane
    # 4. Sample reference depth at projected coordinates
    
    # Placeholder for actual warping logic based on [S2, unofficial] Eq. 6
    # p_j_to_i ~ K * R_j_to_i * D_j(p_j) * K^-1 * p_j
    # This requires converting pixel coordinates to 3D, transforming, and re-projecting.
    
    # For the purpose of this pseudocode, we assume a function `warp_depth` exists
    # that performs the geometric transformation described in [S1][S2, unofficial].
    warped_depth = warp_depth(target_depth, camera_intrinsics, relative_rotation)
    
    # Compare warped depth with reference depth
    # Invisible if warped depth > reference depth + epsilon
    # Note: The formula in [S1] is m'_ij = I(warp(v'_ij) - v_ij > epsilon)
    # This implies that if the warped point is further away than the visible surface,
    # it is occluded (invisible) in the reference view.
    
    depth_diff = warped_depth - reference_depth
    invisible_mask = depth_diff > epsilon
    
    return invisible_mask

def generate_occlusion_mask(target_depth, source_depth, camera_intrinsics, relative_rotation, tau):
    """
    Generates an occlusion-aware mask for feature-level consistency.
    
    Args:
        target_depth: 2D array of depth values for target view.
        source_depth: 2D array of depth values for source view.
        camera_intrinsics: 3x3 camera intrinsic matrix K.
        relative_rotation: 3x3 rotation matrix R_j_to_i.
        tau: Threshold for geometric consistency.
        
    Returns:
        occlusion_mask: 2D binary array (1 if consistent/visible, 0 if occluded).
    """
    h, w = target_depth.shape
    occlusion_mask = np.zeros((h, w), dtype=bool)
    
    # Warp source depth to target view (or vice versa, depending on convention)
    # [S2, unofficial] compares rendered depth D_j at target view with reprojected depth D_i from source view.
    # So we warp source depth to target view coordinates.
    warped_source_depth = warp_depth(source_depth, camera_intrinsics, np.linalg.inv(relative_rotation))
    
    # Calculate Euclidean distance between depths
    depth_diff = np.abs(target_depth - warped_source_depth)
    
    # Mask is 1 if distance < tau (consistent), 0 otherwise
    occlusion_mask = depth_diff < tau
    
    return occlusion_mask

def apply_mask(image, mask):
    """
    Applies mask to image.
    
    Args:
        image: Input image (RGB or Depth).
        mask: Binary mask (1 for keep, 0 for mask out).
        
    Returns:
        masked_image: Image with masked regions zeroed out.
    """
    return image * mask
```

## Subtopics

### Occlusion-Aware Masking
A technique to generate a consistency mask $M_l$ that ignores regions with geometric inconsistencies (artifacts, self-occlusion) during feature-level regularization [S2, unofficial]. It differs from basic depth reprojection masking by applying the mask to feature maps after nearest-neighbor downsampling, rather than directly to RGB/depth images [S2, unofficial]. It is chosen when geometric consistency is enforced at the feature level to prevent distorted warpings from degrading reconstruction quality [S2, unofficial]. See [GeCoNeRF](https://core.ac.uk/works/309257742).

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the image, as it involves per-pixel depth comparison and warping [S1][S2, unofficial].
*   **Memory Complexity:** Linear with respect to the number of pixels, requiring storage for depth maps and masks [S1][S2, unofficial].
*   **Strengths:** Effectively identifies invisible regions for inpainting [S1]; prevents geometric inconsistencies from affecting feature consistency [S2, unofficial].
*   **Limitations:** Sensitive to warping errors and geometric artifacts [S2, unofficial]; may fail to handle large occlusions without additional inpainting strategies [S1].
*   **When to Use:** When evaluating or inpainting radiance fields with multiple views, especially in complex scenes with occlusions [S1][S2, unofficial].
*   **When Not to Use:** When single-view geometry is sufficient or when occlusion handling is not a priority [S1].

## Background
Originated in the context of NeRF training and inpainting, specifically for handling invisible regions in panoramic scenes [S1]. Developed to address the failure of adversarial learning or semantic constraints in filling large-size occlusions [S1]. Also used in few-shot NeRF methods to enforce geometric consistency [S2, unofficial].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
[S2] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
