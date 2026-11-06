# Surface Rendering Pipeline

Also called: Surface Rendering from Volume
## Overview
Surface rendering is a volume rendering mode that stops at the first voxel inside a specified threshold and shades it with a normal taken from the value gradient [S1, unofficial]. It generates no mesh or triangles, avoiding invented shapes between voxels [S1, unofficial]. Unlike volume rendering which integrates over slice spacing, surface rendering exposes the slice spacing as terracing artifacts [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 3D scalar volume field, a threshold value, camera parameters (origin, direction), and image dimensions.
- Output: A 2D image where each pixel contains the color of the first surface hit.

**Data Structures:**
- `RayBundle`: Contains `origins` (N, 3), `directions` (N, 3), `sample_points` (N, K, 3), and `sample_lengths` (N, K, 3) [S2, unofficial].
- `SurfaceModel`: Contains `implicit_fn`, `sampler`, and `renderer` modules [S2, unofficial].

**Steps:**
1. Initialize the `SurfaceModel` with configuration dictionaries for `implicit_fn`, `sampler`, and `renderer` [S2, unofficial].
2. For each camera, generate normalized image plane coordinates using `get_normalized_image_plane_coordinates(image_size)` [S2, unofficial].
   - This creates a grid of $W \times H$ points in normalized device coordinates (NDC) ranging from -1 to 1 [S2, unofficial].
3. Generate a ray bundle from the pixels using `get_rays_from_pixels(xy_grid, camera)` [S2, unofficial].
   - The camera unprojects NDC points to world space to determine ray origins and directions [S2, unofficial].
4. Perform a forward pass through the model: `color = model(ray_bundle, light_dir)` [S2, unofficial].
   - The `implicit_fn` evaluates the scalar field at sample points along the rays [S2, unofficial].
   - The `sampler` identifies the first hit where the value crosses the threshold [S1, unofficial].
   - The `renderer` computes the color using the gradient-based normal at the hit point [S1, unofficial].
5. Reshape the output color tensor to `(H, W, 3)` and convert to a numpy array [S2, unofficial].

**Formulas:**
- Normalized coordinates: $x, y \in [-1, 1]$ [S2, unofficial].
- Thresholding: Stop at first voxel $v$ where $v \ge \text{threshold}$ [S1, unofficial].
- Shading: Color is computed using the gradient $\nabla v$ at the hit voxel [S1, unofficial].

**Parameters:**
- Threshold: Determines the surface isovalue [S1, unofficial].
- Image Size: Width and Height of the output image [S2, unofficial].

**Edge Cases:**
- Terracing: Visible when slice spacing is large, as surface rendering does not integrate over it [S1, unofficial].
- No Hit: If no voxel exceeds the threshold, the pixel may remain background color (implementation detail [G]).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_surface(volume, threshold, cameras, image_size):
    # Initialize model components
    model = SurfaceModel(cfg)
    
    all_images = []
    for camera in cameras:
        # Generate normalized image plane coordinates
        xy_grid = get_normalized_image_plane_coordinates(image_size)
        
        # Generate rays from pixels
        ray_bundle = get_rays_from_pixels(xy_grid, camera)
        
        # Forward pass to get color
        # The model handles implicit function evaluation, sampling, and rendering
        color = model(ray_bundle, light_dir=None)
        
        # Reshape and convert to numpy array
        # Assuming color is a tensor of shape (H*W, 3)
        image = color.view(image_size[1], image_size[0], 3).detach().cpu().numpy()
        all_images.append(image)
        
    return all_images
```

## Subtopics

### [Real Time Cutting Operations](3494-real-time-cutting-operations.md)
Real Time Cutting Operations is a technique for clipping volume surfaces using cutting planes by maintaining near and far depth buffers per pixel, avoiding full re-rendering when planes change. It sets itself apart from siblings by pre-computing visibility intervals via scan-conversion of planes into depth buffers, allowing the renderer to discard intersections outside these intervals without re-tracing the entire volume.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray [S2, unofficial].
- **Memory Complexity:** Linear with respect to the number of pixels and samples [S2, unofficial].
- **Performance:** Measured at 390x844 on a synthetic 700 HU sphere, surface rendering exposed terracing artifacts due to 3 mm slice spacing [S1, unofficial].
- **Strengths:** No mesh generation overhead; direct shading from volume data [S1, unofficial].
- **Limitations:** Terracing artifacts at low resolution or large slice spacing [S1, unofficial].
## Background
Surface rendering is a mode in volume visualization that contrasts with volume rendering by stopping at the first threshold crossing [S1, unofficial]. It is used in medical imaging and scientific visualization to visualize specific structures within a volume [S1, unofficial].
## Sources
[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
