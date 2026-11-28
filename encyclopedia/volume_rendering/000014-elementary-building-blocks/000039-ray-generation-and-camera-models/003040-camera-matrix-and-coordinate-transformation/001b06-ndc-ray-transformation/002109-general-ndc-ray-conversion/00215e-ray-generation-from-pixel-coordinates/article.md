# Ray Generation from Pixel Coordinates

Also called: Beaming Rays, Camera Raycasting, Pixel to Ray Unprojection, Ray Direction Calculation, Ray Generation from Camera/NDC, Ray Generation from Pixels, Ray Generation from Screen Coordinates

## Overview
Ray Generation from Pixel Coordinates converts 2D screen pixel coordinates into 3D world-space ray origins and directions, serving as the foundational step for raymarching and volume rendering [S1, S5, S9, unofficial]. It solves the problem of "lifting" 2D image data into 3D space by establishing the geometric relationship between the camera sensor and the world [S3, S6, unofficial]. Unlike [Pinhole Camera Projection (Pixel to Camera)](3162-pinhole-camera-projection-pixel-to-camera.md), which focuses on the algebraic backprojection to camera space, this method explicitly constructs world-space rays, often leveraging the inverse Projection-View matrix or explicit NDC-to-world transformations [S1, S4, S5, unofficial]. It builds on [General NDC Ray Conversion](2429-general-ndc-ray-conversion.md) by handling the final mapping from pixel indices to the specific camera model's world coordinates [S4, S5, unofficial].

## How it works
**Inputs:**
*   Pixel coordinates $(u, v)$ or a grid of pixel indices [S3, S4, unofficial].
*   Image resolution $(W, H)$ [S1, S3, unofficial].
*   Camera parameters: Intrinsic matrix $K$ (or focal length $f$, principal point $c_x, c_y$), Extrinsics (Rotation $R$, Translation $T$), and Projection matrix $P$ (or View-Projection matrix $VP$) [G].
*   Field of View (FOV) and aspect ratio, if using explicit geometric construction [S6, unofficial].

**Outputs:**
*   Ray origins $\mathbf{o} \in \mathbb{R}^3$ (typically the camera center) [S2, unofficial].
*   Ray directions $\mathbf{d} \in \mathbb{R}^3$ (normalized vectors pointing from origin through the pixel) [S6, unofficial].

**Data Structures:**
*   **RayBundle**: A structure containing tensors for origins and directions.
    *   `rays_o`: Shape $(N, 3)$, where $N = W \times H$ (or batch size $\times N$) [S2, S4, unofficial].
    *   `rays_d`: Shape $(N, 3)$, normalized direction vectors [S2, S3, unofficial].
    *   `sample_points`: Shape $(N, K, 3)$, initialized to zeros for marching steps [S2, S4, S5, unofficial].

**Steps:**

1.  **Generate Pixel Grid**: Create a grid of pixel coordinates $(u, v)$ for all pixels in the image [S3, unofficial].
    *   $u \in [0, W-1]$, $v \in [0, H-1]$.
    *   Note: Coordinate systems vary. Metal uses top-left $(0,0)$ [S6, unofficial], while some implementations assume center $(0,0)$ [S8, unofficial].

2.  **Convert to Normalized Device Coordinates (NDC)**:
    *   Map pixel coordinates to NDC space $[-1, 1]$ [S3, unofficial].
    *   Formula:
        $$ x_{ndc} = \frac{2 u}{W} - 1 $$
        $$ y_{ndc} = \frac{2 v}{H} - 1 $$
    *   For a point on the near plane at $Z=1$ (in camera/NDC space), the 3D NDC point is $\mathbf{p}_{ndc} = [x_{ndc}, y_{ndc}, 1.0]^T$ [S1, S3, S5, unofficial].
    *   *Alternative (Explicit FOV)*: If not using matrix inversion, calculate centered UVs:
        $$ xy = [u - W/2.0, v - H/2.0] $$
        $$ z = \frac{H}{2 \tan(\text{FOV}_{vertical} / 2)} $$
        $$ \mathbf{d}_{cam} = \text{normalize}([xy_x, xy_y, -z]) $$ [S6, S8, unofficial].

