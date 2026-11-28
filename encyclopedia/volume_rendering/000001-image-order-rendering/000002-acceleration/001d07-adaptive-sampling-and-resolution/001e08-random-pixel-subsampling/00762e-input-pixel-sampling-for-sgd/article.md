# Input Pixel Sampling for SGD

Also called: Gaussian Pyramid Pixel Sampling, Multi-level Pixel Sampling

## Overview
This method facilitates convergence in Sparse Voxel Octree (SVO) optimization by simultaneously optimizing the scene at multiple levels of detail. It addresses the slow convergence of standard Stochastic Gradient Descent (SGD) in early reconstruction stages by combining Gaussian pyramid subsampling with importance sampling based on photo consistency error. Unlike [Random Pixel Subsampling](1668-random-pixel-subsampling.md), which selects pixels uniformly from a single resolution, this method leverages coarser pixels to optimize higher-level SVO nodes and finer pixels for detailed structures, while prioritizing pixels with high reconstruction error.

## How it works
**Inputs:** Input images $I_i$, ground truth images $I_{gt,i}$, current SVO structure, learning parameters.
**Outputs:** Updated SVO parameters via SGD.

**Data Structures:**
*   **Gaussian Pyramid:** A set of downsampled images for each input view. Level $l=0$ is the original resolution; level $L$ is the coarsest.
*   **Loss Cache $C$:** Stores prefix sum offsets of sampling weights $w_{i,c}(p_i)$ for fast random selection. It caches running error averages. To save memory, it only stores coarse error data at higher mipmap levels.

**Algorithm Steps:**

1.  **Preprocessing:** Compute a complete Gaussian (mipmap) pyramid for each input image [S1, unofficial].
2.  **Iteration Loop:** For each training iteration:
    a.  **Select Mipmap Level:** Randomly select a mipmap level $l$ from all available levels. This ensures simultaneous optimization of the SVO at multiple levels of detail [S1, unofficial].
    b.  **Select Pixel:**
        i.  If level $l$ is selected, identify the set of fine pixels $P_{fine}$ that map to the coarse pixel $p_{coarse}$ at level $l$.
        ii. Retrieve the cached error weight $w_{i,c}(p_{coarse})$ for this coarse pixel.
        iii. If multiple fine pixels map to the same coarse pixel, uniformly select one fine pixel from $P_{fine}$ [S1, unofficial].
        iv. Use the loss cache $C$ to perform importance sampling based on weights $w_{i,c}(p_i)$ to select the specific pixel coordinate [S1, unofficial].
    c.  **Compute Loss:** Calculate the photo consistency error for the selected pixel.
    d.  **Update Cache:** Update the running average error for the selected pixel.
    e.  **Update SVO:** Perform an SGD step to update the SVO parameters using the gradient from the selected pixel [S1, unofficial].
3.  **Cache Update:** Every 5000 iterations, update the prefix sum offsets in the loss cache $C$ based on the current running error averages [S1, unofficial].

**Formulas:**

The sampling weight for pixel $p_i$ in image $i$ at cache level $c$ is defined as [S1, unofficial]:

$$
w_{i,c}(p_i) = \max_c (\|I_i(p_i, \theta) - I_{gt,i}(p_i, \theta)\|^2) + \epsilon
$$

Where:
*   $I_i(p_i, \theta)$ is the rendered color at pixel $p_i$ with current parameters $\theta$.
*   $I_{gt,i}(p_i, \theta)$ is the ground truth color.
*   $\| \cdot \|^2$ denotes the squared Euclidean distance across color channels.
*   $\epsilon = 0.05$ is a small constant added to ensure low-error pixels are sampled infrequently, preventing oscillating errors [S1, unofficial].

