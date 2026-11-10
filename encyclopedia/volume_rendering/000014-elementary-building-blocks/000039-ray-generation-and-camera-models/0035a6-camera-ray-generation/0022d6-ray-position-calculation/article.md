# Ray Position Calculation

Also called: Query Point Sampling

## Overview
Ray Position Calculation computes 3D query points along rays within specified near and far depth bounds, optionally adding random noise for jittering [S1, unofficial]. It calculates the 3D position along a ray given a starting camera position, ray direction, and distance [S2, unofficial]. This method transforms 1D depth samples into 3D world coordinates, serving as the bridge between ray generation and volume sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `ray_origins`: Tensor of shape `(width, height, 3)` containing the 3D origin for each pixel [S1, unofficial].
- `ray_directions`: Tensor of shape `(width, height, 3)` containing the direction vector for each pixel [S1, unofficial].
- `near_thresh`: Float, the nearest depth coordinate of interest [S1, unofficial].
- `far_thresh`: Float, the farthest depth coordinate of interest [S1, unofficial].
- `num_samples`: Integer, the number of samples to draw along each ray [S1, unofficial].
- `randomize`: Boolean, default `True`. If `True`, adds random noise to depth values [S1, unofficial].
- `query_points`: Tensor of shape `(width, height, num_samples, 3)` containing the 3D coordinates of the sampling points [S1, unofficial].

**Data Structures:**
- `depth_values`: 1D Tensor of shape `(num_samples)` containing the sampled distances along the ray [S1, unofficial].

**Numbered Steps:**
1. Generate a linearly spaced array of depth values between `near_thresh` and `far_thresh` with `num_samples` points [S1, unofficial].
2. If `randomize` is `True`, add uniform random noise in `[0, 1)` to `depth_values` to jitter the samples [G].
3. Compute the query points by scaling the ray directions by the depth values and adding the ray origins, using broadcasting to align dimensions [S1, unofficial].

**Formula:**
$$ \text{query\_points} = \text{ray\_origins}[\dots, \text{None}, :] + \text{ray\_directions}[\dots, \text{None}, :] \times \text{depth\_values}[\dots, :, \text{None}] $$
where `width` and `height` are the image dimensions, `num_samples` is `N`, and `3` denotes the x, y, z coordinates. Broadcasting expands `depth_values` from shape `(N,)` to `(1, 1, N, 1)` to match `(W, H, 1, 3)` for both origins and directions, yielding `(W, H, N, 3)` [S1, unofficial].

**Parameters:**
- `ray_origins`: `torch.Tensor`, shape `(width, height, 3)` [S1, unofficial].
- `ray_directions`: `torch.Tensor`, shape `(width, height, 3)` [S1, unofficial].
- `near_thresh`: `float`, nearest depth coordinate of interest [S1, unofficial].
- `far_thresh`: `float`, farthest depth coordinate of interest [S1, unofficial].
- `num_samples`: `int`, number of samples drawn along each ray [S1, unofficial].
- `randomize`: `bool`, default `True` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def calculate_ray_positions(ray_origins: torch.Tensor, ray_directions: torch.Tensor,
                            near_thresh: float, far_thresh: float,
                            num_samples: int, randomize: bool = True) -> torch.Tensor:
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
        depth_values = depth_values + torch.rand_like(depth_values)
        
    # Step 3: Compute query points using broadcasting
    # ray_origins: (W, H, 3) -> (W, H, 1, 3)
    # ray_directions: (W, H, 3) -> (W, H, 1, 3)
    # depth_values: (N,) -> (1, 1, N, 1)
    ray_origins_expanded = ray_origins.unsqueeze(2)
    ray_directions_expanded = ray_directions.unsqueeze(2)
    depth_values_expanded = depth_values.unsqueeze(0).unsqueeze(0).unsqueeze(3)
    
    query_points = ray_origins_expanded + ray_directions_expanded * depth_values_expanded
    
    return query_points
```

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
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** $O(W \times H \times N \times 3)$ for storing `query_points` [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The method originates from NeRF implementations for efficient 3D scene reconstruction [S1, unofficial]. It is commonly used in neural radiance fields (NeRF) and other volume rendering techniques to sample points along rays for density and color prediction [S1, unofficial].

## Sources
[S1] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
