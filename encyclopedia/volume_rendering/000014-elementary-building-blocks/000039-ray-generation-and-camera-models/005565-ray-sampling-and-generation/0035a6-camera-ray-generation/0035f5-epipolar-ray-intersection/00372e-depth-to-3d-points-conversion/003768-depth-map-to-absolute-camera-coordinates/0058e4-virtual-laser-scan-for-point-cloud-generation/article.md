# Virtual Laser Scan for Point Cloud Generation

Also called: Scan class, get_surface_point_cloud via scan

## Overview
A method to generate a point cloud with normals from a mesh by rendering normal and depth buffers from a specific camera viewpoint and reprojecting the buffer data into world space [S1, unofficial]. It differs from direct mesh sampling by relying on rasterization to handle visibility and occlusion, effectively acting as a "virtual laser scan" from a single viewpoint [S1, unofficial]. It builds on [Depth Map to Absolute Camera Coordinates](7984-depth-map-to-absolute-camera-coordinates.md) by providing the specific reprojection logic required to convert rendered buffers into 3D points [S1, unofficial].

## How it works
**Inputs:**
- `mesh`: The input 3D mesh object.
- `camera_transform`: A $4 \times 4$ transformation matrix representing the camera's pose in world space.
- `resolution`: Integer, width and height of the rendered buffers (default 400) [S1, unofficial].
- `calculate_normals`: Boolean, whether to compute and store normals (default True) [S1, unofficial].
- `fov`: Field of view in radians (default 1) [S1, unofficial].
- `z_near`: Near plane distance (default 0.1) [S1, unofficial].
- `z_far`: Far plane distance (default 10) [S1, unofficial].

**Outputs:**
- `self.points`: Array of shape $(N, 3)$ containing 3D coordinates in world space.
- `self.normals`: Array of shape $(N, 3)$ containing unit normals in world space (if `calculate_normals` is True).

**Data Structures:**
- `depth_buffer`: Rendered depth values from the graphics API.
- `color_buffer`: Rendered normal buffer (RGB8 format) if `calculate_normals` is True.
- `projection_matrix`: $4 \times 4$ matrix derived from camera intrinsics.
- `camera_position`: $3 \times 1$ vector derived from `camera_transform`.

**Steps:**

1.  **Initialize Camera:**
    Compute `camera_position` from `camera_transform`:
    $$ \text{camera\_position} = (\text{camera\_transform} \times [0, 0, 0, 1]^T)[:3] $$
    Create a perspective camera with `yfov=fov`, `aspectRatio=1.0`, `znear=z_near`, `zfar=z_far`. Extract the `projection_matrix` [S1, unofficial].

2.  **Render Buffers:**
    Render the `mesh` using the camera. Obtain `depth_buffer` and `color_buffer` (if `calculate_normals` is True) at the specified `resolution` [S1, unofficial].

3.  **Filter Valid Pixels:**
    Find indices where depth is non-zero:
    $$ \text{indices} = \text{argwhere}(\text{depth\_buffer} \neq 0) $$
    Set depth values equal to 0 to infinity to exclude them from further processing:
    $$ \text{depth\_buffer}[\text{depth\_buffer} == 0] = \infty $$
    [S1, unofficial]

4.  **Revert Depth Processing:**
    Revert to linear clip-space depth $z_{clip}$ using:
    $$ z_{clip} = \frac{z_{far} + z_{near} - \frac{2 z_{near} z_{far}}{depth_{buffer}}}{z_{far} - z_{near}} $$
    Store this as `self.depth` [S1, unofficial].

5.  **Map to Clip Space:**
    Create a points array of shape $(N, 4)$ initialized with ones in the w-component.
    Map pixel indices to normalized device coordinates (NDC) clip space:
    $$ \text{points}[:, [1, 0]] = \frac{\text{indices}}{\text{resolution} - 1} \times 2 - 1 $$
    Negate the y-component to account for coordinate system differences (e.g., screen vs. clip space):
    $$ \text{points}[:, 1] *= -1 $$
    Set the z-component to the calculated clip-space depth:
    $$ \text{points}[:, 2] = \text{self.depth}[\text{indices}[:, 0], \text{indices}[:, 1]] $$
    [S1, unofficial]

6.  **Transform to World Space:**
    Compute the clipping-to-world matrix:
    $$ \text{clipping\_to\_world} = \text{camera\_transform} \times \text{inv}(\text{projection\_matrix}) $$
    Transform points:
    $$ \text{points} = \text{points} \times \text{clipping\_to\_world}^T $$
    Perform perspective divide:
    $$ \text{points} = \text{points} / \text{points}[:, 3][:, \text{newaxis}] $$
    Store the final 3D points:
    $$ \text{self.points} = \text{points}[:, :3] $$
    [S1, unofficial]

