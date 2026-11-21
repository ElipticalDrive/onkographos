# Local Sampling
Also called: Surface-aware Sampling
## Overview
Local Sampling is a ray sampling strategy that concentrates evaluation points around a known surface representation (e.g., a depth texture) to minimize empty-space evaluations in volume rendering [S1]. It distinguishes itself from sibling methods like [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by leveraging geometric priors to achieve high quality with significantly fewer samples, though it relies on the availability of ground truth depth during training [S1].
## How it works
**Inputs and Outputs:**
- Input: Camera ray parameters, a trained neural radiance field (NeRF) or similar implicit representation, and a ground truth depth texture (or depth oracle network output) representing the scene surface [S1].
- Output: A set of sample positions along the ray and the corresponding volume rendering result (color and alpha) [S1].

**Data Structures:**
- Depth Texture: A 2D array storing the distance from the camera origin to the nearest surface intersection for each pixel [S1].
- Ray: Defined by origin $o$ and direction $d$ [S1].

**Steps:**
1. **Depth Retrieval:** For a given pixel, retrieve the ground truth depth $z_{gt}$ from the depth texture [S1].
2. **Sample Placement:** Place a fraction of the total samples $N$ locally around $z_{gt}$. The remaining samples are distributed uniformly or according to another strategy in the empty space before and after the surface [S1].
3. **Volume Rendering:** Evaluate the neural network at the sample positions to obtain density $\sigma$ and color $c$, then perform standard volume integration [S1].

**Formulas:**
The depth $z_{gt}$ is used to define the local region. If $N$ is the total number of samples, and $N_{local}$ is the number of samples placed locally, the positions are clustered around $z_{gt}$ [S1]. The exact distribution within the local cluster is not specified beyond "around the surface" [S1].

**Parameters:**
- $N$: Total samples per ray. Tested values: [2, 4, 8, 16, 32, 64, 128] [S1].
- $N_{local}$: Number of samples placed locally. Can be as low as 2 with nearly no decrease in quality [S1].

**Edge Cases and Pitfalls:**
- **Depth Discontinuities:** A depth oracle network with a single depth output may smooth depth around discontinuities, causing local sampling to mix foreground and background features and distort the result [S1].
- **Inference Availability:** Ground truth depth is often unavailable during inference due to memory/computational constraints and the need for reprojection for novel views [S1].
- **NDC Sampling:** Sampling in Normalized Device Coordinates (NDC) reaches competitive results only at high sample counts (>64) and falls behind uniform sampling at lower counts because the network must reconstruct the nonlinear transformation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_sampling(ray_origin, ray_direction, depth_texture, nerf_model, total_samples=64):
    # Get ground truth depth for the current pixel
    # Note: In practice, this requires a depth oracle or pre-computed depth map
    depth_z = depth_texture[ray_origin.x, ray_origin.y]
    
    # Determine local sample range around depth_z
    # The exact width of the local region is not specified in the source
    # [G] Assuming a small epsilon range around depth_z for demonstration
    epsilon = 0.1 
    local_start = max(0, depth_z - epsilon)
    local_end = depth_z + epsilon
    
    # Place N_local samples locally
    n_local = 2
    local_samples = linspace(local_start, local_end, n_local)
    
    # Place remaining samples uniformly in the rest of the ray
    n_uniform = total_samples - n_local
    # [G] Assuming min_depth and max_depth are defined for the scene
    uniform_samples = linspace(0, max_depth, n_uniform)
    
    # Combine samples
    all_samples = concatenate([uniform_samples, local_samples])
    
    # Evaluate NeRF and render
    colors, alphas = volume_render(ray_origin, ray_direction, all_samples, nerf_model)
    return colors, alphas
```
## Subtopics

### [Mesh-based Sampling](6990-mesh-based-sampling.md)
Uses mesh boundaries to localize surface evaluation, requiring only two samples per ray. It is efficient and applicable as a post-processing step to existing NeRF models.

## Operating characteristics
- **Time Complexity:** Reduced compared to uniform sampling due to fewer total samples needed for equivalent quality [S1].
- **Memory Complexity:** Requires storage for the depth texture or depth oracle network [S1].
- **Performance:** With local sampling, the number of samples can be reduced to two with nearly no decrease in quality [S1]. Log+warp adds about 1.3 dB in quality over uniform for local sampling [S1].
- **Strengths:** Significant gains in quality-speed tradeoff given an ideal sampling oracle [S1].
- **Limitations:** Ground truth depth is often not available during inference, making it a niche scenario without an oracle network [S1].
## Background
Originated in the context of NeRF optimization, specifically in the DONeRF paper, which explores using depth oracle networks to enable local sampling for real-time rendering of compact neural radiance fields [S1].
## Sources
[S1] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
