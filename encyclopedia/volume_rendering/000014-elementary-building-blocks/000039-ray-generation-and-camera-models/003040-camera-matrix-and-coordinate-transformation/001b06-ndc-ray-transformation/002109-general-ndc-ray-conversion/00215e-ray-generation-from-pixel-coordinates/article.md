# Ray Generation from Pixel Coordinates

Also called: Camera Raycasting, Pixel to Ray Unprojection, Ray Direction Calculation, Ray Generation from Camera/NDC, Ray Generation from Pixels, Ray Generation from Screen Coordinates

## Overview
Ray Generation from Pixel Coordinates converts 2D screen pixel coordinates into 3D ray origins and directions in world space, enabling volume rendering or ray marching to begin from the camera's perspective [S1, S5, S7, unofficial]. It differs from [Pinhole Camera Projection (Pixel to Camera)](3162-pinhole-camera-projection-pixel-to-camera.md) by producing a full ray (origin and direction) for traversal rather than a single 3D point at a specific depth, and from [General NDC Ray Conversion](2429-general-ndc-ray-conversion.md) by typically handling the full transformation from pixel indices to world-space rays, often assuming the origin is the camera center [S4, S7, unofficial].

## How it works
The method maps pixel indices to Normalized Device Coordinates (NDC), transforms these to world space to determine the ray origin and a point on the image plane, and computes the normalized direction vector.

### Inputs and Outputs
*   **Inputs**:
    *   Pixel coordinates $(u, v)$ or a grid of them, where $u \in [0, W-1]$ and $v \in [0, H-1]$ [S1, S3, S5, unofficial].
    *   Image resolution $W \times H$ [S1, S5, unofficial].
    *   Camera parameters: Projection-View matrix ($P \cdot V$) or intrinsic/extrinsic parameters (focal length, principal point, rotation, translation) [S1, S4, S7, unofficial].
    *   Field of View (FOV) or focal length [S5, S8, unofficial].
*   **Outputs**:
    *   Ray origins $\mathbf{o}$ (typically the camera center) [S4, S7, unofficial].
    *   Ray directions $\mathbf{d}$ (normalized vectors) [S4, S7, unofficial].

### Data Structures
*   **Pixel Grid**: A tensor or array of shape $(H, W, 2)$ or $(N, 2)$ containing $(u, v)$ coordinates [S3, S6, unofficial].
*   **NDC Points**: Homogeneous coordinates $(x_{ndc}, y_{ndc}, z_{ndc}, 1)$ [S1, S2, unofficial].
*   **RayBundle**: A structure containing:
    *   `rays_o`: Tensor of shape $(N, 3)$ for origins [S4, S6, unofficial].
    *   `rays_d`: Tensor of shape $(N, 3)$ for directions [S4, S6, unofficial].
    *   `sample_points`, `sample_lengths`: Zero-initialized tensors (for volume rendering traversal) [S4, S6, unofficial].

### Steps
1.  **Generate Pixel Coordinates**: Create a grid of pixel indices $(u, v)$ for the image resolution $W \times H$ [S3, unofficial].
2.  **Convert to NDC**: Map pixel coordinates to NDC space $[-1, 1]$:
    $$ x_{ndc} = \frac{2u}{W} - 1 $$
    $$ y_{ndc} = \frac{2v}{H} - 1 $$
    [S1, S3, unofficial]. Note: Some implementations center the coordinate system by subtracting half the resolution from `fragCoord` [S5, unofficial].
3.  **Define Z for Near/Far Planes**:
    *   For the **Origin**: Use $z_{ndc} = -1$ (or $0$ in some NDC conventions where near is 0) corresponding to the near plane [S1, S4, unofficial]. In OpenGL-style NDC, near is $-1$, far is $1$. In some ray marching contexts, $z=0$ is used for the origin if the camera is at the origin in view space [S1, unofficial].
    *   For the **Direction Point**: Use $z_{ndc} = 1$ (or a value corresponding to the image plane distance) [S1, S4, unofficial].
