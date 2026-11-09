# Across View-direction Inconsistency (AVI) Metric

## Overview
The Across View-direction Inconsistency (AVI) metric quantifies the degree of geometrical inconsistency between every pair of test frames corresponding to adjacent viewing directions in a rendered sequence [S1, unofficial]. It is designed to evaluate the temporal and spatial coherence of volume rendering methods, particularly in the context of super-resolution, by measuring how well adjacent views align when warped to one another [S1, unofficial]. Unlike standard image quality metrics such as PSNR or SSIM which evaluate individual frames in isolation, AVI explicitly penalizes artifacts like ghosting and jitter that arise from inconsistent geometry across the view space [S1, unofficial].

## How it works
The metric computes a mean squared error between a rendered frame and its neighbor warped into the same coordinate system, restricted to a mask that excludes occluded regions.

**Inputs and Outputs:**
*   **Inputs:** A set of rendered high-resolution (HR) views $\hat{I}_{sr} = \{\hat{i}^k_{sr}\}_{k=0}^{k_{sr}-1}$ for a specific scene, where $k_{sr}$ is the number of test frames. Ground truth (GT) optical flow vectors for forward flow $f_{k \to k+1}$ and backward flow $f_{k+1 \to k}$ between all adjacent view-direction pairs $k, k+1$ [S1, unofficial].
*   **Output:** A scalar AVI score representing the average inconsistency [S1, unofficial].

**Data Structures:**
*   **Images:** $\hat{i}^k_{sr}$ is an RGB image of resolution $H \times W$.
*   **Flow:** $f_{k \to k+1}(q)$ returns a 2D vector $(u, v)$ for pixel $q$.
*   **Mask:** $M_{k,k+1}$ is a binary mask of size $H \times W$.

**Pre-processing Operator $g$:**
To make the metric robust to texture smoothness and noise, a pre-processing operator $g$ is applied to each frame independently before calculating differences [S1, unofficial].
1.  **Patch Extraction:** $\text{Im2patch}(i)$ extracts all overlapping $7 \times 7$ patches from the RGB image $i$ [S1, unofficial].
2.  **Normalization:** $\text{PatchNorm}(p)$ processes each patch $p$:
    $$
    \text{PatchNorm}(p) = \begin{cases} 
    0, & \text{if } \text{std}(p) < \epsilon \\
    p / \text{std}(p), & \text{otherwise}
    \end{cases}
    $$
    where $\epsilon = 3/255$ [S1, unofficial]. Patches with standard deviation below $\epsilon$ are considered smooth/noisy and zeroed out; others are normalized to unit standard deviation [S1, unofficial].
    $$g(i) = \text{PatchNorm}(\text{Im2patch}(i))$$

**Mask Generation:**
An error mask $M_{k,k+1}$ is computed to exclude pixels suffering from ghosting artifacts due to occlusions [S1, unofficial].
1.  **Raw Mask $\tilde{M}_{k,k+1}$:** Using the forward flow $f_{k \to k+1}$, count how many pixels from view $k$ map to pixel $r$ in view $k+1$:
    $$\text{Count}_{k,k+1}(r) = \#\{q \mid \lfloor q + f_{k \to k+1}(q) \rfloor = r\}$$
    Pixels in view $k+1$ with no correspondence in view $k$ are masked out:
    $$
    \tilde{M}_{k,k+1}(r) = \begin{cases} 
    0, & \text{if } \text{Count}_{k,k+1}(r) = 0 \\
    1, & \text{otherwise}
    \end{cases}
    $$
2.  **Morphological Processing:** To handle subpixel occlusions caused by the floor operation, apply morphological closing followed by erosion to the raw mask using a 'cross' structuring element of radius 1 [S1, unofficial]:
    $$M_{k,k+1} = \text{erosion}(\text{closing}(\tilde{M}_{k,k+1}))$$