**Parameters:**
*   $\epsilon = 0.05$ [S1, unofficial].
*   Cache update frequency: Every 5000 iterations [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The cache stores prefix sums. Ensure floating-point precision is sufficient to avoid errors in cumulative weight calculations over many pixels.
*   **Boundary:** When mapping fine pixels to coarse pixels, ensure the mapping is consistent (e.g., center-based or area-based) to avoid ambiguity.
*   **Degenerate Input:** If an image has no valid pixels (e.g., all black/white), the error might be uniform. The $\epsilon$ term helps maintain sampling diversity.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class InputPixelSampler:
    def __init__(self, images, epsilon=0.05, cache_update_freq=5000):
        self.images = images
        self.epsilon = epsilon
        self.cache_update_freq = cache_update_freq
        self.iteration = 0
        
        # Compute Gaussian pyramids for all images
        self.pyramids = [self._compute_gaussian_pyramid(img) for img in images]
        
        # Initialize loss cache with zeros
        # Cache stores running error averages. 
        # Structure: list of arrays, one per image, indexed by mipmap level.
        # Each array contains the current running average error for each pixel in that level.
        self.error_averages = [
            [np.zeros_like(level) for level in pyramid] 
            for pyramid in self.pyramids
        ]
        
        # Cache stores prefix sums of weights for fast sampling
        self.cache_offsets = [
            [np.zeros(len(level) + 1) for level in pyramid] 
            for pyramid in self.pyramids
        ]
        
        # Update cache initially
        self._update_cache()

    def _compute_gaussian_pyramid(self, image):
        """Compute Gaussian pyramid levels."""
        levels = [image]
        current = image
        while current.shape[0] > 1 and current.shape[1] > 1:
            # Simple average pooling for Gaussian blur/downsample
            current = self._downsample(current)
            levels.append(current)
        return levels

    def _downsample(self, image):
        """Downsample image by factor of 2."""
        h, w = image.shape[:2]
        return image[0:h:2, 0:w:2]

    def _compute_weight(self, rendered, ground_truth, epsilon):
        """Compute sampling weight w = max_channel_error^2 + epsilon."""
        error_sq = np.square(rendered - ground_truth)
        max_channel_error = np.max(error_sq, axis=-1)
        return max_channel_error + epsilon

    def _update_cache(self):
        """Update prefix sum offsets in the loss cache."""
        for i, pyramid in enumerate(self.pyramids):
            for l, level in enumerate(pyramid):
                # Use current error averages to compute weights
                weights = self._compute_weight(
                    self.error_averages[i][l], 
                    np.zeros_like(self.error_averages[i][l]), # Placeholder: actual GT needed
                    self.epsilon
                )
                # Compute prefix sums
                self.cache_offsets[i][l] = np.cumsum(weights)

    def sample_pixel(self, image_idx):
        """Sample a pixel from image_idx using importance sampling and mipmap selection."""
        pyramid = self.pyramids[image_idx]
        
        # 1. Randomly select a mipmap level
        level_idx = np.random.randint(len(pyramid))
        level = pyramid[level_idx]
        
        # 2. Get weights from cache for this level
        # Note: In practice, weights are derived from error_averages
        # For this pseudocode, we assume cache_offsets is precomputed and valid
        offsets = self.cache_offsets[image_idx][level_idx]
        
        # 3. Importance sampling using prefix sums
        total_weight = offsets[-1]
        r = np.random.uniform(0, total_weight)
        
        # Find index in prefix sum array
        pixel_idx = np.searchsorted(offsets, r, side='right') - 1
        
        # 4. Handle coarse-to-fine mapping
        # If level_idx > 0, multiple fine pixels map to this coarse pixel.
        # We need to select one fine pixel uniformly.
        if level_idx > 0:
            # Get the corresponding level in the original image (level 0)
            # This requires tracking the mapping or re-sampling from the finest level
            # based on the coarse pixel's area.
            # Simplified: Assume we have a function to get fine pixels for a coarse pixel
            fine_pixels = self._get_fine_pixels_for_coarse(image_idx, level_idx, pixel_idx)
            selected_fine_pixel = np.random.choice(fine_pixels)
            return selected_fine_pixel
        else:
            # Level 0: pixel_idx directly corresponds to a pixel in the original image
            h, w = self.images[image_idx].shape[:2]
            y, x = np.unravel_index(pixel_idx, level.shape[:2])
            # Map back to original resolution if necessary, or return coarse coords
            # For SGD, we usually need the exact ray origin, so fine coords are preferred
            # If level 0 is selected, we return the fine pixel corresponding to the coarse index
            # Assuming level 0 is the finest, pixel_idx is the index in the flattened level 0
            return (y, x)

    def _get_fine_pixels_for_coarse(self, image_idx, coarse_level, coarse_idx):
        """Return list of fine pixel coordinates that map to a coarse pixel."""
        # This is a placeholder for the actual mapping logic
        # It should return coordinates in the finest resolution (level 0)
        pass

    def update_error(self, image_idx, pixel_coords, rendered_color, gt_color):
        """Update the running error average for a pixel."""
        # pixel_coords is (y, x) in the finest resolution
        # We need to map this to the appropriate mipmap level for caching
        # For simplicity, assume we update the finest level cache
        h, w = self.images[image_idx].shape[:2]
        y, x = pixel_coords
        error = np.square(rendered_color - gt_color)
        max_err = np.max(error)
        
        # Update running average (exponential moving average or simple count-based)
        # Using simple count-based for clarity
        # self.error_averages[image_idx][0][y, x] = (self.error_averages[image_idx][0][y, x] + max_err) / 2
        # More robust:
        count = 1 # Placeholder for actual count tracking
        self.error_averages[image_idx][0][y, x] = max_err # Direct update for simplicity
        
        self.iteration += 1
        if self.iteration % self.cache_update_freq == 0:
            self._update_cache()
```

## Subtopics

### Loss-based Importance Sampling
A sampling technique that prefers pixels with high photo consistency error to accelerate convergence during early reconstruction stages. It uses the per-pixel maxima of color channel errors as weights, adding a small constant $\epsilon = 0.05$ to prevent oscillating errors. The loss cache stores prefix sum offsets of these weights, updating every 5000 iterations. It only stores coarse error cache data at higher mipmap pyramid levels for efficiency. If a low mipmap level is selected where multiple fine pixels fall into the same coarse pixel, the sampler uniformly selects between all fine pixels [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ after cache update. Cache update is $O(N)$ where $N$ is the number of pixels in the pyramid. Cache updates occur every 5000 iterations, amortizing the cost.
*   **Memory Complexity:** $O(N \log N)$ to store the Gaussian pyramid and the loss cache. The cache only stores coarse error data, reducing memory footprint compared to storing fine-grained errors for all levels.
*   **Performance:** Converges faster than uniform sampling in early reconstruction stages. The use of coarser pixels helps optimize higher-level SVO nodes, while finer pixels refine details.
*   **Strengths:** Accelerates convergence, handles multi-scale optimization naturally, reduces memory usage by caching coarse errors.
*   **Limitations:** Requires precomputing Gaussian pyramids. Cache updates are infrequent, which may lead to stale importance weights if the scene changes rapidly.
*   **When to use:** When training explicit radiance fields (ERF) or SVO-based representations where convergence speed is critical and memory is a constraint.
*   **When not to use:** If the scene is static and convergence is not an issue, or if memory is extremely limited and pyramids cannot be stored.

## Background
This method is part of the Explicit Radiance Field (ERF) reconstruction pipeline, introduced by [S1, unofficial]. It builds on the idea of multi-scale optimization and importance sampling to improve the efficiency of SGD in optimizing sparse voxel structures. The approach is designed to handle the Nyquist sampling theorem constraints for leaf node refinement.

## Sources
[S1] ERF: Explicit Radiance Field Reconstruction From Scratch, https://arxiv.org/abs/2203.00051v1, fetched 2026-10-26 (unofficial)
