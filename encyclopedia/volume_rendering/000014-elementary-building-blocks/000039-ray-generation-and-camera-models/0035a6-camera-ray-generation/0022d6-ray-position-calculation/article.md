# Ray Position Calculation

Also called: Query Point Sampling

## Overview
Ray Position Calculation computes the 3D query points along a ray within specified near and far depth bounds, optionally adding random noise for jittering [S1, unofficial]. It calculates the 3D position along a ray given a starting camera position, ray direction, and distance [S2, unofficial]. This method transforms 1D depth samples into 3D world coordinates, serving as the bridge between ray generation and volume sampling [S1, unofficial]. Unlike [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) which focuses on direction derivation, or [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md) which handles data transfer, this method specifically handles the spatial interpolation along the ray axis [S1, unofficial].

## How it works
**Inputs:**
- `ray_origins`: Tensor of shape `(width, height, 3)` containing the 3D origin for each pixel [S1, unofficial].
- `ray_directions`: Tensor of shape `(width, height, 3)` containing the normalized direction vector for each pixel [S1, unofficial].
- `near_thresh`: Float, the nearest depth coordinate of interest [S1, unofficial].
- `far_thresh`: Float, the farthest depth coordinate of interest [S1, unofficial].
- `num_samples`: Integer, the number of samples to draw along each ray [S1, unofficial].
- `randomize`: Boolean, default `True`. If `True`, adds random noise to depth values for jittering [S1, unofficial].

**Outputs:**
- `query_points`: Tensor of shape `(width, height, num_samples, 3)` containing the 3D coordinates of the sampling points [S1, unofficial].

**Data Structures:**
- `depth_values`: 1D Tensor of shape `(num_samples)` containing the sampled distances along the ray [S1, unofficial].

**Steps:**
1. Generate a linearly spaced array of depth values between `near_thresh` and `far_thresh` with `num_samples` points [S1, unofficial].
2. If `randomize` is `True`, add uniform random noise to `depth_values` to jitter the samples [S1, unofficial].
3. Compute the query points using the formula:
   $$ \text{query\_points} = \text{ray\_origins} + \text{ray\_directions} \times \text{depth\_values} $$
   where broadcasting expands `depth_values` to match the spatial dimensions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def calculate_ray_positions(ray_origins, ray_directions, near_thresh, far_thresh, num_samples, randomize=True):
    """
    Computes 3D query points along rays within near_thresh and far_thresh bounds.
    
    Args:
        ray_origins (torch.Tensor): Shape (width, height, 3).
        ray_directions (torch.Tensor): Shape (width, height, 3).
        near_thresh (float): Nearest depth coordinate.
        far_thresh (float): Farthest depth coordinate.
        num_samples (int): Number of samples per ray.
        randomize (bool): Whether to add jitter noise.
        
    Returns:
        torch.Tensor: Shape (width, height, num_samples, 3).
    """
    # Step 1: Create depth values
    depth_values = torch.linspace(near_thresh, far_thresh, num_samples)
    
    # Step 2: Add random noise if requested
    if randomize:
        # Add uniform noise in range [0, 1/num_samples) to jitter samples
        noise = torch.rand_like(depth_values) / num_samples
        depth_values = depth_values + noise
        
    # Step 3: Compute query points
    # Expand dimensions for broadcasting:
    # ray_origins: (W, H, 1, 3)
    # ray_directions: (W, H, 1, 3)
    # depth_values: (1, 1, N, 1)
    
    ray_origins_expanded = ray_origins.unsqueeze(2)  # (W, H, 1, 3)
    ray_directions_expanded = ray_directions.unsqueeze(2)  # (W, H, 1, 3)
    depth_values_expanded = depth_values.unsqueeze(0).unsqueeze(0).unsqueeze(3)  # (1, 1, N, 1)
    
    query_points = ray_origins_expanded + ray_directions_expanded * depth_values_expanded
    
    return query_points
```

**Edge Cases and Pitfalls:**
- If `near_thresh` >= `far_thresh`, the depth range is invalid [S1, unofficial].
- If `num_samples` is 0, no points are generated [S1, unofficial].
- Precision issues may arise if `near_thresh` and `far_thresh` differ significantly in magnitude, potentially causing floating-point errors in the multiplication [S1, unofficial].
- The ray directions must be normalized for accurate distance interpretation, though the method itself does not enforce normalization [S1, unofficial].

## Subtopics

### [Ray Structure](4026-ray-structure.md)
Defines the data container for ray origins and directions. Use when organizing ray data for batch processing.

### [Ray Origin and Direction Setup](1793-ray-origin-and-direction-setup.md)
Generates the initial ray origins and directions. Use when setting up the camera model before sampling.

### [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md)
Generates parallel rays. Use when the camera has no perspective distortion.

### [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md)
Handles data streaming for ray computation. Use when optimizing memory bandwidth for large scenes.

### [Object Space Ray Setup](1642-object-space-ray-setup.md)
Sets up rays in object space. Use when working with local coordinate systems.

### [Custom Frustum Ray Generation via Vertex Z-Index](1753-custom-frustum-ray-generation-via-vertex-z-index.md)
Generates rays based on vertex Z-index. Use for custom frustum culling.

### [Ray Origin Offset Noise](2118-ray-origin-offset-noise.md)
Adds noise to ray origins. Use when anti-aliasing requires origin jittering.

### [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md)
Uses camera matrices for ray generation. Use when integrating with standard graphics pipelines.

### [Basic GPU Ray Generation via Fragment Shader](2208-basic-gpu-ray-generation-via-fragment-shader.md)
Generates rays in fragment shaders. Use for real-time rendering.

### [Interactive Camera Ray Generation](3931-interactive-camera-ray-generation.md)
Optimizes ray generation for interactive frame rates. Use when latency is critical.

## Operating characteristics
- **Time Complexity:** $O(W \times H \times N)$, where $W$ is width, $H$ is height, and $N$ is `num_samples` [S1, unofficial].
- **Memory Complexity:** $O(W \times H \times N \times 3)$ for storing `query_points` [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with respect to image resolution and number of samples [S1, unofficial].
- **Strengths:** Simple, efficient, and easily parallelizable on GPUs [S1, unofficial].
- **Limitations:** Does not handle occlusion or adaptive sampling [S1, unofficial].
- **When to use:** When uniform sampling along rays is sufficient for volume rendering [S1, unofficial].
- **When not to use:** When adaptive sampling or importance sampling is required [S1, unofficial].

## Background
The method originates from NeRF implementations for efficient 3D scene reconstruction [S1, unofficial]. It is commonly used in neural radiance fields (NeRF) and other volume rendering techniques to sample points along rays for density and color prediction [S1, unofficial]. The approach is standard in differentiable rendering pipelines [S1, unofficial].

## Sources
[S1] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S3] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
