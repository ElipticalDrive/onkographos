# Arbitrary Source-Detector Geometry Projector

Also called: Arbitrary Source-Detector Geometry, Axial cone-beam, Cone-Beam Geometry Projector, Flexible cone-beam, Parallel Beam Geometry, Parallel-Beam Geometry Projector

## Overview
The Arbitrary Source-Detector Geometry Projector is a flexible CT forward/backward projection model that allows the user to define the 3D location and orientation of every individual source-detector pair, overriding standard geometric assumptions [S1, unofficial]. It supports planar or curved detectors (axial) and arbitrary source/detector placement, enabling complex trajectories and non-equispaced projection angles [S1, unofficial]. Unlike standard Radon models which assume parallel rays, this geometry handles point-source cone-beam and fan-beam configurations where projection lines are non-parallel [S2, unofficial]. It builds on [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md) by mapping world-space voxel intersections to arbitrary detector coordinates.

## How it works
**Inputs:**
*   Volume data $V$ (3D array of voxel values, units $\text{mm}^{-1}$) [S1, unofficial].
*   Geometry specification:
    *   Source positions $\mathbf{s}_i$ for each projection angle $i$.
    *   Detector plane parameters: center, normal vector, pixel size, resolution.
    *   Projection angles $\Theta = \{\theta_1, \dots, \theta_N\}$ [S4, unofficial].
*   Optional: Source-to-Isocenter Distance (DSO), Source-to-Detector Distance (DSD) [S4, unofficial].

**Outputs:**
*   Projection data $P$ (2D array per angle, representing line integrals or attenuated intensity).

**Data Structures:**
*   **Detector Grid:** A 2D array of pixel centers in world coordinates.
*   **Ray Representation:** Origin $\mathbf{o} = \mathbf{s}_i$, Direction $\mathbf{d} = \text{normalize}(\mathbf{p}_{\text{pixel}} - \mathbf{s}_i)$ [S3, unofficial].

**Steps:**
1.  **Compute Detector Pixel Positions in World Coordinates:**
    a. Calculate $y_{\text{coords}}$ and $x_{\text{coords}}$ using linear spacing based on `detector_size` and `pixel_size` [S3, unofficial].
    b. Create a meshgrid with 'ij' indexing to form 2D coordinate arrays $X, Y$ [S3, unofficial].
    c. Set the $z$ coordinate to `center_offset_z` (SID) [S3, unofficial].
    d. Stack into a $(H, W, 3)$ array with order $[Z, Y, X]$ [S3, unofficial].
2.  **Compute Ray Directions:**
    For each pixel $p$ at position $\mathbf{p}_{\text{world}}$ and source $\mathbf{s}$:
    $$ \mathbf{d} = \mathbf{p}_{\text{world}} - \mathbf{s} $$
    $$ \mathbf{d} = \frac{\mathbf{d}}{\|\mathbf{d}\|} $$
    [S3, unofficial]
3.  **Ray Casting / Integration:**
    Trace rays from $\mathbf{s}$ along $\mathbf{d}$ through the volume $V$.
    *   Use a projector model such as Siddon [S1, unofficial], Joseph [S1, unofficial], or Distance Driven (DD) [S1, unofficial].
    *   Accumulate the integral of voxel values along the ray segment within the volume bounds.
4.  **Handle Geometry Variants:**
    *   **Parallel Beam:** Rays are parallel; source is at infinity. [S2, unofficial]
    *   **Cone Beam:** Point source, diverging rays. [S2, unofficial]
    *   **Flexible:** Arbitrary source/detector positions. [S1, unofficial]