4.  **Transform to World Space**:
    *   **Method A (Matrix Inverse)**: Construct homogeneous NDC points $\mathbf{p}_{ndc} = [x_{ndc}, y_{ndc}, z_{ndc}, 1]^T$. Multiply by the inverse Projection-View matrix $M^{-1}$:
        $$ \mathbf{p}_{world\_hom} = M^{-1} \mathbf{p}_{ndc} $$
        Perform perspective division:
        $$ \mathbf{p}_{world} = \left[ \frac{x_{world}}{w}, \frac{y_{world}}{w}, \frac{z_{world}}{w} \right] $$
        [S1, unofficial].
    *   **Method B (Camera Unproject)**: Use a camera utility `unproject_points(ndc_points, from_ndc=True)` which internally handles the matrix inversion and perspective division [S2, S4, S6, unofficial].
    *   **Method C (Analytic FOV)**: If using FOV and no complex projection matrix, calculate the direction directly in camera space and rotate to world space [S5, S7, unofficial].
5.  **Compute Ray Origin**: Set $\mathbf{o}$ to the camera center (eye position) in world space [S4, S6, S7, unofficial]. This can be obtained via `camera.get_camera_center()` or extracted from the extrinsic matrix [S4, S6, unofficial].
6.  **Compute Ray Direction**:
    *   Calculate the world-space point $\mathbf{p}_{plane}$ on the image plane (using $z_{ndc}=1$ or equivalent) [S1, S4, unofficial].
    *   Compute direction: $\mathbf{d}_{raw} = \mathbf{p}_{plane} - \mathbf{o}$ [S4, S7, unofficial].
    *   Normalize: $\mathbf{d} = \frac{\mathbf{d}_{raw}}{\|\mathbf{d}_{raw}\|}$ [S4, S6, S7, unofficial].