3.  **Unproject to World Space**:
    *   **Method A: Inverse Projection-View Matrix** [S1, S4, S5, unofficial]:
        *   Form homogeneous NDC point: $\mathbf{p}_{ndc}^{hom} = [x_{ndc}, y_{ndc}, 1.0, 1.0]^T$ [S2, S5, unofficial].
        *   Compute inverse matrix $M^{-1} = (VP)^{-1}$ [S1, unofficial].
        *   Transform: $\mathbf{p}_{world}^{hom} = M^{-1} \mathbf{p}_{ndc}^{hom}$.
        *   Perform perspective division: $\mathbf{p}_{world} = \mathbf{p}_{world}^{hom} / w$, where $w$ is the fourth component of $\mathbf{p}_{world}^{hom}$ [S1, unofficial].
        *   This yields a point on the image plane in world space [S4, unofficial].
    *   **Method B: Camera Unproject API** [S2, S3, S7, unofficial]:
        *   Use a library function (e.g., `camera.unproject_points`) that takes NDC points and returns world coordinates, handling the matrix inversion internally [S2, S5, unofficial].

4.  **Determine Ray Origin**:
    *   The ray origin is the camera center in world space [S2, unofficial].
    *   $\mathbf{o} = \text{camera.get_camera_center()}$ [S2, S4, S5, unofficial].
    *   If using extrinsics $R, T$, the camera center is $-\mathbf{R}^T \mathbf{T}$ [S9, unofficial].

