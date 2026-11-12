# Across View-direction Inconsistency (AVI) Metric

## Overview
The Across View-direction Inconsistency (AVI) metric quantifies the degree of geometrical inconsistency between every pair of test frames corresponding to adjacent viewing directions [S1, unofficial]. It is designed to evaluate the temporal and spatial coherence of rendered volumetric super-resolution outputs by leveraging ground truth optical flow to warp adjacent views against each other [S1, unofficial]. Unlike sibling metrics that may focus on static image quality or temporal stability in video sequences, AVI specifically targets the consistency of 3D geometry reconstruction across different camera viewpoints [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of rendered high-resolution (HR) views $\hat{I}_{sr}$ for a scene; ground truth (GT) optical flow motion vectors for adjacent view pairs; a pre-processing operator $g$.
*   **Output:** A scalar AVI score representing the mean squared error of warped adjacent views, masked for occlusions and normalized for texture.

**Data Structures**
*   **Rendered Views:** $\hat{i}^k_{sr}$, an RGB image for view index $k$.
*   **Optical Flow:** $f_{k \to k+1}$ and $f_{k+1 \to k}$, dense displacement fields mapping pixel coordinates from view $k$ to $k+1$ and vice versa.
*   **Mask:** $M_{k,k+1}$, a binary mask of the same spatial dimensions as the images, where $1$ indicates valid pixels and $0$ indicates masked (occluded/invalid) pixels.

**Steps**
1.  **Pre-processing:** Apply the operator $g$ to each rendered frame $\hat{i}^k_{sr}$ and $\hat{i}^{k+1}_{sr}$ independently.
2.  **Warping:** Warp the image from view $k$ to the coordinate system of view $k+1$ using the backward flow $f_{k+1 \to k}$.
3.  **Mask Generation:**
    a. Compute a raw mask $\tilde{M}_{k,k+1}$ using the forward flow $f_{k \to k+1}$ to identify pixels in view $k+1$ that have no correspondence in view $k$.
    b. Apply morphological closing followed by erosion to $\tilde{M}_{k,k+1}$ to handle sub-pixel occlusions.
4.  **Error Calculation:** Compute the weighted mean squared error between the warped pre-processed image and the pre-processed target image, considering only unmasked pixels.

**Formulas**
The AVI score is defined as [S1, unofficial]:

$$
AVI = \frac{1}{n} \sum_{k=0}^{k_{sr}-1} M_{k,k+1} \cdot \| \text{Warp}(\hat{i}^k_{sr}) - g(\hat{i}^{k+1}) \|_2^2
$$

where:
*   $k_{sr}$ is the number of rendered HR views.
*   $M_{k,k+1}$ is the final error mask.
*   $\text{Warp}(\hat{i}^k_{sr})$ is the warped image from view $k$.
*   $g(\hat{i}^{k+1})$ is the pre-processed image from view $k+1$.
*   $n$ is the number of unmasked elements: $n = \#\{r | M_{k,k+1}(r) = 1\}$.

The warping operation is defined as [S1, unofficial]:

$$
\text{Warp}(\hat{i}^k_{sr})(r) = \hat{i}^k_{sr}(r + f_{k+1 \to k}(r))
$$

where $r$ denotes the 2D pixel location.

The pre-processing operator $g$ is defined as [S1, unofficial]:

$$
g(i) = \text{PatchNorm}(\text{Im2patch}(i))
$$

where $\text{Im2patch}(\cdot)$ extracts all overlapping $7 \times 7$ patches from an RGB image. The $\text{PatchNorm}(p)$ function normalizes patches to have unit standard deviation, discarding smooth patches [S1, unofficial]:

$$
\text{PatchNorm}(p) = \begin{cases} 
0, & \text{if } \text{std}(p) < \epsilon \\
p/\text{std}(p), & \text{otherwise}
\end{cases}
$$

where $\epsilon = 3/255$.

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
    The morphological closing and erosion use a 'cross' structuring element of radius 1.

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
    
    # Iterate over adjacent pairs
    for k in range(len(rendered_views) - 1):
        img_k = rendered_views[k]
        img_k1 = rendered_views[k+1]
        flow_fwd = forward_flows[k] # k -> k+1
        flow_bwd = backward_flows[k] # k+1 -> k
        
        # 1. Pre-processing
        proc_k = apply_preprocessing(img_k, epsilon)
        proc_k1 = apply_preprocessing(img_k1, epsilon)
        
        # 2. Warp img_k to k+1 coordinates using backward flow
        # Warp(r) = img_k(r + flow_bwd(r))
        warped_k = warp_image(img_k, flow_bwd)
        proc_warped_k = apply_preprocessing(warped_k, epsilon)
        
        # 3. Generate Mask
        mask = generate_mask(img_k1.shape, flow_fwd)
        
        # 4. Compute Error
        # Difference between warped pre-processed k and pre-processed k+1
        diff = proc_warped_k - proc_k1
        squared_diff = np.sum(diff ** 2, axis=2) # Sum over RGB channels
        
        # Apply mask
        masked_diff = squared_diff * mask
        
        # Count valid pixels
        n = np.sum(mask)
        
        if n > 0:
            total_error += np.sum(masked_diff)
            total_valid_pixels += n
            
    if total_valid_pixels == 0:
        return 0.0
        
    avI_score = total_error / total_valid_pixels
    return avI_score

def apply_preprocessing(image, epsilon):
    """
    Apply PatchNorm(Im2patch(image)).
    Extracts 7x7 overlapping patches, normalizes std, zeros out smooth patches.
    """
    H, W, C = image.shape
    # Extract patches. Result shape: (H-6, W-6, 7, 7, C)
    patches = extract_patches_2d(image, patch_size=(7, 7))
    
    # Calculate std for each patch
    # std over spatial dimensions (7,7) and channel dimension (3)
    patch_stds = np.std(patches, axis=(3, 4, 5)) # Shape: (H-6, W-6)
    
    # Normalize patches
    # Avoid division by zero
    safe_stds = np.where(patch_stds < epsilon, 1.0, patch_stds)
    normalized_patches = patches / safe_stds[:, :, np.newaxis, np.newaxis, np.newaxis]
    
    # Zero out smooth patches
    normalized_patches = np.where(patch_stds[:, :, np.newaxis, np.newaxis, np.newaxis] < epsilon, 0, normalized_patches)
    
    # Reconstruct image from patches (average overlap)
    # Note: In practice, this requires a weighted sum to handle overlaps correctly.
    # For the purpose of the metric definition, we assume a standard reconstruction.
    reconstructed = reconstruct_from_patches(normalized_patches, patch_size=(7, 7), stride=(1, 1))
    
    return reconstructed

def extract_patches_2d(image, patch_size):
    """Helper to extract overlapping patches."""
    H, W, C = image.shape
    ph, pw = patch_size
    patches = []
    for i in range(H - ph + 1):
        for j in range(W - pw + 1):
            patches.append(image[i:i+ph, j:j+pw, :])
    return np.array(patches).reshape(H-ph+1, W-pw+1, ph, pw, C)

def reconstruct_from_patches(patches, patch_size, stride):
    """Helper to reconstruct image from patches."""
    H, W, C = patches.shape[0:3] # Note: patches shape is (H', W', ph, pw, C)
    ph, pw = patch_size
    out_H = (H - 1) * stride[0] + ph
    out_W = (W - 1) * stride[1] + pw
    
    output = np.zeros((out_H, out_W, C))
    count = np.zeros((out_H, out_W, C))
    
    for i in range(H):
        for j in range(W):
            patch = patches[i, j]
            output[i:i+ph, j:j+pw, :] += patch
            count[i:i+ph, j:j+pw, :] += 1
            
    # Avoid division by zero
    count = np.maximum(count, 1)
    output /= count
    return output

def warp_image(image, flow):
    """
    Warp image using flow.
    Warp(r) = image(r + flow(r))
    """
    H, W, C = image.shape
    # Flow is [H, W, 2], where flow[r] = (dx, dy)
    # Target coordinate r' = r + flow[r]
    # We need to sample image at r'
    
    # Create grid of source coordinates
    y, x = np.ogrid[:H, :W]
    
    # Target coordinates
    ty = y + flow[:, :, 1]
    tx = x + flow[:, :, 0]
    
    # Clip to image boundaries
    ty = np.clip(ty, 0, H - 1)
    tx = np.clip(tx, 0, W - 1)
    
    # Interpolate
    # Using nearest neighbor for simplicity as per floor operation in formula
    # Formula uses floor(q + flow), which implies nearest neighbor sampling
    ty_int = ty.astype(int)
    tx_int = tx.astype(int)
    
    warped = image[ty_int, tx_int, :]
    return warped

def generate_mask(target_shape, forward_flow):
    """
    Generate mask M_k,k+1.
    """
    H, W = target_shape[:2]
    
    # 1. Compute Count
    # Count_{k,k+1}(r) = #{q | floor(q + flow_fwd(q)) == r}
    # q is source coordinate (y, x)
    # r is target coordinate
    
    # Initialize count array
    count = np.zeros((H, W), dtype=int)
    
    # Iterate over source pixels q
    y_src, x_src = np.ogrid[:H, :W]
    
    # Compute target coordinates for each source pixel
    ty_tgt = y_src + forward_flow[:, :, 1]
    tx_tgt = x_src + forward_flow[:, :, 0]
    
    # Floor the coordinates
    ty_floor = np.floor(ty_tgt).astype(int)
    tx_floor = np.floor(tx_tgt).astype(int)
    
    # Clip to valid range
    ty_floor = np.clip(ty_floor, 0, H - 1)
    tx_floor = np.clip(tx_floor, 0, W - 1)
    
    # Count occurrences
    # Use bincount or loop. For efficiency, we can use np.add.at
    # Flatten indices
    flat_ty = ty_floor.ravel()
    flat_tx = tx_floor.ravel()
    flat_indices = flat_ty * W + flat_tx
    
    # Count
    unique, counts = np.unique(flat_indices, return_counts=True)
    count_flat = np.zeros(H * W, dtype=int)
    count_flat[unique] = counts
    count = count_flat.reshape(H, W)
    
    # 2. Raw Mask
    raw_mask = (count > 0).astype(float)
    
    # 3. Morphological Closing then Erosion
    # Structuring element: cross radius 1
    # Cross radius 1: [[0,1,0], [1,1,1], [0,1,0]]
    se = np.array([[0, 1, 0],
                   [1, 1, 1],
                   [0, 1, 0]])
    
    # Closing: Dilation then Erosion
    # Binary dilation
    dilated = binary_closing(raw_mask.astype(bool), structure=se).astype(float) # Note: scipy binary_closing is dilation then erosion
    # Wait, scipy binary_closing is dilation followed by erosion.
    # The formula says "closing followed by erosion".
    # So: closing = dilation -> erosion. Then another erosion.
    # Effectively: dilation -> erosion -> erosion.
    
    # Let's follow the text exactly: "morphological closing followed by erosion"
    # closing(A) = erosion(dilation(A))
    # So: M = erosion(closing(raw)) = erosion(erosion(dilation(raw)))
    
    # Step 3a: Closing
    closed = binary_closing(raw_mask.astype(bool), structure=se).astype(float)
    
    # Step 3b: Erosion
    final_mask = binary_erosion(closed.astype(bool), structure=se).astype(float)
    
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
*   **Time Complexity:** The metric requires computing optical flow (if not provided), warping images, and patch-wise operations. The complexity is dominated by the image processing steps, roughly $O(N \cdot H \cdot W)$ where $N$ is the number of view pairs, and $H, W$ are image dimensions.
*   **Memory Complexity:** Requires storing the rendered views, flow fields, and intermediate masks. The patch extraction and reconstruction can be memory-intensive if not implemented carefully [G].
*   **Performance:** In evaluations on synthetic scenes, the AVI metric showed that volumetric super-resolution methods (AVI score 1.86) performed better than post-2D SR methods (AVI score 2.82) and were comparable to naive pre-2D SR methods (AVI score 1.63) [S1, unofficial].
*   **Strengths:** Provides a quantitative measure of 3D consistency, which is difficult to assess with standard 2D metrics like PSNR or SSIM [S1, unofficial].
*   **Limitations:** Requires ground truth optical flow, which is only available for synthetic datasets or scenes with known geometry [S1, unofficial].

## Background
The AVI metric was introduced in the context of Neural Volume Super-Resolution to evaluate the consistency of rendered views across different camera angles [S1, unofficial]. It was proposed to address the limitation of standard image quality metrics in capturing the geometric coherence of volumetric representations [S1, unofficial].

## Sources
[S1] Neural Volume Super-Resolution, https://doi.org/10.48550/arXiv.2212.04666, fetched 2026-10-30 (unofficial)
