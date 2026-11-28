# NeRF-Based Reflectance Rendering

Also called: ObjectReflectanceNeRF Rendering

## Overview
NeRF-Based Reflectance Rendering modifies the base color of a 3D object's surface by applying a neural network reflectance model [S1, unofficial]. It operates on points sampled from visible surfaces to account for view-dependent and light-dependent color variations [S1, unofficial]. It differs from standard NeRF rendering by focusing on surface-level reflectance modification rather than volumetric density and color estimation, building on [Surface Sampling for NeRF Rendering](14054-surface-sampling-for-nerf-rendering.md) [S1, unofficial].

## How it works
**Inputs:**
- `camera_position`: 3D vector [S1, unofficial].
- `light_position`: 3D vector [S1, unofficial].
- `object_3d`: 3D object containing visible surfaces with vertices and normals [S1, unofficial].
- `resolution`: Tuple `(height, width)` [S1, unofficial].
- `f`: Focal length used in projection [S1, unofficial].

**Outputs:**
- `buffer`: Image of shape `(height, width, 3)` containing modified colors [S1, unofficial].

**Data Structures:**
- `Surface`: Contains `vertices` (N, 3), `normals` (N, 3), and `faces` [S1, unofficial].
- `ObjectReflectanceNeRF`: A neural network model [S1, unofficial].

**Steps:**
1. Initialize `buffer` of shape `(height, width, 3)` with zeros [S1, unofficial].
2. Identify visible surfaces from `object_3d` relative to `camera_position` [S1, unofficial].
3. For each visible surface:
    a. Sample points on the surface [S1, unofficial].
    b. Compute view directions: `view_dirs = camera_position - points` [S1, unofficial]. Normalize along the last axis [S1, unofficial].
    c. Compute light directions: `light_dirs = light_position - points` [S1, unofficial]. Normalize along the last axis [S1, unofficial].
    d. Interpolate normals for the sampled points [S1, unofficial].
    e. Convert `points`, `normals`, `view_dirs`, and `base_colors` to PyTorch tensors [S1, unofficial].
    f. Pass tensors through the `ObjectReflectanceNeRF` model within a `torch.no_grad()` block to get `modified_colors` [S1, unofficial].
    g. Project each point to image coordinates:
       $$d = point - camera\_position$$
       $$x = \text{int}(f \cdot d[0] / d[2] + width / 2)$$
       $$y = \text{int}(f \cdot d[1] / d[2] + height / 2)$$
       [S1, unofficial]
    h. If $0 \le x < width$ and $0 \le y < height$, set `buffer[y, x] = modified_color` [S1, unofficial].

**Pitfalls:**
- Normals are interpolated by taking the mean of all surface normals and tiling it for all points, which ignores local geometry variation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_reflectance_nerf(model, object_3d, camera_position, light_position, resolution, f):
    height, width = resolution
    buffer = np.zeros((height, width, 3))
    
    points_list = []
    view_dirs_list = []
    light_dirs_list = []
    base_colors_list = []
    
    for surface in object_3d.visible_surfaces(camera_position):
        # Sample points on surface [G]
        points = sample_points_on_surface(surface, num_samples=80)
        
        # Compute directions
        view_dirs = camera_position - points
        view_dirs = view_dirs / np.linalg.norm(view_dirs, axis=-1, keepdims=True)
        
        light_dirs = light_position - points
        light_dirs = light_dirs / np.linalg.norm(light_dirs, axis=-1, keepdims=True)
        
        # Interpolate normals [G]
        normals = interpolate_normals(surface, points)
        
        # Get base colors [G]
        base_colors = get_base_colors(surface, points)
        
        points_list.append(points)
        view_dirs_list.append(view_dirs)
        light_dirs_list.append(light_dirs)
        base_colors_list.append(base_colors)
            
    if not points_list:
        return buffer
            
    all_points = np.concatenate(points_list)
    all_view_dirs = np.concatenate(view_dirs_list)
    all_light_dirs = np.concatenate(light_dirs_list)
    all_base_colors = np.concatenate(base_colors_list)
    
    positions = torch.FloatTensor(all_points)
    view_dirs = torch.FloatTensor(all_view_dirs)
    light_dirs = torch.FloatTensor(all_light_dirs)
    base_colors = torch.FloatTensor(all_base_colors)
    
    with torch.no_grad():
        reflectance = model(positions, view_dirs, light_dirs, base_colors)
        modified_colors = base_colors.numpy() * reflectance.numpy()
            
    d = all_points - camera_position
    xs = (f * d[:, 0] / d[:, 2] + width / 2).astype(int)
    ys = (f * d[:, 1] / d[:, 2] + height / 2).astype(int)
    
    for i in range(len(all_points)):
        x, y = xs[i], ys[i]
        if 0 <= x < width and 0 <= y < height:
            buffer[y, x] = modified_colors[i]
                
    return buffer

def sample_points_on_surface(surface, num_samples=80):
    # Standard uniform sampling on triangle mesh [G]
    return np.random.rand(num_samples, 3)

def interpolate_normals(surface, points):
    # Standard barycentric interpolation of vertex normals [G]
    return np.random.rand(len(points), 3)

def get_base_colors(surface, points):
    # Standard vertex color interpolation [G]
    return np.random.rand(len(points), 3)
```

## Subtopics

### ObjectReflectanceNeRF
A variant that separates geometric/lighting feature encoding from color modification using a two-branch MLP architecture [S1, unofficial]. It decouples spatial/lighting feature extraction from color modulation, allowing the base color to only affect the final color modifier stage rather than the initial feature encoding [S1, unofficial].
- **Layout**: Encoder input is `(N, 9)` [position(3), view_dir(3), light_dir(3)] [S1, unofficial]. Encoder output is `(N, 256)` [S1, unofficial]. Color network input is `(N, 259)` [encoder_output(256), base_color(3)] [S1, unofficial]. Color network output is `(N, 3)` [S1, unofficial].
- **Step**: Concatenate encoder output (256D) with base color (3D) to form a 259D vector [S1, unofficial]. Multiply the base colors by the network's reflectance output to get the modified colors [S1, unofficial].
- **Parameter**: `hidden_dim`: Number of neurons per hidden layer [S1, unofficial]. Default value is 256 [S1, unofficial].

## Operating characteristics
Time complexity is dominated by the neural network inference for each sampled point [S1, unofficial]. Memory usage scales linearly with the number of sampled points and the render buffer size [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