7.  **Compute Normals (Optional):**
    If `calculate_normals` is True:
    Decode normals from the color buffer (assuming RGB8 encoding where $[0, 255]$ maps to $[-1, 1]$):
    $$ \text{normals} = \frac{\text{color\_buffer}[\text{indices}[:, 0], \text{indices}[:, 1]]}{255} \times 2 - 1 $$
    Compute vector from camera to points:
    $$ \text{camera\_to\_points} = \text{camera\_position} - \text{self.points} $$
    Determine orientation via dot product:
    $$ \text{normal\_orientation} = \text{einsum}('ij,ij->i', \text{camera\_to\_points}, \text{normals}) $$
    Flip normals pointing away from the camera:
    $$ \text{normals}[\text{normal\_orientation} < 0] *= -1 $$
    Store normals:
    $$ \text{self.normals} = \text{normals} $$
    [S1, unofficial]

**Pitfalls:**
If the camera is inside the model, the visibility check in `is_visible` must account for depth comparison against the stored depth buffer to determine if a point is occluded [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class Scan:
    def __init__(self, mesh, camera_transform, resolution=400, calculate_normals=True, 
                 fov=1.0, z_near=0.1, z_far=10.0):
        self.mesh = mesh
        self.camera_transform = camera_transform
        self.resolution = resolution
        self.calculate_normals = calculate_normals
        self.z_near = z_near
        self.z_far = z_far
        
        # Compute camera position
        self.camera_position = np.matmul(self.camera_transform, np.array([0, 0, 0, 1]))[:3]
        
        # Sources do not provide the full implementation of camera creation and rendering [G].
        self.projection_matrix = np.eye(4)
        depth_buffer, color_buffer = self._render_buffers()
        
        # Filter valid pixels
        indices = np.argwhere(depth_buffer != 0)
        depth_buffer[depth_buffer == 0] = float('inf')
        
        if indices.size == 0:
            self.points = np.empty((0, 3))
            self.normals = None
            return

        # Revert depth processing to get clip-space depth
        self.depth = (z_far + z_near - (2.0 * z_near * z_far) / depth_buffer) / (z_far - z_near)
        
        # Map pixel indices to clip space coordinates
        points = np.ones((indices.shape[0], 4))
        points[:, [1, 0]] = indices.astype(float) / (resolution - 1) * 2 - 1
        points[:, 1] *= -1  # Negate y
        points[:, 2] = self.depth[indices[:, 0], indices[:, 1]]
        
        # Transform to world space
        clipping_to_world = np.matmul(self.camera_transform, np.linalg.inv(self.projection_matrix))
        points = np.matmul(points, clipping_to_world.transpose())
        
        # Perspective divide
        points /= points[:, 3][:, np.newaxis]
        
        self.points = points[:, :3]
        
        if calculate_normals:
            # Decode normals from color buffer
            normals = color_buffer[indices[:, 0], indices[:, 1]] / 255.0 * 2.0 - 1.0
            
            # Compute vector from camera to points
            camera_to_points = self.camera_position - self.points
            
            # Determine normal orientation
            normal_orientation = np.einsum('ij,ij->i', camera_to_points, normals)
            
            # Flip normals pointing away from camera
            normals[normal_orientation < 0] *= -1
            
            self.normals = normals
        else:
            self.normals = None

    def _render_buffers(self):
        """Helper to render depth and normal buffers using a graphics API."""
        # Sources do not give the rendering implementation; typically handled by a graphics API [G].
        return np.zeros((self.resolution, self.resolution)), np.zeros((self.resolution, self.resolution, 3))

    def is_visible(self, points):
        """Check if points are visible from this scan's perspective."""
        # Sources do not give the full implementation of is_visible [G].
        # If the camera is inside the model, the visibility check must account for depth comparison 
        # against the stored depth buffer to determine if a point is occluded [S1, unofficial].
        return np.ones(points.shape[0], dtype=bool)
```

## Subtopics

### Surface Point Cloud Generation via Scans
A variant that generates a complete surface point cloud by aggregating scans from multiple viewpoints [S1, unofficial]. It uses a Fibonacci sphere distribution to generate camera angles, ensuring uniform coverage [S1, unofficial]. For each angle, a `Scan` object is created with specific parameters: `fov=1.0472`, `z_near=bounding_radius`, `z_far=3*bounding_radius`, and `resolution=400` [S1, unofficial]. The points and normals from all scans are concatenated to form the final `SurfacePointCloud` [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Originated in the `mesh_to_sdf` library by marian42, which uses virtual laser scans to generate point clouds for Signed Distance Field (SDF) reconstruction [S1, unofficial].

## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
