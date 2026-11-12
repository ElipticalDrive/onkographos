# Surface Rendering Pipeline

Also called: Surface Rendering (R_so, Surface Rendering from Volume

## Overview
Surface rendering is a volume rendering mode that stops at the first voxel inside a threshold and shades it with a normal taken from the value gradient [S1, unofficial]. It generates no mesh or triangles, avoiding invented shapes between voxels [S1, unofficial]. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (scalar field), a threshold value, a camera configuration (position, orientation, image size), and a transfer function (mapping scalar values to color/opacity).
*   **Output:** A 2D image (color buffer).

**Data Structures:**
*   **Volume Data:** Stored as a 3D texture on the GPU [S2, unofficial].
*   **Camera:** Contains projection matrices and view parameters.
*   **Ray Bundle:** A structure containing origins, directions, and sample points for each pixel [S2, unofficial].

**Method Steps:**
1.  **Initialization:** Initialize the rendering model with an implicit function (representing the volume), a sampler (for ray marching or point sampling), and a renderer (for shading and compositing) [S2, unofficial].
2.  **Ray Generation:** For each camera, generate normalized image plane coordinates $xy\_grid$ based on the image size $W \times H$ [S2, unofficial].
    $$ x = \text{linspace}(-1, 1, W) $$
    $$ y = \text{linspace}(-1, 1, H) $$
    $$ xy\_grid = \text{meshgrid}(y, x) $$
    Convert these to ray origins and directions using the camera's unprojection matrix [S2, unofficial].
3.  **Ray Marching / Sampling:** Cast rays from the camera through the volume. For surface rendering, the ray is marched until the first voxel value exceeds the specified threshold [S1, unofficial].
4.  **Normal Estimation:** At the hit point (the first voxel inside the threshold), compute the surface normal using the gradient of the scalar field [S1, unofficial]. This can be done in scene space, object space, or view space [S3].
5.  **Shading:** Shade the hit point using the computed normal and a lighting model (e.g., Phong or Lambertian) [S2, unofficial].
6.  **Compositing:** Since surface rendering stops at the first hit, no alpha blending is required for depth compositing in the traditional sense, though the result is a single-color per pixel image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def get_normalized_image_plane_coordinates(image_size):
    W, H = image_size
    x = torch.linspace(-1, 1, steps=W)
    y = torch.linspace(-1, 1, steps=H)
    grid_y, grid_x = torch.meshgrid(y, x)
    xy_grid = torch.stack((grid_x, grid_y), dim=-1).view(W * H, 2)
    return -xy_grid

def get_rays_from_pixels(xy_grid, camera):
    N = xy_grid.shape[0]
    xy_grid_copy = xy_grid.clone().to(camera.device)
    ndc_pts_ip = torch.cat([xy_grid_copy, torch.ones([N, 1], device=camera.device)], dim=-1)
    # Unproject to world space to get ray origins and directions
    # Note: Implementation details depend on specific camera library (e.g., PyTorch3D)
    origins = camera.unproject_points(ndc_pts_ip, from_ndc=True)[:, :3]
    directions = camera.unproject_points(ndc_pts_ip, from_ndc=True)[:, 3:] # Simplified
    return origins, directions

def surface_render(volume_data, threshold, camera, image_size):
    # 1. Generate rays
    xy_grid = get_normalized_image_plane_coordinates(image_size)
    origins, directions = get_rays_from_pixels(xy_grid, camera)
    
    # 2. Ray Marching to find first hit
    # In a real implementation, this involves marching along directions
    # and sampling volume_data. Here we assume a simplified hit detection.
    hit_points = []
    normals = []
    
    for i in range(origins.shape[0]):
        # Simplified: Assume we find the first voxel index where value > threshold
        # This requires mapping ray to voxel coordinates
        voxel_idx = find_first_voxel_inside_threshold(origins[i], directions[i], volume_data, threshold)
        if voxel_idx is not None:
            hit_points.append(voxel_to_world(voxel_idx, volume_data))
            # Compute gradient for normal
            grad = compute_gradient(voxel_idx, volume_data)
            normals.append(grad / torch.norm(grad))
            
    # 3. Shade
    # If no hit, background color
    # If hit, use normal for lighting
    colors = []
    for i, hit in enumerate(hit_points):
        normal = normals[i]
        # Simple diffuse shading
        light_dir = torch.tensor([0.0, 0.0, 1.0], device=volume_data.device)
        diffuse = max(0, torch.dot(normal, light_dir))
        color = volume_data[hit] * diffuse
        colors.append(color)
        
    # Reshape to image
    image = torch.stack(colors).view(image_size[1], image_size[0], 3)
    return image
```

**Parameters:**
*   **Threshold:** The scalar value below which voxels are considered background and above which they are part of the surface. Typical values depend on the data (e.g., 390 HU for soft tissue in CT) [S1, unofficial].
*   **Slice Spacing:** The distance between voxels along the depth axis. This directly affects the visibility of terracing artifacts [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Terracing:** The discrete nature of voxel sampling can lead to visible "stair-step" artifacts, especially with large slice spacing [S1, unofficial].
*   **Threshold Selection:** Incorrect thresholding can lead to incomplete surfaces or inclusion of noise [S1, unofficial].
*   **Gradient Noise:** The gradient computation can be noisy if the volume data is not smoothed, leading to jagged normals [S1, unofficial].

## Subtopics

### [Real Time Cutting Operations](3494-real-time-cutting-operations.md)
Real Time Cutting Operations is a technique for clipping volume surfaces using cutting planes by maintaining near and far depth buffers per pixel, avoiding full re-rendering when planes change. It differs from static cutting methods by maintaining the full volume dataset and using a halfspace check to dynamically filter voxels per frame, whereas static methods pre-filter and discard cut voxels.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the average ray length until hit. Faster than full volume integration as it stops at the first hit [S1, unofficial].
*   **Memory Complexity:** Requires storage for the 3D volume texture and intermediate ray data [S2, unofficial].
*   **Performance:** Measured at 390x844 on a synthetic 700 HU sphere, surface rendering exposed 9.8% of the canvas at peak intensity 255, compared to volume rendering's 9.3% at 238 [S1, unofficial].
*   **Strengths:** No mesh generation overhead; avoids shape invention between voxels; fast for opaque surfaces [S1, unofficial].
*   **Limitations:** Terracing artifacts due to discrete sampling; sensitive to threshold selection; no internal structure visibility [S1, unofficial].

## Background
Surface rendering is a fundamental technique in volume visualization, often used in medical imaging to highlight specific structures [S3]. It is classified as a scene space, object space, or view space operation depending on the normal estimation technique [S3]. The method has been discussed in the context of imaging transforms for visualizing surfaces and volumes [S3].

## Sources
[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S3] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
