# Sample Streaming Payload Ray Computation

Also called: Ray Direction Computation, SampleStreamingPayload, Ray Generation from Pixels

## Overview
Sample Streaming Payload Ray Computation regenerates ray geometry (origin and direction) from stored pixel indices and camera parameters, enabling sample streaming and resumption in volume rendering pipelines [S1, unofficial]. It maps pixel coordinates to Normalized Device Coordinates (NDC) and unprojects them to world space to compute unit-length direction vectors [S3, S4, unofficial]. Unlike static ray generation, this method is designed for payloads that persist across frames or batches, allowing rays to be reconstructed on-demand from minimal state [S1, unofficial].

## How it works
**Inputs:**
- `pixel_index`: Integer identifier for the pixel (often derived from framebuffer index) [S1, unofficial].
- `frame_size`: Tuple `(W, H)` representing image width and height [S1, S4, unofficial].
- `camera`: Object or structure containing camera parameters, including:
  - `position`: Camera origin in world space [S1, S4, unofficial].
  - `direction`, `horizontal`, `vertical`: Basis vectors defining the view frustum [S1, unofficial].
  - `c2w`: Camera-to-world transformation matrix (4x4) [S2, S6, unofficial].
  - `focal_length`: Focal length parameter [S2, S6, unofficial].
  - `unproject_points`: Function/method to map NDC points to world space [S4, unofficial].
  - `get_camera_center`: Function/method to retrieve camera origin [S4, unofficial].
- `params`: Configuration object containing light direction for shadow rays [S1, unofficial].

**Outputs:**
- `RayBundle` or `Ray` structure containing:
  - `origins`: Tensor of shape `(N, 3)` or `(num_cameras, N, 3)` [S4, S5, unofficial].
  - `directions`: Tensor of shape `(N, 3)` or `(num_cameras, N, 3)` with unit length [S4, S5, unofficial].
  - `sample_points`, `sample_lengths`: Zero-initialized tensors for volume integration [S4, unofficial].

**Data Structures:**
- `RayBundle`: A container holding origins, directions, and sample metadata. In PyTorch implementations, these are typically `torch.Tensor` objects [S4, unofficial].
- `Ray` (C++ style): Struct with fields `org` (vec3f), `dir` (vec3f), `alpha` (float), `color` (vec3f) [S1, unofficial].

**Steps:**

1. **Generate Pixel Coordinates:**
   Create 2D grids of column (`i`) and row (`j`) indices [S2, unofficial].
   - For streaming/pixel index approach: Compute `ix` and `iy` from `pixel_index` [S1, unofficial]:
     $$ ix = \text{pixel\_index} \pmod W $$
     $$ iy = \lfloor \text{pixel\_index} / W \rfloor $$
   - For full image generation: Create arrays `x` and `y` using `np.linspace(0, W, endpoint=False)` and `np.linspace(0, H, endpoint=False)` [S4, unofficial].

2. **Normalize to NDC Space:**
   Map pixel coordinates to NDC space $[-1, 1]$ [S3, unofficial].
   - Using pixel indices:
     $$ \text{screen}_x = \frac{ix + 0.5}{W} $$
     $$ \text{screen}_y = \frac{iy + 0.5}{H} $$
     $$ \text{ndc}_x = 2 \cdot \text{screen}_x - 1 $$
     $$ \text{ndc}_y = 2 \cdot \text{screen}_y - 1 $$
     [S1, S4, unofficial]
   - Using meshgrid: Stack `y` and `x` grids, reverse dimensions to shape `(W*H, 2)`, negate, and append `1` for homogeneous coordinates [S3, S4, unofficial].

3. **Unproject to World Space:**
   - **Method A (Using Unproject Function):**
     Concatenate `1` to NDC points to form homogeneous coordinates [S4, unofficial].
     Call `camera.unproject_points(ndc_points, from_ndc=True)` to get world space points `world_pts` [S4, unofficial].
     Get camera center `rays_o` via `camera.get_camera_center()` [S4, unofficial].
     Repeat `rays_o` for all pixels [S4, unofficial].
   - **Method B (Using Camera Basis):**
     Compute raw directions in camera space:
     $$ x = \frac{i - W/2}{f} $$
     $$ y = -\frac{j - H/2}{f} $$
     $$ z = -1 $$
     [S2, S6, unofficial]
     Combine with camera basis vectors `u` (x-axis), `v` (y-axis), `w` (z-axis):
     $$ \text{raw\_dir} = x \cdot u + y \cdot v + w $$
     [S5, unofficial]

4. **Compute Ray Origins:**
   - **Method A:** `rays_o` is the camera center, broadcasted to match the number of rays [S4, unofficial].
   - **Method B:** `rays_o` is the translation part of the `c2w` matrix (last column) [S6, unofficial].

5. **Compute Ray Directions:**
   - **Method A:**
     $$ \text{raw\_dir} = \text{world\_pts} - \text{rays\_o} $$
     [S4, unofficial]
   - **Method B:**
     Rotate `raw_dir` to world space using the upper-left 3x3 rotation submatrix of `c2w`:
     $$ \text{rays\_d} = \text{raw\_dir} \cdot \text{c2w}_{3\times3} $$
     [S2, S6, unofficial]

