# Masked Local-Global Ray Sampling

Also called: MLG Ray Sampling

## Overview
Masked Local-Global (MLG) ray sampling is a strategy for X-ray volume rendering that extracts contextual and geometric information by segmenting informative foreground regions and sampling rays on specific patches and outside regions [S1]. It combines patch-level sampling on foreground regions with pixel-level sampling on the rest of the foreground to capture local semantic information and global contextual representations while avoiding uninformative background rays [S1]. Unlike standard uniform sampling, it explicitly partitions the foreground mask into non-overlapping windows to prioritize local structure perception for the Lineformer backbone [S1].

## How it works
**Inputs:**
- Projection image $I \in \mathbb{R}^{H \times W}$ [S1].
- Threshold $T \in \mathbb{R}$ for binarization [S1].
- Window size $S$ (integer) for patch partitioning [S1].
- Number of windows for patch sampling $N_l$ [S1].
- Number of pixels for global sampling $N_g$ [S1].

**Outputs:**
- A set of rays $\mathcal{R}$ comprising local rays $\mathcal{R}_l$ and global rays $\mathcal{R}_g$ [S1].

**Data Structures:**
- Binary mask $M \in \{0, 1\}^{H \times W}$ [S1].
- Set of non-overlapping windows $\mathcal{W} \in \{0, 1\}^{\frac{HW}{S^2} \times S \times S}$ [S1].
- Set of foreground windows $\mathcal{W}_f \subseteq \mathcal{W}$ [S1].
- Selected local windows $\mathcal{W}_l \subseteq \mathcal{W}_f$ [S1].
- Set of selected global pixels $P_g$ [S1].

**Steps:**

1.  **Binarize Projection:** Create the binary mask $M$ by comparing the projection image $I$ against the threshold $T$:
    $$ M = \mathbb{1}_{I > T} $$
    where $\mathbb{1}_{condition}$ is 1 if the condition is true, 0 otherwise [S1].

2.  **Partition into Windows:** Partition the mask $M$ into a set $\mathcal{W}$ of non-overlapping windows of size $S \times S$. The total number of windows is $\frac{HW}{S^2}$ [S1].

3.  **Identify Foreground Windows:** Define $\mathcal{W}_f$ as the subset of windows $\mathbf{w} \in \mathcal{W}$ that are entirely contained in the foreground regions (i.e., all pixels in the window are 1 in $M$):
    $$ \mathcal{W}_f = \{ \mathbf{w} \in \mathcal{W} \mid \mathbf{w} = \mathbb{1}_{S \times S} \} $$
    where $\mathbb{1}_{S \times S}$ denotes an $S \times S$ matrix of ones [S1].

4.  **Patch-Level Sampling (Local):** Randomly select $N_l$ windows $\mathcal{W}_l = \{ \mathbf{w}_1, \dots, \mathbf{w}_{N_l} \}$ from $\mathcal{W}_f$ [S1]. Construct the set of local rays $\mathcal{R}_l$ by sampling rays for all pixels $p$ within the selected windows:
    $$ \mathcal{R}_l = \bigcup_{i=1}^{N_l} \bigcup_{p \in \mathbf{w}_i} \text{Ray}(p) $$
    where $\text{Ray}(p)$ maps pixel $p$ to its corresponding X-ray [S1].

5.  **Pixel-Level Sampling (Global):** Randomly select $N_g$ pixels $P_g$ from the foreground regions of $M$, excluding the pixels covered by the windows in $\mathcal{W}_l$, to avoid repeated sampling [S1]. Construct the set of global rays $\mathcal{R}_g$ by sampling rays for these pixels:
    $$ \mathcal{R}_g = \{ \text{Ray}(p) \mid p \in P_g \} $$

6.  **Combine:** The final ray batch is $\mathcal{R} = \mathcal{R}_l \cup \mathcal{R}_g$ [S1].

