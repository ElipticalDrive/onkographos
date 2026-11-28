# Light Field Tomographic Reconstruction from Focal Stack

Also called: Epipolar Image Reconstruction, Masked Back-projection, Partial light field tomographic reconstruction

## Overview
This method reconstructs a 2D epipolar image (a slice of the 4D light field) from a focal stack—a set of 2D images of the same scene captured at different focus distances by a fixed camera [S1, unofficial]. It treats each photograph as a 2D projection of the light field and applies tomographic back-projection restricted to in-focus regions, estimating projection angles from a depth map derived via focus cues [S1, unofficial]. Unlike spectral averaging approaches that deconvolve in the frequency domain [S1, unofficial], this method operates in the spatial domain, exploiting the regular linear structure of epipolar images to achieve high angular resolution with a limited number of projections [S1, unofficial].

## How it works
**Inputs:** A focal stack $\{I_k\}_{k=1}^K$ of $K$ images captured by a fixed camera at different focus distances, and camera calibration parameters (focal length $f_0$ and reference distance $D_{ref}$).
**Outputs:** A reconstructed 2D epipolar image $LF(x, y, u, 0)$, representing the light field subset sufficient for refocusing and perspective shifts.

**Data Structures:**
1.  **Focal Stack:** Array of $K$ images $I_k$ of size $W \times H$.
2.  **Focus Map:** Dense map $l$ of size $W \times H$, where $l_{x,y} \in \{1, \dots, K\}$ indicates the index of the image $I_k$ where pixel $(x,y)$ is in focus.
3.  **Depth Map:** Dense map $D$ of size $W \times H$, where $D_{x,y}$ is the physical distance to the object at pixel $(x,y)$.
4.  **Sharpness Layers:** $K$ intermediate layers $F_k$ of size $W \times H$, storing sharpness scores.
5.  **Epipolar Image:** Output buffer $LF$ of size $W \times U$ (where $U$ is the angular dimension), initialized to background values.

**Parameters:**
*   $K$: Number of images in the focal stack.
*   $M$: Number of gradient thresholds used in sharpness scoring.
*   $\alpha_m$: Weighting coefficients for the $m$-th threshold.
*   $\delta_k$: Gradient threshold for image $I_k$, typically selected automatically by an edge detection algorithm.
*   $f_0$: Focal length of the lens.
*   $D_{ref}$: Distance to the reference object (green plane), chosen to lie at the reference focal distance.

**Steps:**

1.  **Compute Sharpness Scores:** For each image $I_k$ in the focal stack, compute a sharpness score layer $F_k(x,y)$ by summing weighted unit step functions of the image intensity gradient magnitude against increasing thresholds [S1, unofficial]:
    $$
    F_k(x,y) = \sum_{m=1}^{M} \alpha_m u(|\nabla I_k(x,y)| - m\delta_k)
    $$
    where $u(\cdot)$ is the unit step function and $\nabla I_k(x,y)$ is the image intensity gradient.