7.  **Handle Masking/Sampling** (Optional): If only a subset of rays is needed (e.g., for extra sampling), select random indices from valid masks and gather the corresponding rays [S7, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def generate_rays_from_pixels(W, H, camera, device='cpu'):
    """
    Generate ray origins and directions for all pixels in an image.
    
    Args:
        W: Image width
        H: Image height
        camera: Camera object with methods get_camera_center() and unproject_points()
        device: Device to run computations on
        
    Returns:
        rays_o: Tensor of shape (H*W, 3)
        rays_d: Tensor of shape (H*W, 3)
    """
    # 1. Generate pixel coordinates
    u = torch.arange(W, device=device)
    v = torch.arange(H, device=device)
    # meshgrid with indexing='xy' gives shape (H, W, 2)
    uv_grid = torch.meshgrid(u, v, indexing='xy')
    uv_grid = torch.stack(uv_grid, dim=-1) # (H, W, 2)
    
    N = W * H
    pixels = uv_grid.view(-1, 2) # (N, 2)
    
    # 2. Convert to NDC: x_ndc = (2*u/W) - 1, y_ndc = (2*v/H) - 1
    ndc_xy = 2.0 * pixels / torch.tensor([W, H], device=device) - 1.0 # (N, 2)
    
    # 3. Create NDC points for Origin (z_ndc = -1 for OpenGL near plane)
    z_ndc_origin = -1.0
    ndc_points_origin = torch.cat([ndc_xy, torch.full((N, 1), z_ndc_origin, device=device), torch.ones((N, 1), device=device)], dim=-1) # (N, 4)
    
    # 4. Create NDC points for Direction Point (z_ndc = 1 for OpenGL far plane / image plane)
    z_ndc_plane = 1.0
    ndc_points_plane = torch.cat([ndc_xy, torch.full((N, 1), z_ndc_plane, device=device), torch.ones((N, 1), device=device)], dim=-1) # (N, 4)
    
    # 5. Unproject to World Space
    # Assumes camera.unproject_points handles inverse PV matrix and perspective division
    world_points_origin = camera.unproject_points(ndc_points_origin, from_ndc=True) # (N, 3)
    world_points_plane = camera.unproject_points(ndc_points_plane, from_ndc=True) # (N, 3)
    
    # 6. Get Camera Center
    cam_center = camera.get_camera_center() # (3,)
    rays_o = cam_center.expand(N, -1) # (N, 3)
    
    # 7. Compute Directions
    rays_d_raw = world_points_plane - rays_o # (N, 3)
    rays_d = torch.nn.functional.normalize(rays_d_raw, dim=-1) # (N, 3)
    
    return rays_o, rays_d

def generate_rays_analytic_fov(W, H, fov_deg, cam_world_pos, cam_world_rot, device='cpu'):
    """
    Alternative method using analytic FOV calculation.
    """
    N = W * H
    fov_rad = math.radians(fov_deg)
    
    u = torch.arange(W, device=device)
    v = torch.arange(H, device=device)
    uv_grid = torch.meshgrid(u, v, indexing='xy')
    uv_grid = torch.stack(uv_grid, dim=-1).float() # (H, W, 2)
    
    # Center coordinates: (0,0) at center
    xy = uv_grid - torch.tensor([W/2.0, H/2.0], device=device) # (H, W, 2)
    
    # Calculate Z in camera space: z = size.y / tan(fov / 2)
    z_cam = H / math.tan(fov_rad / 2.0) # Scalar
    
    # Construct direction in camera space: float3(xy, -z)
    xy_expanded = xy.unsqueeze(-1) # (H, W, 2, 1)
    z_tensor = torch.full((H, W, 1), -z_cam, device=device) # (H, W, 1)
    dir_cam = torch.cat([xy_expanded, z_tensor], dim=-1).view(-1, 3) # (N, 3)
    
    # Normalize
    dir_cam = torch.nn.functional.normalize(dir_cam, dim=-1) # (N, 3)
    
    # Rotate to World Space: world_dir = dir_cam @ R.T
    rays_d = dir_cam @ cam_world_rot.T # (N, 3)
    
    # Origin is camera position
    rays_o = cam_world_pos.expand(N, -1) # (N, 3)
    
    return rays_o, rays_d
```

### Parameters
*   **Field of View (FOV)**: Vertical FOV in degrees, typically 45.0 [S5, unofficial].
*   **Resolution**: Image width $W$ and height $H$ [S5, unofficial].
*   **ProjectionZPos**: Z-coordinate in NDC for the near plane, often 0 or -1 depending on the API [S1, unofficial].

### Edge Cases and Pitfalls
*   **Coordinate System**: Metal/OpenGL NDC may have $(0,0)$ at the upper left, requiring adjustment to center $(0,0)$ [S5, unofficial].
*   **Device Mismatch**: Hardcoding `.cuda()` breaks device-agnostic execution; use `camera.device` [S4, unofficial].
*   **Aspect Ratio**: Analytic methods must correctly handle aspect ratio if FOV is vertical vs horizontal [S8, unofficial].

## Subtopics

### NDC-to-World Ray Generation
Computes the world-space position of a ray origin for a given NDC pixel, assuming the camera is at the origin in view space, facing positive Z, with a near plane at Z=1 [S8, unofficial]. It calculates the near plane dimensions from the horizontal half-FOV and aspect ratio, then offsets the NDC coordinates to find the view-space position, which is then translated by the camera position [S8, unofficial]. Choose this when working with simple view-space to world-space transformations without a full projection matrix [S8, unofficial].

### NDC to Screen Camera Conversion
Transforms a camera from NDC space to screen pixel coordinates for rasterization [S9, unofficial]. It calculates screen focal length and principal point based on image size and returns a `PerspectiveCameras` object with `in_ndc=False` [S9, unofficial]. Choose this when converting between NDC-based rendering pipelines and screen-based rasterization pipelines [S9, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N = W \times H$, as each pixel is processed independently [S4, S7, unofficial].
*   **Memory Complexity**: $O(N)$ to store the ray origins and directions [S4, S6, unofficial].
*   **Strengths**: Accurate mapping from screen to world; handles arbitrary camera poses via matrix inversion [S1, S4, unofficial].
*   **Limitations**: Requires access to camera matrices or intrinsics; precision issues may arise with extreme FOV or large distances [S1, S5, unofficial].
*   **When to Use**: When rendering from a known camera position and orientation [S4, S7, unofficial].

## Background
Ray generation from pixel coordinates is a fundamental step in ray tracing and volume rendering, originating from the need to map 2D image samples to 3D space [S1, S5, unofficial]. The method relies on the inverse of the camera projection matrix to unproject pixels to world space [S1, S4, unofficial].

## Sources
[S1] SamCZ/RayTracer, https://github.com/SamCZ/RayTracer, fetched 2026-10-24 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S4] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S6] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S7] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S8] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S9] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
