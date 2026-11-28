# Across View-direction Inconsistency (AVI) Metric


## Overview
The Across View-direction Inconsistency (AVI) metric quantifies the degree of geometrical inconsistency between every pair of test frames corresponding to adjacent viewing directions [S1, unofficial]. It is designed to evaluate the spatial and geometric coherence of rendered volumetric super-resolution outputs by leveraging ground truth optical flow to warp adjacent views against each other [S1, unofficial]. Unlike sibling metrics that focus on static image quality or temporal stability in video sequences, AVI specifically targets the consistency of 3D geometry reconstruction across different camera viewpoints [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of rendered high-resolution (HR) views $\hat{I}_{sr}$ for a scene; ground truth (GT) optical flow motion vectors for adjacent view pairs; a pre-processing operator $g$ [S1, unofficial].
*   **Output:** A scalar AVI score representing the mean squared error of warped adjacent views, masked for occlusions and normalized for texture [S1, unofficial].

**Data Structures**
*   **Rendered Views:** $\hat{i}^k_{sr}$, an RGB image for view index $k$ [S1, unofficial].
*   **Optical Flow:** $f_{k \to k+1}$ and $f_{k+1 \to k}$, dense displacement fields mapping pixel coordinates from view $k$ to $k+1$ and vice versa [S1, unofficial].
*   **Mask:** $M_{k,k+1}$, a binary mask of the same spatial dimensions as the images, where $1$ indicates valid pixels and $0$ indicates masked (occluded/invalid) pixels [S1, unofficial].

**Steps**
1.  **Pre-processing:** Apply the operator $g$ to the target frame $\hat{i}^{k+1}_{sr}$ independently [S1, unofficial].
2.  **Warping:** Warp the image from view $k$ to the coordinate system of view $k+1$ using the backward flow $f_{k+1 \to k}$ [S1, unofficial].
3.  **Mask Generation:**
    a. Compute a raw mask $\tilde{M}_{k,k+1}$ using the forward flow $f_{k \to k+1}$ to identify pixels in view $k+1$ that have no correspondence in view $k$ [S1, unofficial].
    b. Apply morphological closing followed by erosion to $\tilde{M}_{k,k+1}$ to handle sub-pixel occlusions [S1, unofficial].
4.  **Error Calculation:** Compute the weighted mean squared error between the warped image from view $k$ and the pre-processed target image from view $k+1$, considering only unmasked pixels [S1, unofficial].

**Formulas**
The AVI score is defined as [S1, unofficial]:

$$
AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2
$$

where:
*   $k_{sr}$ is the number of rendered HR views [S1, unofficial].
*   $M_{k,k+1}$ is the final error mask [S1, unofficial].
*   $\text{Warp}(\hat{i}^k_{sr})$ is the warped image from view $k$ [S1, unofficial].
*   $g(\hat{i}^{k+1})$ is the pre-processed image from view $k+1$ [S1, unofficial].
*   $n$ is the number of unmasked elements: $n = \#\{r | M_{k,k+1}(r) = 1\}$ [S1, unofficial].

The warping operation is defined as [S1, unofficial]:

$$
\text{Warp}(\hat{i}^k_{sr})(r) = \hat{i}^k_{sr}(r + f_{k+1 \to k}(r))
$$

where $r$ denotes the 2D pixel location.

The pre-processing operator $g$ is defined as [S1, unofficial]:

$$
g(i) = \text{PatchNorm}(\text{Im2patch}(i))
$$

where $\text{Im2patch}(\cdot)$ extracts all overlapping $7 \times 7$ patches from an RGB image [S1, unofficial]. The $\text{PatchNorm}(p)$ function normalizes patches to have unit standard deviation, discarding smooth patches [S1, unofficial]:

$$
\text{PatchNorm}(p) = \begin{cases} 
0, & \text{if } \text{std}(p) < \epsilon \\
p/\text{std}(p), & \text{otherwise}
\end{cases}
$$

where $\epsilon = 3/255$ [S1, unofficial].

The mask generation proceeds as follows [S1, unofficial]:
1.  Compute the count of pixels mapping to each location in the target view:
    $$
    \text{Count}_{k,k+1}(r) = \#\{q | \lfloor q + f_{k \to k+1}(q) \rfloor = r\}
    $$
2.  Create the raw mask $\tilde{M}_{k,k+1}$:
    $$
    \tilde{M}_{k,k+1}(r) = \begin{cases} 
    0, & \text{if } \text{Count}_{k,k+1}(r) = 0 \\
    1, & \text{otherwise}
    \end{cases}
    $$
3.  Apply morphological operations to obtain the final mask $M_{k,k+1}$:
    $$
    M_{k,k+1} = \text{erosion}(\text{closing}(\tilde{M}_{k,k+1}))
    $$
    The morphological closing and erosion use a 'cross' structuring element of radius 1 [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.ndimage import binary_closing, binary_erosion

def compute_avI(rendered_views, forward_flows, backward_flows, epsilon=3/255):
    """
    Compute the Across View-direction Inconsistency (AVI) metric.
    
    Args:
        rendered_views: List of numpy arrays [H, W, 3] for each view k.
        forward_flows: List of numpy arrays [H, W, 2] for flow k->k+1.
        backward_flows: List of numpy arrays [H, W, 2] for flow k+1->k.
        epsilon: Threshold for patch normalization.
        
    Returns:
        avI_score: Scalar float.
    """
    total_error = 0.0
    total_valid_pixels = 0
    
    for k in range(len(rendered_views) - 1):
        img_k = rendered_views[k]
        img_k1 = rendered_views[k+1]
        flow_fwd = forward_flows[k]
        flow_bwd = backward_flows[k]
        
        # 1. Pre-processing: apply g only to the target frame k+1
        proc_k1 = apply_preprocessing(img_k1, epsilon)
        
        # 2. Warp img_k to k+1 coordinates using backward flow
        warped_k = warp_image(img_k, flow_bwd)
        
        # 3. Generate Mask
        mask = generate_mask(img_k1.shape, flow_fwd)
        
        # 4. Compute Error: ||Warp(i^k) - g(i^{k+1})||_2^2
        diff = warped_k - proc_k1
        squared_diff = np.sum(diff ** 2, axis=2)
        
        masked_diff = squared_diff * mask
        n = np.sum(mask)
        
        if n > 0:
            total_error += np.sum(masked_diff)
            total_valid_pixels += n
            
    if total_valid_pixels == 0:
        return 0.0
        
    return total_error / total_valid_pixels

def apply_preprocessing(image, epsilon):
    """Apply PatchNorm(Im2patch(image))."""
    H, W, C = image.shape
    ph, pw = 7, 7
    # Extract overlapping 7x7 patches
    patches = []
    for i in range(H - ph + 1):
        for j in range(W - pw + 1):
            patches.append(image[i:i+ph, j:j+pw, :])
    patches = np.array(patches).reshape(H-ph+1, W-pw+1, ph, pw, C)
    
    # Calculate std for each patch over spatial and channel dimensions
    patch_stds = np.std(patches, axis=(3, 4, 5))
    
    # Normalize patches, zero out smooth ones
    safe_stds = np.where(patch_stds < epsilon, 1.0, patch_stds)
    normalized_patches = patches / safe_stds[:, :, np.newaxis, np.newaxis, np.newaxis]
    normalized_patches = np.where(patch_stds[:, :, np.newaxis, np.newaxis, np.newaxis] < epsilon, 0, normalized_patches)
    
    # Reconstruct image by averaging overlapping patches
    out_H, out_W = H, W
    reconstructed = np.zeros((out_H, out_W, C))
    count = np.zeros((out_H, out_W, C))
    for i in range(H - ph + 1):
        for j in range(W - pw + 1):
            reconstructed[i:i+ph, j:j+pw, :] += normalized_patches[i, j]
            count[i:i+ph, j:j+pw, :] += 1
    count = np.maximum(count, 1)
    reconstructed /= count
    return reconstructed

def warp_image(image, flow):
    """Warp image using flow: Warp(r) = image(r + flow(r))."""
    H, W, C = image.shape
    y, x = np.ogrid[:H, :W]
    ty = y + flow[:, :, 1]
    tx = x + flow[:, :, 0]
    ty = np.clip(ty, 0, H - 1).astype(int)
    tx = np.clip(tx, 0, W - 1).astype(int)
    return image[ty, tx, :]

def generate_mask(target_shape, forward_flow):
    """Generate mask M_k,k+1."""
    H, W = target_shape[:2]
    count = np.zeros((H, W), dtype=int)
    y_src, x_src = np.ogrid[:H, :W]
    ty_tgt = y_src + forward_flow[:, :, 1]
    tx_tgt = x_src + forward_flow[:, :, 0]
    ty_floor = np.floor(ty_tgt).astype(int)
    tx_floor = np.floor(tx_tgt).astype(int)
    ty_floor = np.clip(ty_floor, 0, H - 1)
    tx_floor = np.clip(tx_floor, 0, W - 1)
    flat_ty = ty_floor.ravel()
    flat_tx = tx_floor.ravel()
    flat_indices = flat_ty * W + flat_tx
    unique, counts = np.unique(flat_indices, return_counts=True)
    count_flat = np.zeros(H * W, dtype=int)
    count_flat[unique] = counts
    count = count_flat.reshape(H, W)
    raw_mask = (count > 0).astype(bool)
    se = np.array([[0, 1, 0], [1, 1, 1], [0, 1, 0]])
    closed = binary_closing(raw_mask, structure=se)
    final_mask = binary_erosion(closed, structure=se).astype(float)
    return final_mask
```

**Parameters**
*   $\epsilon = 3/255$: Minimum standard deviation threshold for patch normalization [S1, unofficial].
*   Patch size: $7 \times 7$ [S1, unofficial].
*   Structuring element: 'Cross' of radius 1 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Occlusions:** The mask generation is critical to prevent ghosting artifacts from dominating the metric. Sub-pixel occlusions are handled by the morphological operations [S1, unofficial].
*   **Smooth Regions:** Patches with standard deviation below $\epsilon$ are zeroed out to prevent noise from influencing the metric [S1, unofficial].
*   **Boundary Conditions:** The warping and patch extraction must handle image boundaries correctly. The pseudocode clips coordinates to valid ranges [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** In evaluations on synthetic scenes, the AVI metric showed that volumetric super-resolution methods (AVI score 1.86) performed better than post-2D SR methods (AVI score 2.82) and were comparable to naive pre-2D SR methods (AVI score 1.63) [S1, unofficial].
*   **Strengths:** Provides a quantitative measure of 3D consistency, which is difficult to assess with standard 2D metrics like PSNR or SSIM [S1, unofficial].
*   **Limitations:** Requires ground truth optical flow, which is only available for synthetic datasets or scenes with known geometry [S1, unofficial].

## Background
The AVI metric was introduced in the context of Neural Volume Super-Resolution to evaluate the consistency of rendered views across different camera angles [S1, unofficial]. It was proposed to address the limitation of standard image quality metrics in capturing the geometric coherence of volumetric representations [S1, unofficial].

## Sources
[S1] Neural Volume Super-Resolution, https://doi.org/10.48550/arXiv.2212.04666, fetched 2026-10-30 (unofficial)