2.  **Estimate Focus Map:** Compute the focus map $l$ by minimizing the objective function $E(l)$ using a graph cut algorithm [S1, unofficial]:
    $$
    E(l) = \sum_{X} D(X, l_X) + \sum_{X \sim X'} S(l_X, l_{X'})
    $$
    where $X=(x,y)$ is a pixel, $X \sim X'$ denotes neighboring pixels, $l_X$ is the label (image index) assigned to pixel $X$, $D$ is the data term, and $S$ is the smoothness term.
    *   **Data Term:** $D(X, l_X) = \sum_{m=1}^{M} \alpha_m (1 - F_{l_X}(x,y))$. This cost decreases as the sharpness score of the assigned image increases.
    *   **Smoothness Term:**
        $$
        S(l_X, l_{X'}) = \begin{cases} 
        0 & \text{if } l_X = l_{X'} \\
        0.5 + \log(|l_X - l_{X'}|) & \text{if } l_X \neq l_{X'}
        \end{cases}
        $$
        The logarithmic cost avoids over-penalizing depth discontinuities. The minimization yields a dense focus map $l$ where $l_X$ is the index of the sharpest image for pixel $X$.

3.  **Convert to Depth Map:** Transform the discrete focus map $l$ into a continuous depth map $D$ using the camera calibration parameters and capture metadata [S1, unofficial]. The precision of $D$ is limited by $K$ (the number of focal planes), but the map is dense due to the graph cut diffusion.

4.  **Calculate Back-projection Angles:** For each pixel $(x,y)$ in the epipolar image, estimate the back-projection angle $\theta$ based on the depth value $D_{x,y}$ from the depth map [S1, unofficial]:
    $$
    \theta = \arctan \left( \frac{1/D_{ref} - 1/D_{x,y}}{1/f_0 - 1/D_{ref}} \right)
    $$
    Here, $\tan \theta = u/x$, relating the angular coordinate $u$ to the spatial coordinate $x$.

5.  **Masked Back-projection:** Reconstruct the epipolar image $LF(x, y, u, 0)$ by back-projecting pixels from the focal stack images according to their estimated angles, using masks to restrict contributions to in-focus regions [S1, unofficial].
    *   **Background:** Back-project all pixels from every image in the focal stack entirely, as the background is assumed to be in focus for all or most images or serves as the base layer.
    *   **Foreground/Objects:** For non-background regions, back-project **only** the in-focus pixels. Determine in-focus regions by creating a binary mask for each image $I_k$ where pixels satisfy $l_X = k$.
    *   **Ordering:** Process objects in decreasing order of depth (from farthest to nearest).
    *   **Overwriting:** Instead of summing contributions (as in classical Radon back-projection), **overwrite** the values in the epipolar image buffer $LF$. This ensures that closer objects occlude distant ones, consistent with physical reality and the linear structure of epipolar images.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.ndimage import sobel

def compute_sharpness_scores(focal_stack, alpha, delta):
    """
    Compute sharpness score layers for each image in the focal stack.
    focal_stack: List of K images (H, W)
    alpha: Array of M weights
    delta: Array of K thresholds (one per image)
    Returns: List of K sharpness layers (H, W)
    """
    K = len(focal_stack)
    M = len(alpha)
    sharpness_layers = []
    
    for k in range(K):
        I_k = focal_stack[k]
        # Compute gradient magnitude
        # Using Sobel for X and Y gradients
        grad_x = sobel(I_k, axis=0)
        grad_y = sobel(I_k, axis=1)
        grad_mag = np.sqrt(grad_x**2 + grad_y**2)
        
        # Compute sharpness score
        F_k = np.zeros_like(I_k)
        for m in range(M):
            threshold = (m + 1) * delta[k]
            # Unit step function: 1 if grad_mag > threshold, else 0
            step = (grad_mag > threshold).astype(float)
            F_k += alpha[m] * step
        sharpness_layers.append(F_k)
        
    return sharpness_layers

def estimate_focus_map(sharpness_layers, K):
    """
    Estimate focus map using Graph Cut minimization.
    Note: This is a conceptual implementation. A real implementation 
    would use a library like OpenCV or a dedicated Graph Cut solver.
    """
    H, W = sharpness_layers[0].shape
    # Initialize focus map with random labels or argmax of sharpness
    # Data cost D(X, l_X) = sum(alpha_m * (1 - F_lX))
    # We want to minimize E(l). 
    # For simplicity, we assume a pre-existing graph cut function `graph_cut_minimize`
    # that takes data costs and smoothness penalties.
    
    # Construct data cost matrix: (H, W, K)
    # D[x, y, k] = cost of assigning label k to pixel (x,y)
    data_costs = np.zeros((H, W, K))
    for k in range(K):
        # D(X, l_X) = sum_m alpha_m * (1 - F_k(x,y))
        # Since alpha and 1-F are constant for a given k, we can precompute
        cost = np.sum(alpha) - sharpness_layers[k]
        # Ensure non-negative
        cost = np.maximum(cost, 0)
        data_costs[:, :, k] = cost
        
    # Smoothness cost S(l_X, l_X') depends on label difference
    # S = 0 if equal, 0.5 + log(|diff|) if unequal
    
    # Placeholder for actual Graph Cut solver
    # l = graph_cut_minimize(data_costs, smoothness_func, K)
    # For the purpose of this article, we assume l is obtained.
    # In practice, one might use `cv2.ximgproc.graphCut` or similar.
    l = np.zeros((H, W), dtype=int) # Placeholder
    return l

def focus_to_depth(focus_map, K, f0, D_ref):
    """
    Convert discrete focus map to continuous depth map.
    This requires camera calibration metadata mapping focus index to distance.
    """
    # The source states this conversion uses camera calibration parameters.
    # Exact mapping function is not provided in the text, 
    # but typically involves the thin lens equation inversion or lookup table.
    # We assume a function `get_depth_from_focus_index` exists.
    depth_map = np.zeros_like(focus_map, dtype=float)
    for k in range(K):
        # Assign depth corresponding to focus index k
        # This is a placeholder for the specific calibration curve
        depth_map[focus_map == k] = get_depth_from_focus_index(k, f0, D_ref)
    return depth_map

def masked_back_projection(focal_stack, focus_map, depth_map, f0, D_ref, K):
    """
    Reconstruct epipolar image using masked back-projection.
    """
    H, W = focal_stack[0].shape
    # Epipolar image dimensions: Spatial (W) x Angular (U)
    # U is determined by the range of angles. 
    # For simplicity, assume U is large enough or computed from max angle.
    # Let's assume U = W for square epipolar image representation, 
    # though physically U might differ.
    U = W 
    LF = np.zeros((H, U))
    
    # Calculate angles for all pixels
    # theta = arctan((1/D_ref - 1/D) / (1/f0 - 1/D_ref))
    # tan_theta = u/x => u = x * tan_theta
    # We need to map pixel (x, y) in epipolar image to u coordinate.
    # In the source, x is spatial, u is angular.
    # The back-projection angle theta defines the slope u/x.
    
    # Precompute tan_theta for each pixel in the scene (which maps to x in epipolar)
    # Note: The depth map D is per pixel (x,y) in the original image.
    # The epipolar image LF(x, u) corresponds to a specific y-slice.
    # We reconstruct one horizontal line (epipolar image) at a time.
    
    # Get depth map for the specific y-slice (assuming we reconstruct for a fixed y)
    # Or reconstruct the whole 2D epipolar image for all y.
    # The source says "For each horizontal line... reconstruct the corresponding 2D epipolar image".
    # So we iterate over y.
    
    for y in range(H):
        # Extract depth and focus map for this row
        depth_row = depth_map[y, :]
        focus_row = focus_map[y, :]
        
        # Calculate tan_theta for each x
        # Avoid division by zero if D = D_ref (theta = 0)
        # tan_theta = (1/D_ref - 1/D) / (1/f0 - 1/D_ref)
        denom = 1.0/f0 - 1.0/depth_row
        numer = 1.0/D_ref - 1.0/depth_row
        
        # Handle D = D_ref case
        valid = np.abs(denom) > 1e-9
        tan_theta = np.zeros_like(depth_row)
        tan_theta[valid] = numer[valid] / denom[valid]
        
        # Sort pixels by depth (decreasing) for overwriting
        # Indices sorted by depth descending
        sorted_indices = np.argsort(-depth_row)
        
        # Process background first? 
        # Source: "We first back-project entirely the background."
        # Then "back-project only the in-focus parts... from second farthest to foreground".
        # Distinguishing background is tricky without semantic info. 
        # Usually, background is the farthest depth or a specific plane.
        # We assume the farthest depth pixels are background or handled by full projection.
        
        # Let's assume we process all pixels in decreasing depth order.
        # For each pixel x in sorted order:
        for idx in sorted_indices:
            x = idx
            D_val = depth_row[x]
            tan_t = tan_theta[x]
            
            # Determine which image k is in focus for this pixel
            k = focus_row[x]
            
            # Calculate angular coordinate u in the epipolar image
            # u = x * tan_theta
            u = x * tan_t
            
            # Map u to integer index in LF
            # Assuming LF is centered or scaled appropriately.
            # Let's assume u is in range [-U/2, U/2] or similar.
            # For simplicity, assume u is directly indexable if we scale LF.
            # Or we map u to a bin.
            # Let's assume LF has sufficient resolution and we just write to u.
            # If u is out of bounds, skip.
            
            u_idx = int(round(u))
            if 0 <= u_idx < U:
                # Get pixel value from image I_k
                # I_k is the k-th image (0-indexed)
                pixel_val = focal_stack[k][y, x]
                
                # Overwrite LF[y, u_idx]
                LF[y, u_idx] = pixel_val
                
    return LF
```

## Operating characteristics
*   **Time Complexity:** Dominated by the graph cut optimization for the focus map, which is generally $O(N \log N)$ or similar depending on the solver, where $N$ is the number of pixels. The back-projection is $O(N \cdot K)$ if naive, but can be optimized.
*   **Memory Complexity:** Requires storing the focal stack ($K \cdot W \cdot H$), sharpness layers ($K \cdot W \cdot H$), focus map ($W \cdot H$), depth map ($W \cdot H$), and the output epipolar image ($W \cdot U$).
*   **Performance:** The method achieves high angular resolution and perspective shifts despite using a single fixed camera and a small number of projections (focal stack images) [S1, unofficial]. It is more robust to variations in capturing conditions (number of images, depth ranges) than spectral averaging methods [S1, unofficial].
*   **Strengths:** Preserves the full native spatial resolution of the conventional 2D camera, unlike plenoptic cameras which trade spatial for angular resolution [S1, unofficial]. It handles limited projections effectively by exploiting the linear structure of epipolar images and using occlusion-aware overwriting [S1, unofficial].
*   **Limitations:** The depth map precision is limited by the number of focal planes $K$ [S1, unofficial]. It requires accurate camera calibration and focus map estimation, which can fail in texture-poor regions without the graph cut diffusion step [S1, unofficial]. It does not reconstruct the full 4D light field, only the subset $LF(x, y, u, 0)$ sufficient for refocusing and perspective shifts [S1, unofficial].

## Background
The method was proposed by [S1, unofficial] as a technique to reconstruct 4D light fields from a focal stack captured by a conventional high-resolution camera. It addresses the limitations of plenoptic/microlens cameras, which sacrifice spatial resolution for angular resolution, and spectral averaging methods, which require deconvolution and are sensitive to capturing conditions. The approach is rooted in the tomographic principle that a photograph is a 2D projection of the 4D light field along a direction determined by the focus distance [S1, unofficial]. It differs from classical medical tomography where projection angles are known; here, angles are estimated from a depth map derived from focus cues. The method enables applications such as digital refocusing, extended depth of field, and perspective shifts [S1, unofficial].

## Sources
[S1] Partial light field tomographic reconstruction from a fixed-camera focal stack, https://arxiv.org/abs/1503.01903v1, fetched 2026-10-26 (unofficial)