**Formula:**
The AVI score is the mean squared error between the warped pre-processed image from view $k$ and the pre-processed image from view $k+1$, averaged over all unmasked pixels $n$ across all adjacent pairs [S1, unofficial]:
$$
AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(g(\hat{i}^k_{sr})) - g(\hat{i}^{k+1}) \|_2^2
$$
where $n = \#\{r \mid M_{k,k+1}(r) = 1\}$ is the number of unmasked elements [S1, unofficial]. The warping operation uses the backward flow to map view $k$ to view $k+1$:
$$\text{Warp}(\hat{i}^k_{sr})(r) = \hat{i}^k_{sr}(r + f_{k+1 \to k}(r))$$
Note: The source defines the warp on the raw image $\hat{i}^k_{sr}$ but applies the norm to the pre-processed versions. The formula in the source text [S1, unofficial] is:
$$AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2$$
However, the text states "apply a pre-processing operator $g$ on each of the two frames independently, before calculating the difference". The formula in Eq. 6 of the source [S1, unofficial] actually shows:
$$AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(g(\hat{i}^k_{sr})) - g(\hat{i}^{k+1}) \|_2^2$$
Wait, looking closely at the source snippet: "$\| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2$". But earlier it says "apply a pre-processing operator $g$ on each of the two frames independently". Let's re-read carefully.
Source: "To this end, we apply a pre-processing operator $g$ on each of the two frames independently, before calculating the difference between them... The AVI for each scene is computed... $AVI = \frac{1}{n} \sum ... \| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2$".
There is a slight ambiguity in the source formula notation vs text. The text says "apply $g$ on each... before calculating". The formula shows $g$ on the second term but not explicitly on the first term inside the warp, or perhaps $\text{Warp}$ implies the warping of the pre-processed image. Given the context of "independently", it is safest to assume both are pre-processed. However, the explicit formula in the source [S1, unofficial] is:
$$AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2$$
Actually, looking at Eq 6 in the source [S1, unofficial] provided in the notes:
"$AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2$"
But the text immediately preceding says "apply a pre-processing operator $g$ on each of the two frames independently". It is highly likely the first term should also be $g(\hat{i}^k_{sr})$. Let's look at the warping definition. $\text{Warp}(\hat{i}^k_{sr})(r) = \hat{i}^k_{sr}(r + f_{k+1 \to k}(r))$. This warps the raw image. If we compare raw warped to pre-processed, the scales differ. The most consistent interpretation of "apply $g$ on each... before calculating" is that both inputs to the norm are pre-processed. I will implement both as pre-processed to satisfy the textual description, noting the source formula ambiguity.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_avi(rendered_views, gt_forward_flow, gt_backward_flow):
    """
    Compute the Across View-direction Inconsistency (AVI) metric.
    
    Args:
        rendered_views: List of rendered HR RGB images [i^0, i^1, ..., i^{k_sr-1}].
        gt_forward_flow: List of forward flow fields [f_{0->1}, f_{1->2}, ...].
        gt_backward_flow: List of backward flow fields [f_{1->0}, f_{2->1}, ...].
        
    Returns:
        avl_score: Scalar AVI score.
    """
    epsilon = 3.0 / 255.0
    total_error = 0.0
    total_unmasked_pixels = 0
    
    k_sr = len(rendered_views)
    
    for k in range(k_sr - 1):
        img_k = rendered_views[k]
        img_k_plus_1 = rendered_views[k + 1]
        
        # 1. Pre-processing Operator g
        # Extract 7x7 overlapping patches and normalize
        patches_k = extract_patches(img_k, patch_size=7)
        patches_k_norm = patch_normalize(patches_k, epsilon)
        
        patches_k1 = extract_patches(img_k_plus_1, patch_size=7)
        patches_k1_norm = patch_normalize(patches_k1, epsilon)
        
        # 2. Mask Generation
        # Use forward flow f_{k->k+1} to determine correspondence
        flow_f = gt_forward_flow[k]
        raw_mask = compute_raw_mask(img_k, flow_f)
        
        # Morphological closing followed by erosion with cross structuring element radius 1
        mask = morphological_close(raw_mask, structuring_element='cross', radius=1)
        mask = morphological_erosion(mask, structuring_element='cross', radius=1)
        
        # 3. Warping
        # Use backward flow f_{k+1->k} to warp img_k to the coordinate system of img_k+1
        flow_b = gt_backward_flow[k]
        warped_img_k = warp_image(img_k, flow_b)
        
        # Note: The source text says apply g on each frame independently before calculating difference.
        # The formula in source [S1, unofficial] shows Warp(i^k) - g(i^{k+1}).
        # However, to be consistent with "apply g on each", we apply g to the warped image as well.
        # If the source strictly meant raw warped vs pre-processed neighbor, the scales would mismatch.
        # We assume both are pre-processed for the error calculation.
        warped_patches = extract_patches(warped_img_k, patch_size=7)
        warped_patches_norm = patch_normalize(warped_patches, epsilon)
        
        # 4. Error Calculation
        # Compute MSE between pre-processed warped image and pre-processed next image
        # Only over unmasked pixels
        error_map = (warped_patches_norm - patches_k1_norm) ** 2
        # Reshape to image grid for masking
        error_img = patches_to_image(error_map, img_k.shape)
        
        # Apply mask
        masked_error = error_img * mask
        
        # Count unmasked pixels
        n = np.sum(mask == 1)
        
        if n > 0:
            total_error += np.sum(masked_error)
            total_unmasked_pixels += n
            
    if total_unmasked_pixels == 0:
        return 0.0
        
    avl_score = total_error / total_unmasked_pixels
    return avl_score

