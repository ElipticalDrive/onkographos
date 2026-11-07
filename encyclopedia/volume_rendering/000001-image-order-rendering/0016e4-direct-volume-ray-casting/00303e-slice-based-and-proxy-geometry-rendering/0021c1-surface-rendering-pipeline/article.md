# Surface Rendering Pipeline

Also called: Surface Rendering from Volume
## Overview
Surface rendering is a volume rendering mode that stops at the first voxel inside a specified threshold and shades it with a normal taken from the value gradient [S1, unofficial]. It generates no mesh or triangles, avoiding invented shapes between voxels [S1, unofficial]. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume field, a threshold value, camera parameters (origin, direction), and image dimensions [S1, S2, unofficial].
- Output: A 2D image where each pixel contains the color of the first surface hit [S1, S2, unofficial].

**Data Structures:**
- `RayBundle`: Contains `origins` (N, 3), `directions` (N, 3), `sample_points` (N, K, 3), and `sample_lengths` (N, K, 3) [S2, unofficial].
- `SurfaceModel`: Contains `implicit_fn`, `sampler`, and `renderer` modules initialized from configuration dictionaries [S2, unofficial].

**Steps:**
1. Initialize the `SurfaceModel` with configuration dictionaries for `implicit_fn`, `sampler`, and `renderer` [S2, unofficial].
2. For each camera, generate normalized image plane coordinates in the range $[-1, 1]$ for width $W$ and height $H$ [S2, unofficial].
3. Unproject the normalized coordinates to world space to determine ray origins and directions [S2, unofficial].
4. Evaluate the scalar field along each ray until the first sample point satisfies the threshold condition [S1, unofficial].
5. Compute the surface normal at the hit point using the gradient of the scalar field [S1, unofficial].
6. Shade the hit point, accumulate the color, and reshape the result to $(H, W, 3)$ [S2, unofficial].

**Formulas:**
- Ray definition: $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction [S1, S2, unofficial].
- Threshold condition: Stop at the first sample $i$ where $V(\mathbf{r}(t_i)) \ge \tau$ [S1, unofficial].
- Normal computation: $\mathbf{n} = \nabla V(\mathbf{r}(t_i)) / \|\nabla V(\mathbf{r}(t_i))\|$ [S1, unofficial].
- Shading: $\mathbf{c} = \text{Shading}(\mathbf{n}, \mathbf{l})$, where $\mathbf{l}$ is the light direction [S1, unofficial].

**Parameters:**
- Threshold $\tau$: Determines the surface isovalue [S1, unofficial].
- Image dimensions $(W, H)$: Width and height of the output image [S2, unofficial].

**Edge Cases and Pitfalls:**
- Terracing: Visible when slice spacing is large, as surface rendering does not integrate over it [S1, unofficial].
- No Hit: If no voxel exceeds the threshold, the pixel remains background color [G].
- Precision: Gradient estimation at threshold crossings can be sensitive to interpolation artifacts [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_surface(volume, threshold, cameras, image_size):
    H, W = image_size
    all_images = []
    for camera in cameras:
        # 1. Generate normalized image plane coordinates in [-1, 1]
        x = torch.linspace(-1, 1, steps=W)
        y = torch.linspace(-1, 1, steps=H)
        grid_y, grid_x = torch.meshgrid(y, x, indexing='ij')
        xy_grid = torch.stack((grid_x, grid_y), dim=-1).view(W * H, 2)
        xy_grid = -xy_grid  # Source convention [S2, unofficial]

        # 2. Unproject NDC points to world space for ray origins and directions
        ndc_pts = torch.cat([xy_grid, torch.ones(W * H, 1)], dim=-1)
        origins, directions = standard_camera_unprojection(ndc_pts, camera)  # [G] inverse projection

        # 3. Forward pass: evaluate implicit function, sample, and render
        # Assume fixed ray length and sample count for demonstration
        t = torch.linspace(0, 10.0, 100)  # [G] typical integration range
        points = origins[:, None, :] + directions[:, None, :] * t[None, :, None]
        scalar_values = volume.evaluate(points)  # [G] trilinear interpolation of volume data

        # 4. Stop at first voxel inside threshold
        hit_mask = scalar_values >= threshold
        hit_idx = torch.argmax(hit_mask.int(), dim=1)
        hit_points = points[torch.arange(len(origins)), hit_idx]

        # 5. Shade with normal from value gradient
        normals = volume.compute_gradient(hit_points)
        normals = normals / torch.norm(normals, dim=-1, keepdim=True)
        colors = shade_phong(normals, light_dir=None)  # [G] standard illumination model

        # 6. Reshape and convert to numpy array
        image = colors.view(H, W, 3).detach().cpu().numpy()
        all_images.append(image)
    return all_images
```
## Subtopics

### [Real Time Cutting Operations](3494-real-time-cutting-operations.md)
Real Time Cutting Operations is a technique for clipping volume surfaces using cutting planes by maintaining near and far depth buffers per pixel, avoiding full re-rendering when planes change. It differs from static cutting methods by maintaining the full volume dataset and using a halfspace check to dynamically filter voxels per frame, whereas static methods pre-filter and discard cut voxels.

## Operating characteristics
- Performance: Measured at 390x844 on a synthetic 700 HU sphere, surface rendering exposed terracing artifacts due to 3 mm slice spacing [S1, unofficial].
- Strengths: No mesh generation overhead; direct shading from volume data [S1, unofficial].
- Limitations: Terracing artifacts at low resolution or large slice spacing [S1, unofficial].
## Background
Surface rendering is a mode in volume visualization that contrasts with volume rendering by stopping at the first threshold crossing [S1, unofficial].
## Sources
[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