**Parameters:**
- $T$: Threshold for binarizing the projection image. Value not specified in sources [S1].
- $S$: Size of the non-overlapping windows. Value not specified in sources [S1].
- $N_l$: Number of windows selected for patch-level sampling. In implementation, 1024 rays are from patch-level sampling [S1].
- $N_g$: Number of pixels selected for pixel-level sampling. In implementation, 1024 rays are from pixel-level sampling [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def masked_local_global_sampling(I, T, S, N_l, N_g):
    """
    I: Projection image, shape (H, W)
    T: Threshold for binarization
    S: Window size (integer)
    N_l: Number of windows for local sampling
    N_g: Number of pixels for global sampling
    Returns: List of ray indices or coordinates
    """
    H, W = I.shape
    
    # Step 1: Binarize
    M = (I > T).astype(int)
    
    # Step 2: Partition into windows
    # Calculate number of windows
    num_windows_h = H // S
    num_windows_w = W // S
    total_windows = num_windows_h * num_windows_w
    
    # Create window indices
    window_indices = np.arange(total_windows)
    
    # Step 3: Identify foreground windows
    # A window is foreground if all pixels in it are 1
    # Reshape M to (num_windows_h, S, num_windows_w, S)
    M_reshaped = M[:num_windows_h*S, :num_windows_w*S].reshape(num_windows_h, S, num_windows_w, S)
    # Check if all pixels in each window are 1
    window_is_fg = np.all(M_reshaped == 1, axis=(1, 3))
    # Flatten to 1D boolean array
    window_is_fg_flat = window_is_fg.flatten()
    
    # Get indices of foreground windows
    fg_window_indices = np.where(window_is_fg_flat)[0]
    
    # Step 4: Patch-level sampling
    # Randomly select N_l windows from foreground windows
    if len(fg_window_indices) < N_l:
        # If not enough foreground windows, take all and pad or handle error
        # Based on source, we assume sufficient foreground exists
        selected_local_window_indices = fg_window_indices
    else:
        selected_local_window_indices = np.random.choice(fg_window_indices, N_l, replace=False)
    
    # Map window indices to pixel coordinates
    local_pixels = []
    for w_idx in selected_local_window_indices:
        # Convert flat index to 2D window index
        w_h, w_w = divmod(w_idx, num_windows_w)
        # Generate all pixel coordinates in this window
        for dy in range(S):
            for dx in range(S):
                py = w_h * S + dy
                px = w_w * S + dx
                local_pixels.append((py, px))
    
    # Step 5: Pixel-level sampling
    # Create a mask of pixels already sampled in local step
    local_pixel_set = set(local_pixels)
    
    # Find all foreground pixels
    fg_pixels = np.argwhere(M == 1)
    # Convert to set for fast lookup
    fg_pixel_set = set(map(tuple, fg_pixels))
    
    # Exclude local pixels
    available_global_pixels = list(fg_pixel_set - local_pixel_set)
    
    if len(available_global_pixels) < N_g:
        # If not enough pixels, take all
        selected_global_pixels = available_global_pixels
    else:
        selected_global_pixels = np.random.choice(available_global_pixels, N_g, replace=False)
    
    # Step 6: Combine
    # Ray(p) is assumed to be a function that returns ray data for pixel p
    # Here we return the pixel coordinates as a proxy for rays
    all_rays = local_pixels + selected_global_pixels
    
    return all_rays
```

**Edge Cases and Pitfalls:**
- If the number of foreground windows is less than $N_l$, the algorithm cannot select $N_l$ unique windows. The sources do not specify the fallback behavior [S1].
- If the number of available global pixels (foreground minus local) is less than $N_g$, the algorithm cannot select $N_g$ unique pixels. The sources do not specify the fallback behavior [S1].
- The threshold $T$ is critical for defining the foreground mask but its value is not specified [S1].
- The window size $S$ must divide $H$ and $W$ for the partitioning to be exact, or the sources do not specify how to handle remainders [S1].

## Operating characteristics
- **Time Complexity:** The binarization and partitioning steps are $O(HW)$. Identifying foreground windows is $O(\frac{HW}{S^2})$. Selecting windows and pixels is $O(N_l + N_g)$ assuming efficient data structures. The dominant cost is likely the ray generation if $\text{Ray}(p)$ is expensive, but the sampling logic itself is efficient [S1].
- **Memory Complexity:** Storing the mask $M$ is $O(HW)$. Storing the windows $\mathcal{W}$ is $O(\frac{HW}{S^2} \cdot S^2) = O(HW)$. The output ray set is $O(N_l \cdot S^2 + N_g)$ [S1].
- **Measured Performance:** In SAX-NeRF, the batch size is 2048 rays, with 1024 from patch-level and 1024 from pixel-level sampling [S1]. The method improves novel view synthesis and CT reconstruction performance compared to previous NeRF-based methods [S1].
- **Strengths:** Avoids uninformative background rays, captures local semantic information via patches, and captures global shape via pixel-level sampling [S1].
- **Limitations:** Requires a reliable foreground mask (dependent on $T$). Performance depends on the choice of $S$, $N_l$, and $N_g$ [S1].

## Background
MLG ray sampling was proposed in the SAX-NeRF paper for sparse-view X-ray 3D reconstruction [S1]. It is designed to work with the Lineformer backbone, which captures internal structures by modeling dependencies within line segments of X-rays [S1]. The method addresses the problem of learning from uninformative background rays which degrade model efficiency [S1].

## Sources
[S1] Structure-Aware Sparse-View X-Ray 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01062, fetched 2026-10-26