5.  **Compute Ray Direction**:
    *   Calculate the vector from the origin to the unprojected world point on the image plane [S5, unofficial].
    *   $\mathbf{d}_{raw} = \mathbf{p}_{world} - \mathbf{o}$ [S2, S3, S5, unofficial].
    *   Normalize the direction:
        $$ \mathbf{d} = \frac{\mathbf{d}_{raw}}{\|\mathbf{d}_{raw}\|} $$ [S2, S3, S5, S7, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def generate_rays_from_pixels(image_size, camera):
    """
    Generates ray origins and directions for all pixels in an image.
    
    Args:
        image_size: Tuple (W, H)
        camera: Camera object with methods get_camera_center() and unproject_points()
        
    Returns:
        rays_o: Tensor of shape (N, 3)
        rays_d: Tensor of shape (N, 3)
    """
    W, H = image_size
    N = W * H
    
    # Step 1: Generate pixel coordinates
    # Create meshgrid of pixel indices
    # Note: meshgrid(y, x) creates grids where first dim is y (rows) and second is x (cols)
    y_indices, x_indices = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    
    # Flatten to list of (x, y) pairs
    # Shape: (N, 2)
    pixel_coords = torch.stack([x_indices.flatten(), y_indices.flatten()], dim=-1).float()
    
    # Step 2: Convert to NDC
    # Map [0, W-1] -> [-1, 1]
    ndc_x = (2.0 * pixel_coords[:, 0]) / W - 1.0
    ndc_y = (2.0 * pixel_coords[:, 1]) / H - 1.0
    
    # Construct NDC points on the near plane (Z=1 in camera/NDC space)
    # Shape: (N, 3)
    ndc_points = torch.stack([ndc_x, ndc_y, torch.ones_like(ndc_x)], dim=-1)
    
    # Step 3: Unproject to World Space
    # Append homogeneous coordinate 1.0
    ndc_points_hom = torch.cat([ndc_points, torch.ones_like(ndc_points[..., :1])], dim=-1) # Shape: (N, 4)
    
    # Use camera's unproject method. 
    # Assumes camera handles the inverse VP matrix multiplication and perspective division.
    # from_ndc=True indicates input is in NDC space [-1, 1]
    world_points = camera.unproject_points(ndc_points_hom, from_ndc=True) # Shape: (N, 3)
    
    # Step 4: Get Ray Origin
    # Get camera center in world space
    cam_center = camera.get_camera_center() # Shape: (3,)
    
    # Expand to match number of rays
    rays_o = cam_center.expand(N, -1) # Shape: (N, 3)
    
    # Step 5: Compute Ray Direction
    # Vector from origin to world point on image plane
    rays_d = world_points - rays_o # Shape: (N, 3)
    
    # Normalize directions
    rays_d = F.normalize(rays_d, dim=-1) # Shape: (N, 3)
    
    return rays_o, rays_d
```

**Edge Cases and Pitfalls:**
*   **Coordinate System Origin**: Metal/OpenGL uses top-left $(0,0)$ for pixels, while some mathematical derivations assume center $(0,0)$ [S6, S8, unofficial]. Implementations must adjust `fragCoord` by subtracting `size / 2.0` if using the explicit FOV method [S6, S8, unofficial].
*   **Homogeneous Dimension**: When appending the homogeneous coordinate, ensure the shape matches the expected input of the unprojection function. Some APIs expect $(N, 4)$, others handle $(N, 3)$ NDC internally [S2, S4, unofficial].
*   **Device Agnosticism**: Hardcoding `.cuda()` breaks execution on CPU or other devices [S4, unofficial]. Use `camera.device` [S7, unofficial].
*   **Aspect Ratio**: If the explicit FOV method is used, the vertical FOV must be combined with the aspect ratio (implicitly via $W/H$ in the $xy$ calculation) to prevent distortion [S8, unofficial].

## Subtopics

### [Pinhole Camera Projection (Pixel to Camera)](3162-pinhole-camera-projection-pixel-to-camera.md)
Converts 2D pixel coordinates $(u, v)$ and depth $z$ into 3D homogeneous coordinates in the camera frame using the intrinsic matrix. It implements the backprojection operator $\pi^{-1}$, mapping a pixel $p$ and depth $z$ to a 3D point $x_c$ via $z K^{-1} \bar{p}$. It is the inverse of the forward camera projection operator $\pi(x_c) \approx K x_c$, building on [Ray Generation from Pixel Coordinates](2514-ray-generation-from-pixel-coordinates.md) and distinct from [NDC-to-World Ray Generation](3437-ndc-to-world-ray-generation.md).

### [NDC-to-World Ray Generation](3437-ndc-to-world-ray-generation.md)
NDC-to-World Ray Generation computes the world-space origin of a ray for a given Normalized Device Coordinate (NDC) pixel, assuming a pinhole camera model where the camera is at the origin in view space, facing the positive Z axis, with a near plane at $Z=1$. It differs from sibling methods like [Pinhole Camera Projection (Pixel to Camera)](3162-pinhole-camera-projection-pixel-to-camera.md) by explicitly constructing the ray origin in world space based on the field of view and aspect ratio, rather than just transforming pixel coordinates to camera space. A common pitfall in implementation is incorrectly offsetting the ray origin by screen-space coordinates, which causes field-of-view distortion and curvature artifacts.

### NDC to Screen Camera Conversion
Transforms a camera from normalized device coordinates (NDC) space to screen pixel coordinates for rasterization [S10, unofficial]. It calculates screen focal length as $f_{screen} = f \cdot \min(W, H) / 2.0$ and sets the principal point to $(W/2, H/2)$ [S10, unofficial]. This is the inverse operation of the projection step in ray generation, used for rendering rather than raycasting [S10, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N = W \times H$ [G]. The operations (grid generation, matrix multiplication, normalization) are vectorizable and parallelizable across pixels [G].
*   **Memory Complexity**: $O(N)$ to store origins and directions [G].
*   **Performance**: Highly efficient on GPUs due to tensor operations. The bottleneck is typically the matrix inversion if done per-frame (though $VP$ is usually constant per camera pose) or the unprojection API call overhead [G].
*   **Strengths**: Generalizable to any camera model supported by the unprojection API; handles arbitrary extrinsics and intrinsics [G].
*   **Limitations**: Requires accurate camera calibration. If the inverse matrix is singular (e.g., camera at infinity), it fails [G].

## Background
Ray generation from pixel coordinates is a fundamental technique in computer graphics [G]. It is used in volume rendering, neural radiance fields (NeRF), and 3D Gaussian Splatting [S2, S4, S5, S10, unofficial]. The specific implementation using NDC and inverse Projection-View matrices is standard in modern graphics APIs (OpenGL, Vulkan, Metal) and deep learning frameworks (PyTorch3D, DiffRast) [S1, S2, S4, S6, unofficial].

## Sources
[S1] SamCZ/RayTracer, https://github.com/SamCZ/RayTracer, fetched 2026-10-24 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S4] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
[S6] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S7] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S8] Painting with Math: A Gentle Study of Raymarching, https://blog.maximeheckel.com/posts/painting-with-math-a-gentle-study-of-raymarching/, fetched 2026-10-27 (unofficial)
[S9] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S10] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S11] sarfrazkhan66003/sinus-3d-viewer-pro, https://github.com/sarfrazkhan66003/sinus-3d-viewer-pro, fetched 2026-10-26 (unofficial)