6. **Normalize Directions:**
   Compute L2 norm and divide:
   $$ \text{norm} = \sqrt{\sum_{k=1}^3 \text{rays\_d}_k^2} $$
   $$ \text{rays\_d} = \frac{\text{rays\_d}}{\text{norm}} $$
   [S4, S5, unofficial]

7. **Initialize Payload (Optional for Streaming):**
   Set `ray.alpha = alpha` (passed parameter), `ray.color = 0.f` (shadow) or `color` (main), and initialize sample points/lengths to zero [S1, S4, unofficial].

**Pseudocode (PyTorch-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_rays_from_pixels(pixel_indices, image_size, camera):
    """
    Compute ray origins and directions from pixel indices.
    
    Args:
        pixel_indices: Tensor of shape (N,) containing pixel indices.
        image_size: Tuple (W, H).
        camera: Object with attributes:
            - c2w: Tensor (4, 4) camera-to-world matrix.
            - focal_length: Float.
            - get_camera_center(): Returns Tensor (3,) camera position.
            - unproject_points(ndc_points, from_ndc=True): Returns Tensor (N, 3) world points.
    
    Returns:
        origins: Tensor (N, 3)
        directions: Tensor (N, 3)
    """
    W, H = image_size
    
    # Step 1: Compute screen coordinates from pixel index
    ix = pixel_indices % W
    iy = pixel_indices // W
    
    # Step 2: Normalize to NDC [-1, 1]
    # Add 0.5 for pixel center, then scale to [0, 1], then to [-1, 1]
    ndc_x = 2.0 * ((ix.float() + 0.5) / W) - 1.0
    ndc_y = 2.0 * ((iy.float() + 0.5) / H) - 1.0
    
    # Stack to (N, 2)
    ndc_points_2d = torch.stack([ndc_x, ndc_y], dim=-1)
    
    # Step 3: Unproject to world space
    # Append 1 for homogeneous coordinates
    ndc_points_homogeneous = torch.cat([ndc_points_2d, torch.ones_like(ndc_points_2d[..., -1:])], dim=-1)
    
    # Use camera's unproject function
    world_pts = camera.unproject_points(ndc_points_homogeneous, from_ndc=True)
    
    # Step 4: Get camera origin
    rays_o = camera.get_camera_center()
    # Broadcast origin to match number of rays
    rays_o = rays_o.unsqueeze(0).expand(world_pts.shape[0], -1)
    
    # Step 5: Compute directions
    rays_d = world_pts - rays_o
    
    # Step 6: Normalize directions
    norms = torch.linalg.norm(rays_d, dim=1, keepdim=True)
    rays_d = rays_d / norms
    
    return rays_o, rays_d

def compute_rays_from_grid(image_size, camera):
    """
    Compute rays for all pixels in an image using grid generation.
    """
    W, H = image_size
    
    # Step 1: Create pixel grids
    x = torch.linspace(0, W, W, endpoint=False)
    y = torch.linspace(0, H, H, endpoint=False)
    
    # Step 2: Meshgrid and reshape
    # torch.meshgrid returns (H, W) grids. Stack and reshape to (W*H, 2)
    y_grid, x_grid = torch.meshgrid(y, x, indexing='xy')
    xy_grid = torch.stack([x_grid, y_grid], dim=-1).view(-1, 2)
    
    # Step 3: Normalize to NDC
    ndc_x = 2.0 * (xy_grid[:, 0] / W) - 1.0
    ndc_y = 2.0 * (xy_grid[:, 1] / H) - 1.0
    ndc_points_2d = torch.stack([ndc_x, ndc_y], dim=-1)
    
    # Step 4: Unproject
    ndc_points_homogeneous = torch.cat([ndc_points_2d, torch.ones_like(ndc_points_2d[..., -1:])], dim=-1)
    world_pts = camera.unproject_points(ndc_points_homogeneous, from_ndc=True)
    
    # Step 5: Origins
    rays_o = camera.get_camera_center()
    rays_o = rays_o.unsqueeze(0).expand(world_pts.shape[0], -1)
    
    # Step 6: Directions
    rays_d = world_pts - rays_o
    norms = torch.linalg.norm(rays_d, dim=1, keepdim=True)
    rays_d = rays_d / norms
    
    return rays_o, rays_d
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in normalization can cause directions to deviate slightly from unit length. Re-normalization is recommended [S4, unofficial].
- **Coordinate System:** Ensure NDC space matches the camera's expected input (e.g., y-axis direction). Some implementations negate the NDC y-coordinate [S3, S4, unofficial].

## Operating characteristics

## Background
The method originates from implementations in neural volume rendering and NeRF pipelines, where efficient ray generation is critical for training and rendering [S2, S4, unofficial]. The `SampleStreamingPayload` concept is used in instantVNR for resuming volume rendering tasks [S1, unofficial]. The pixel-to-ray mapping via NDC unprojection is a standard technique in computer graphics, adapted here for deep learning-based volume rendering [S3, S4, unofficial].

## Sources
[S1] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S3] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S4] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S6] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
