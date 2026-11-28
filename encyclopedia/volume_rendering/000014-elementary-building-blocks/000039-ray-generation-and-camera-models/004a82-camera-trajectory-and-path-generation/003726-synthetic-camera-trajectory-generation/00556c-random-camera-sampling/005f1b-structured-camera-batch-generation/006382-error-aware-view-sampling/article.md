# Error-aware View Sampling

## Overview
Error-aware view sampling is a dynamic camera trajectory generation method that refines camera positions based on the current rendering error to ensure focused coverage of complex or poorly reconstructed regions [S1].

## How it works
The method operates in two phases: initial global coverage and iterative local refinement.

**Inputs:**
- Reference image (ground truth) [S1].
- Current rendered image [S1].
- Set of 3D points (e.g., point cloud or mesh vertices) [S1].
- Hyperparameters: $K$ (number of points for focal center), sampling radius for refinement [S1].

**Outputs:**
- New camera poses (position and orientation) [S1].

**Data Structures:**
- Reference and rendered images: 2D arrays of pixel values [S1].
- 3D points: List of $(x, y, z)$ coordinates [S1].
- Camera pose: Position $\mathbf{c}$ and look-at target $\mathbf{t}$ [G].

**Steps:**

1. **Initial Sampling (Global Coverage):**
   - Sample camera positions on a hulling sphere surrounding the object using farthest point sampling to ensure uniform initial distribution [S1].
   - Randomly perturb these sampled positions to break symmetry [S1].
   - Set the camera look-at target to the center of the object (centroid of the 3D points) [S1].
   - Repeat this process periodically during optimization to maintain global coverage [S1].

2. **Error-Aware Refinement (Local Focus):**
   - Downsample the reference image and the current rendered image [S1].
   - Compute the pixel-wise error between the downsampled images [S1].
   - Find the pixel coordinates $(u_{max}, v_{max})$ with the largest error value [S1].
   - Identify $K$ 3D points whose projection onto the image plane is closest to $(u_{max}, v_{max})$ [S1].
   - Calculate the mean 3D position $\mathbf{c}_{focus}$ of these $K$ points:
     $$ \mathbf{c}_{focus} = \frac{1}{K} \sum_{i=1}^{K} \mathbf{p}_i $$
     where $\mathbf{p}_i$ are the selected 3D points [S1].
   - Sample new camera positions on a sphere centered at $\mathbf{c}_{focus}$ with a relatively small radius (distance) [S1].
   - Set the new camera look-at target to $\mathbf{c}_{focus}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_aware_view_sampling(points, ref_img, render_img, K, radius, fps_count, num_local_views):
    # Phase 1: Initial Global Sampling (performed periodically)
    global_positions = farthest_point_sampling_on_sphere(fps_count)
    global_positions = global_positions + random_perturbation(global_positions)
    object_center = sum(points) / len(points)
    global_cameras = [Camera(pos, object_center) for pos in global_positions]

    # Phase 2: Error-Aware Refinement
    ref_down = ref_img[::4, ::4]
    render_down = render_img[::4, ::4]
    error_map = abs(ref_down - render_down)
    max_error_pixel = argmax(error_map)
    projected_points = project_points_to_image(points, global_cameras[0])
    distances = sqrt((projected_points[:, 0] - max_error_pixel[0])**2 + (projected_points[:, 1] - max_error_pixel[1])**2)
    k_indices = argsort(distances)[:K]
    k_points = [points[i] for i in k_indices]
    focal_center = sum(k_points) / len(k_points)
    local_positions = sample_on_sphere(focal_center, radius, num_local_views)
    local_cameras = [Camera(pos, focal_center) for pos in local_positions]

    return global_cameras, local_cameras

# Helper definitions
def farthest_point_sampling_on_sphere(n):
    # Standard iterative farthest-point sampling on a unit sphere [G]
    return [random_unit_vector() for _ in range(n)]

def random_perturbation(pts, scale=0.1):
    return pts + random_normal(pts.shape) * scale

def project_points_to_image(pts, cam):
    # Simplified projection to image plane [G]
    return pts[:, :2]

def sample_on_sphere(center, radius, n):
    return [center + random_unit_vector() * radius for _ in range(n)]

class Camera:
    def __init__(self, pos, target):
        self.pos = pos
        self.target = target
```

**Parameters:**
- $K$: Number of points to average for the focal center [S1].
- Radius: Distance from the focal center for sampling new cameras. Described as "relatively small" [S1].
- Downsampling factor: Not given in the sources.
- FPS count: Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision: Downsampling images may smooth out high-frequency errors, leading to suboptimal focal points. Not given in the sources.
- Boundaries: If the object has complex topology, the initial uniform sampling may be insufficient, necessitating the error-aware step [S1].
- Degenerate Input: If $K$ points are not available (e.g., too few points), the method fails. Not given in the sources.

## Subtopics

### Mesh-Guided Multi-View Sampling
This variant uses a target mesh and camera poses to sample views [S2]. At each training step, $V$ camera views are sampled [S2]. All rays hitting the mesh across these views are identified [S2]. A random subset of these rays is selected to provide uniform coverage around the object [S2]. The rendering error against ground-truth pixels at the ray intersections is minimized to drive optimization (e.g., HKTex optimization) [S2].

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Strengths: Quickly converges on complex shapes by focusing on high-error regions. Improves point positions in small holes during large shape deformation [S1].
- Limitations: Requires rendering at each step to compute error, which can be computationally expensive [G].
- When to use: When optimizing geometry or texture for complex shapes where uniform sampling is insufficient [S1].
- When not to use: When computational resources are limited and rendering is too slow [G].

## Background
The method was introduced in "Differentiable surface splatting for point-based geometry processing" [S1]. It addresses the challenge of quick convergence in differentiable rendering for point-based geometry editing. The authors found that simple uniform sampling on a hulling sphere was insufficient for shapes with complex topology, leading to the development of the error-aware refinement strategy [S1].

## Sources
[S1] Differentiable surface splatting for point-based geometry processing, https://doi.org/10.1145/3355089.3356513, fetched 2026-10-26
[S2] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
