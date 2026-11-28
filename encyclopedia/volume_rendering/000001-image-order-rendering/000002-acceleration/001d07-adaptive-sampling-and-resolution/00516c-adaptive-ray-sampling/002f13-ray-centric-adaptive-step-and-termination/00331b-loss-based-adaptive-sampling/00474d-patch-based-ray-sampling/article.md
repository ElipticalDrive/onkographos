# Patch-based Ray Sampling

Also called: Patch-based Rendering

## Overview
Patch-based ray sampling is a strategy where rays are grouped into spatially adjacent 2D patches to form local image grids, ensuring the adjacency required for local-structure losses like Discrete Wavelet Transform (DWT) loss [S1, unofficial]. It is distinguished from random ray sampling, which is used for global-structure losses (e.g., MSE), by enforcing local spatial coherence within each sampling batch [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: Image dimensions, patch size, total number of rays to sample per iteration.
- Output: A set of ray origins and directions corresponding to a contiguous rectangular region of the image plane.

**Data structures:**
- Ray coordinates: 2D integer indices $(u, v)$ representing pixel locations in the image grid.
- Patch buffer: A list of ray indices or coordinates with shape $(H_p, W_p)$, where $H_p$ and $W_p$ are patch height and width.

**Steps:**
1. Determine patch dimensions $H_p \times W_p$. Standard configuration uses $192 \times 192$ square patches [S1, unofficial].
2. Calculate the number of rays per patch: $N_{patch} = H_p \times W_p$. For $192 \times 192$, $N_{patch} = 36,864$ [S1, unofficial].
3. Select a random top-left corner $(u_0, v_0)$ within the valid image bounds such that the patch fits entirely within the image:
   $$ 0 \le u_0 \le W_{img} - W_p $$
   $$ 0 \le v_0 \le H_{img} - H_p $$
   where $W_{img}$ and $H_{img}$ are image width and height.
4. Generate all ray indices $(u, v)$ for $u \in [u_0, u_0 + W_p - 1]$ and $v \in [v_0, v_0 + H_p - 1]$.
5. Convert indices to ray origins and directions using the camera projection matrix.
6. Use these rays for loss computation (e.g., DWT loss) [S1, unofficial].

**Parameters:**
- Patch size: $192 \times 192$ [S1, unofficial].
- Sampling frequency: DWT loss calculated every 10 iterations for LLFF dataset and every 150 iterations for NeRF-Synthetic (NS) dataset [S1, unofficial].
- Training duration for DWT: Stopped altogether at 5,000 iterations to facilitate high-frequency learning [S1, unofficial].

**Edge cases and pitfalls:**
- Boundary conditions: The patch must fit within the image. Random selection of top-left corner ensures this if bounds are respected [S1, unofficial].
- Precision: No specific precision issues noted, but standard floating-point ray casting applies.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_patch_rays(image_width, image_height, patch_size=192):
    """
    Samples rays for a single patch-based batch.
    
    Args:
        image_width (int): Width of the image in pixels.
        image_height (int): Height of the image in pixels.
        patch_size (int): Side length of the square patch.
        
    Returns:
        list of tuples: List of (u, v) pixel coordinates in the patch.
    """
    # Step 1 & 2: Define patch dimensions
    h_p, w_p = patch_size, patch_size
    
    # Step 3: Select random top-left corner within valid bounds
    # Valid range for u0: [0, image_width - w_p]
    # Valid range for v0: [0, image_height - h_p]
    u0 = np.random.randint(0, image_width - w_p + 1)
    v0 = np.random.randint(0, image_height - h_p + 1)
    
    # Step 4: Generate all ray indices in the patch
    # Using meshgrid to create all combinations
    u_coords, v_coords = np.meshgrid(
        range(u0, u0 + w_p),
        range(v0, v0 + h_p)
    )
    
    # Flatten to list of (u, v) tuples
    rays = list(zip(u_coords.flatten(), v_coords.flatten()))
    
    return rays
```

## Subtopics

### Grid-based Patch Sampling
A variant where rays are divided into equal-size patches following rigid grid lines determined by a fixed grid size, then shuffled [S2]. It is chosen for simplicity in SR-based NeRF pipelines but suffers from a pitfall: cutting ray space strictly with grid boundaries causes variations between boundaries to never be seen by convolutional kernels during training [S2].

## Operating characteristics
- **Time complexity:** Sampling is $O(N_{patch})$ where $N_{patch}$ is the number of rays in the patch. The overhead of coordinate generation is negligible compared to ray casting.
- **Memory complexity:** $O(N_{patch})$ to store ray origins and directions. For $192 \times 192$, this is 36,864 rays [S1, unofficial].
- **Performance:** Evaluated on GeForce RTX 4070 Ti Super [S1, unofficial]. DWT loss calculation adds computational cost but is limited to 5,000 iterations [S1, unofficial].
- **Strengths:** Captures local structures required for DWT loss [S1, unofficial].
- **Limitations:** Grid-based variants (see Subtopics) may miss boundary variations [S2]. Random patch sampling (the primary method) avoids this but requires careful boundary handling [S1, unofficial].

## Background
Originated in DWTNeRF, which boosts few-shot Neural Radiance Fields via Discrete Wavelet Transform [S1, unofficial]. The method was introduced to support the DW loss, which requires adjacently sampled rays to form a 2D image grid [S1, unofficial]. It contrasts with random ray sampling used for MSE loss to capture global structures [S1, unofficial].

## Sources
[S1] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
[S2] FastSR-NeRF: Improving NeRF Efficiency on Consumer Devices with A Simple Super-Resolution Pipeline, https://doi.org/10.1109/wacv57701.2024.00593, fetched 2026-10-26