def extract_patches(image, patch_size):
    """Extract all overlapping patches of size patch_size x patch_size."""
    # Implementation detail: standard sliding window extraction
    pass

def patch_normalize(patches, epsilon):
    """Normalize patches by standard deviation, zeroing out smooth patches."""
    stds = np.std(patches, axis=(1, 2, 3), keepdims=True) # Assuming batched patches
    # Handle epsilon comparison
    mask_smooth = stds < epsilon
    normalized = np.where(mask_smooth, 0, patches / stds)
    return normalized

def compute_raw_mask(img, flow):
    """Compute raw mask based on forward flow mapping."""
    H, W = img.shape[:2]
    mask = np.zeros((H, W), dtype=int)
    
    # Iterate over source pixels q
    for y in range(H):
        for x in range(W):
            q = (x, y)
            # Get flow vector
            u, v = flow[y, x]
            # Map to destination
            dest_x = x + u
            dest_y = y + v
            # Floor operation
            r_x = int(np.floor(dest_x))
            r_y = int(np.floor(dest_y))
            
            if 0 <= r_y < H and 0 <= r_x < W:
                # Increment count for this destination pixel
                # Note: The source counts how many source pixels map to dest
                # We can use a counter array
                pass
                
    # Re-implementing the counting logic efficiently
    count_map = np.zeros((H, W), dtype=int)
    for y in range(H):
        for x in range(W):
            u, v = flow[y, x]
            r_x = int(np.floor(x + u))
            r_y = int(np.floor(y + v))
            if 0 <= r_y < H and 0 <= r_x < W:
                count_map[r_y, r_x] += 1
                
    # Set raw mask: 0 if count is 0, else 1
    raw_mask = (count_map > 0).astype(int)
    return raw_mask

def warp_image(img, flow):
    """Warp image using backward flow."""
    H, W = img.shape[:2]
    warped = np.zeros_like(img)
    for y in range(H):
        for x in range(W):
            u, v = flow[y, x]
            src_x = x + u
            src_y = y + v
            # Bilinear interpolation or nearest neighbor for sampling
            # Source uses simple indexing in formula, but warping usually implies interpolation
            # Source formula: i^k(r + f(r))
            # We assume standard image warping
            warped[y, x] = sample_at(img, src_x, src_y)
    return warped

def sample_at(img, x, y):
    """Sample image at float coordinates."""
    # Nearest neighbor or bilinear
    pass
```

**Edge Cases and Pitfalls:**
*   **Subpixel Occlusions:** The floor operation in the count computation can lead to ghosting artifacts. Morphological closing and erosion are required to mitigate this [S1, unofficial].
*   **Smooth Patches:** Patches with standard deviation below $\epsilon = 3/255$ are dominated by noise and are zeroed out to prevent them from dominating the metric [S1, unofficial].
*   **Ground Truth Flow:** The metric requires access to ground truth optical flow, which is only available for synthetic datasets where camera poses and geometry are known [S1, unofficial].

## Operating characteristics
*   **Complexity:** The metric requires $O(k_{sr} \cdot H \cdot W)$ operations for flow warping and patch extraction, where $k_{sr}$ is the number of test frames and $H, W$ are image dimensions.
*   **Performance:** In evaluations on 800 test frames across four synthetic scenes, the AVI score for the proposed Volume SR method was 1.86, compared to 2.82 for SwinIR (post-2D SR) and 1.63 for Naive (Planes pre-2D SR) [S1, unofficial]. Lower scores indicate better consistency.
*   **Strengths:** It effectively quantifies geometric consistency across views, highlighting the advantage of 3D-domain methods over 2D post-processing super-resolution [S1, unofficial].
*   **Limitations:** It relies on ground truth optical flow, limiting its applicability to synthetic scenes or scenarios where GT flow is available [S1, unofficial].

## Background
The AVI metric was introduced in the paper "Neural Volume Super-Resolution" [S1, unofficial]. It was developed to quantitatively evaluate the advantage of volume-based super-resolution approaches over 2D post-processing methods in maintaining geometric consistency across adjacent viewing directions. The metric leverages ground truth optical flow from synthetic datasets (such as those generated from Blender models) to compute the inconsistency [S1, unofficial].

## Sources
[S1] Neural Volume Super-Resolution, https://doi.org/10.48550/arXiv.2212.04666, fetched 2026-10-30 (unofficial)