**Parameters:**
*   `DSO` (Source-to-Isocenter Distance): Default 500 [S4, unofficial].
*   `DSD` (Source-to-Detector Distance): Default 800 [S4, unofficial].
*   `det_pixels`: Detector pixel count per side (assumed square). Default 256 [S4, unofficial].
*   `pixel_size`: Physical size of a detector pixel. Default 1.0 [S4, unofficial].
*   `angles`: Array of rotation angles from $0$ to $2\pi$. Default 180 angles [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Parallel Rays:** Standard Radon models assume parallel rays. Cone-beam and fan-beam geometries use a point source, making projection lines non-parallel [S2, unofficial].
*   **Precision:** Floating-point errors in ray-volume intersection calculations can cause artifacts. Use robust intersection algorithms (e.g., Siddon) [S1, unofficial].
*   **Boundary Conditions:** Rays may miss the volume entirely. Handle these by setting projection value to 0 or background intensity.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def arbitrary_projector(volume, sources, detector_centers, detector_normals, pixel_size, det_res):
    """
    Forward projection for arbitrary source-detector geometry.
    
    Args:
        volume: 3D numpy array (H, W, D) of voxel values.
        sources: List of 3D source positions (N, 3).
        detector_centers: List of detector center positions (N, 3).
        detector_normals: List of detector normal vectors (N, 3).
        pixel_size: Physical size of a detector pixel.
        det_res: Tuple (height, width) of detector resolution.
        
    Returns:
        projections: List of 2D arrays (N, H, W).
    """
    projections = []
    
    for i in range(len(sources)):
        s = sources[i]
        center = detector_centers[i]
        normal = detector_normals[i]
        
        # 1. Compute detector pixel positions in world coordinates
        # Create coordinate grids
        y_coords = np.linspace(-det_res[0]/2 * pixel_size, det_res[0]/2 * pixel_size, det_res[0])
        x_coords = np.linspace(-det_res[1]/2 * pixel_size, det_res[1]/2 * pixel_size, det_res[1])
        X, Y = np.meshgrid(x_coords, y_coords, indexing='ij')
        
        # Construct detector plane in local coordinates, then transform to world
        # Assuming detector is centered at 'center' with 'normal' as Z-axis
        # We need to construct a basis for the detector plane
        # Let's assume a simple case where detector is perpendicular to the ray from source to center
        # For arbitrary geometry, we need a full coordinate transform
        
        # Simplified: Assume detector is in a plane defined by center and normal
        # We need two orthogonal vectors u, v in the detector plane
        # u = normalize(cross(normal, arbitrary_vector))
        # v = cross(u, normal)
        
        arbitrary_vector = np.array([0, 0, 1])
        if np.abs(np.dot(normal, arbitrary_vector)) > 0.9:
            arbitrary_vector = np.array([1, 0, 0])
            
        u = np.cross(normal, arbitrary_vector)
        u = u / np.linalg.norm(u)
        v = np.cross(u, normal)
        
        # World coordinates of each pixel
        # Pixel position = center + x * u * pixel_size + y * v * pixel_size
        # But X, Y are in pixel indices, so we scale by pixel_size
        # Actually, X, Y are already in physical units if we used linspace correctly
        
        # Let's re-calculate world coordinates for each pixel
        # This is O(H*W), so we vectorize
        # Pixel positions in world space
        # P_world = center + X[..., None] * u + Y[..., None] * v
        # But X, Y are 2D arrays. We need to broadcast.
        
        # Reshape for broadcasting
        X_3d = X[:, :, np.newaxis] # (H, W, 1)
        Y_3d = Y[:, :, np.newaxis] # (H, W, 1)
        
        # World coordinates of detector pixels
        # Note: This assumes the detector is flat and centered at 'center'
        # The 'z' coordinate in the original note was 'center_offset_z', which implies a specific setup.
        # For arbitrary geometry, we use the normal and center.
        
        # Let's stick to the note's step 1c: "Set z coordinate to center_offset_z (sid)"
        # This implies a specific coordinate system where the detector is at z = SID.
        # For a general arbitrary geometry, we must construct the detector plane from center and normal.
        
        # Re-interpreting Step 1 for arbitrary geometry:
        # The note's step 1 is for a specific DRR setup. For arbitrary geometry, we generalize.
        # We create a grid of points on the detector plane.
        
        # Let's assume the detector is a plane passing through 'center' with normal 'normal'.
        # We need to map 2D pixel indices to 3D world coordinates.
        
        # Create a basis for the detector plane
        # u and v are orthogonal vectors in the plane
        u = np.cross(normal, np.array([0, 0, 1]))
        if np.linalg.norm(u) < 1e-8:
            u = np.cross(normal, np.array([1, 0, 0]))
        u = u / np.linalg.norm(u)
        v = np.cross(u, normal)
        
        # Generate pixel centers in world coordinates
        # x_range = [-W/2 * pixel_size, W/2 * pixel_size]
        # y_range = [-H/2 * pixel_size, H/2 * pixel_size]
        
        x_range = np.linspace(-det_res[1]/2 * pixel_size, det_res[1]/2 * pixel_size, det_res[1])
        y_range = np.linspace(-det_res[0]/2 * pixel_size, det_res[0]/2 * pixel_size, det_res[0])
        X_grid, Y_grid = np.meshgrid(x_range, y_range, indexing='ij')
        
        # World coordinates
        # P = center + X_grid * u + Y_grid * v
        # This is a (H, W, 3) array
        detector_pixels = center[np.newaxis, np.newaxis, :] + X_grid[:, :, np.newaxis] * u[np.newaxis, np.newaxis, :] + Y_grid[:, :, np.newaxis] * v[np.newaxis, np.newaxis, :]
        
        # 2. Compute ray directions
        # ray_dir = detector_pixel - source
        ray_dirs = detector_pixels - s[np.newaxis, np.newaxis, :]
        ray_dirs = ray_dirs / np.linalg.norm(ray_dirs, axis=2, keepdims=True)
        
        # 3. Normalize ray direction (already done above)
        
        # 4. Project
        # For each ray, integrate through the volume
        # This is the core of the projector (Siddon, etc.)
        # We will use a placeholder for the integration step
        projection = np.zeros((det_res[0], det_res[1]))
        
        for y in range(det_res[0]):
            for x in range(det_res[1]):
                origin = s
                direction = ray_dirs[y, x]
                # Integrate through volume
                # This is a complex step, usually implemented in CUDA/C++
                # Here we assume a simple ray-marching or analytical intersection
                integral = integrate_ray(volume, origin, direction)
                projection[y, x] = integral
                
        projections.append(projection)
        
    return projections

def integrate_ray(volume, origin, direction):
    """
    Integrate ray through volume using Siddon's method.
    """
    # This is a placeholder for the actual Siddon/Joseph/DD implementation
    # The actual implementation would calculate intersections with voxel boundaries
    # and sum the weighted path lengths.
    # For the purpose of this article, we assume this function exists and is correct.
    pass
```

## Subtopics

### [t-beam projectors](3067-t-beam-projectors.md)
t-beam projectors integrate the source-to-pixel segment along a ray for volume rendering or projection in arbitrary source-detector geometries. They differ from standard fixed-camera models by allowing explicit definition of every source-detector pair, supporting complex trajectories (circular, spiral, saddle) and parameterized detector surfaces while maintaining exact cell-constant integration.

### [TIGRE Geometry Setup](5605-tigre-geometry-setup.md)
Configures a TIGRE geometry object from a configuration dictionary, setting detector, voxel, and offset parameters for cone or parallel beam modes. It provides the geometry specification required by the arbitrary source-detector projector.

### [Projection Geometry to Vector Conversion](13361-projection-geometry-to-vector-conversion.md)
This method converts a dictionary-based projection geometry specification into a flat 12-element vector array for each projection angle, supporting both cone and parallel3D beam geometries. It serves as the bridge between high-level geometric parameters and the low-level vector representations required by downstream differentiable ray-casting kernels, distinguishing itself from higher-level setup utilities by producing the exact numerical arrays expected by the projection engine.

### DRR Geometry and Coordinate Transformation
DRR (Digitally Reconstructed Radiograph) geometry involves computing detector pixel positions in world coordinates and ray directions from a source to each pixel [S3, unofficial]. It is a specific application of arbitrary source-detector geometry for generating virtual X-ray images from 3D CT/MRI data [S3, unofficial].

### ConeBeamGeometry
A class defining the geometric parameters for a cone-beam CT scanner setup [S4, unofficial]. It includes attributes like `DSO`, `DSD`, `det_pixels`, `pixel_size`, and `angles` [S4, unofficial]. It is used to define the acquisition geometry for forward projection [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W \cdot L)$, where $N$ is the number of projections, $H \times W$ is the detector resolution, and $L$ is the average number of voxels intersected per ray [S1, unofficial].
*   **Memory Complexity:** $O(V + P)$, where $V$ is the volume size and $P$ is the projection data size [S1, unofficial].
*   **Performance:** GPU implementation (CUDA) is significantly faster than CPU [S1, unofficial].
*   **Strengths:** Flexibility in geometry specification, support for complex trajectories, exact integration methods [S1, unofficial].
*   **Limitations:** Computationally expensive for high-resolution volumes and large numbers of projections [S1, unofficial].
*   **When to Use:** When the imaging system has a non-standard geometry (e.g., C-arm, interventional CT) or when simulating arbitrary source-detector configurations [S1, unofficial].
*   **When Not to Use:** For standard parallel-beam CT where simpler, faster algorithms exist [S2, unofficial].

## Background
The Arbitrary Source-Detector Geometry Projector is part of a family of CT forward models that support flexible geometry specifications [S1, unofficial]. It builds on earlier work in parallel-beam and cone-beam geometries [S1, unofficial]. The Siddon method [S1, unofficial] and Joseph method [S1, unofficial] are common underlying algorithms for ray-volume intersection. The TIGRE library [S1, unofficial] provides a popular implementation of these methods.

## Sources
[S1] Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S2] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S3] 医疗影像开发实战:如何用光线投射算法快速生成DRR图像(附Python代码), https://un.csdn.net/98nhkcg5t04h, fetched 2026-10-26 (unofficial)
[S4] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
