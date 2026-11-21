# Arbitrary Source-Detector Geometry Projector

Also called: Arbitrary Source-Detector Geometry, Axial cone-beam, Cone-Beam Geometry Projector, Flexible cone-beam, Parallel Beam Geometry, Parallel-Beam Geometry Projector

## Overview
The Arbitrary Source-Detector Geometry Projector is a flexible CT forward/backward projection model that allows the user to define the 3D location and orientation of every individual source-detector pair, overriding standard geometric assumptions [S1, unofficial]. It supports planar or curved detectors (axial) and arbitrary source/detector placement, enabling complex trajectories and non-equispaced projection angles [S1, unofficial]. Unlike standard Radon models which assume parallel rays, this geometry handles point-source cone-beam and fan-beam configurations where projection lines are non-parallel [S3, unofficial]. It builds on [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md) by mapping world-space voxel intersections to arbitrary detector coordinates [G].

## How it works
**Inputs:**
* Volume data $V$ (3D array of voxel values, units $\text{mm}^{-1}$) [S1, unofficial].
* Geometry specification:
  * Source positions $\mathbf{s}_i$ for each projection angle $i$ [G].
  * Detector plane parameters: center, normal vector, pixel size, resolution [G].
  * Projection angles $\Theta = \{\theta_1, \dots, \theta_N\}$ [S4, unofficial].
* Optional: Source-to-Isocenter Distance (DSO), Source-to-Detector Distance (DSD) [S4, unofficial].

**Outputs:**
* Projection data $P$ (2D array per angle, representing line integrals or attenuated intensity) [G].

**Data Structures:**
* **Detector Grid:** A 2D array of pixel centers in world coordinates [G].
* **Ray Representation:** Origin $\mathbf{o} = \mathbf{s}_i$, Direction $\mathbf{d} = \text{normalize}(\mathbf{p}_{\text{pixel}} - \mathbf{s}_i)$ [S3, unofficial].

**Steps:**
1. **Compute Detector Pixel Positions in World Coordinates:**
   a. Calculate $y_{\text{coords}}$ and $x_{\text{coords}}$ using linear spacing based on `detector_size` and `pixel_size` [S3, unofficial].
   b. Create a meshgrid with 'ij' indexing to form 2D coordinate arrays $X, Y$ [S3, unofficial].
   c. Set the $z$ coordinate to `center_offset_z` (SID) [S3, unofficial].
   d. Stack into a $(H, W, 3)$ array with order $[Z, Y, X]$ [S3, unofficial].
2. **Compute Ray Directions:**
   For each pixel $p$ at position $\mathbf{p}_{\text{world}}$ and source $\mathbf{s}$:
   $$ \mathbf{d} = \mathbf{p}_{\text{world}} - \mathbf{s} $$
   $$ \mathbf{d} = \frac{\mathbf{d}}{\|\mathbf{d}\|} $$
   [S3, unofficial]
3. **Ray Casting / Integration:**
   Trace rays from $\mathbf{s}$ along $\mathbf{d}$ through the volume $V$ [G].
   * Use a projector model such as Siddon [S1, unofficial], Joseph [S1, unofficial], or Distance Driven (DD) [S1, unofficial].
   * Accumulate the integral of voxel values along the ray segment within the volume bounds [G].
4. **Handle Geometry Variants:**
   * **Parallel Beam:** Rays are parallel; source is at infinity. [S2, unofficial]
   * **Cone Beam:** Point source, diverging rays. [S2, unofficial]
   * **Flexible:** Arbitrary source/detector positions. [S1, unofficial]

**Parameters:**
* `DSO` (Source-to-Isocenter Distance): Default 500 [S4, unofficial].
* `DSD` (Source-to-Detector Distance): Default 800 [S4, unofficial].
* `det_pixels`: Detector pixel count per side (assumed square). Default 256 [S4, unofficial].
* `pixel_size`: Physical size of a detector pixel. Default 1.0 [S4, unofficial].
* `angles`: Array of rotation angles from $0$ to $2\pi$. Default 180 angles [S4, unofficial].

**Edge Cases and Pitfalls:**
* **Non-Parallel Rays:** Standard Radon models assume parallel rays. Cone-beam and fan-beam geometries use a point source, making projection lines non-parallel [S3, unofficial].
* **Precision:** Floating-point errors in ray-volume intersection calculations can cause artifacts. Use robust intersection algorithms (e.g., Siddon) [S1, unofficial].
* **Boundary Conditions:** Rays may miss the volume entirely [G]. Handle these by setting projection value to 0 or background intensity [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def cross(a, b): return [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]
def norm(v): return (v[0]**2 + v[1]**2 + v[2]**2)**0.5
def normalize(v): n = norm(v); return [v[0]/n, v[1]/n, v[2]/n]

def arbitrary_projector(volume, sources, detector_centers, detector_normals, pixel_size, det_res):
    """
    Forward projection for arbitrary source-detector geometry.
    
    Args:
        volume: 3D list of lists of lists (H, W, D) of voxel values.
        sources: List of 3D source positions (N, 3).
        detector_centers: List of detector center positions (N, 3).
        detector_normals: List of detector normal vectors (N, 3).
        pixel_size: Physical size of a detector pixel.
        det_res: Tuple (height, width) of detector resolution.
        
    Returns:
        projections: List of 2D lists (N, H, W).
    """
    projections = []
    
    for i in range(len(sources)):
        s = sources[i]
        center = detector_centers[i]
        normal = detector_normals[i]
        
        # 1. Compute detector pixel positions in world coordinates
        y_coords = [j * pixel_size for j in range(-det_res[0]//2, det_res[0]//2 + 1)]
        x_coords = [j * pixel_size for j in range(-det_res[1]//2, det_res[1]//2 + 1)]
        
        # Construct basis vectors for the detector plane
        arbitrary_vec = [0, 0, 1]
        if abs(dot(normal, arbitrary_vec)) > 0.9:
            arbitrary_vec = [1, 0, 0]
        u = normalize(cross(normal, arbitrary_vec))
        v = cross(u, normal)
        
        # 2. Compute ray directions
        ray_dirs = []
        for y_idx, y_val in enumerate(y_coords):
            row_dirs = []
            for x_idx, x_val in enumerate(x_coords):
                pixel_pos = [center[0] + x_val*u[0] + y_val*v[0],
                             center[1] + x_val*u[1] + y_val*v[1],
                             center[2] + x_val*u[2] + y_val*v[2]]
                ray_dir = normalize([pixel_pos[k] - s[k] for k in range(3)])
                row_dirs.append(ray_dir)
            ray_dirs.append(row_dirs)
            
        # 3. Project
        projection = [[0.0 for _ in range(det_res[1])] for _ in range(det_res[0])]
        for y in range(det_res[0]):
            for x in range(det_res[1]):
                integral = integrate_ray(volume, s, ray_dirs[y][x])
                projection[y][x] = integral
        projections.append(projection)
    return projections

def integrate_ray(volume, origin, direction):
    # The sources do not specify the exact integration method (Siddon, Joseph, DD).
    # This placeholder would calculate intersections with voxel boundaries 
    # and sum the weighted path lengths or sample along the ray.
    return 0.0
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

## Background
The Arbitrary Source-Detector Geometry Projector is part of a family of CT forward models that support flexible geometry specifications [S1, unofficial]. It builds on earlier work in parallel-beam and cone-beam geometries [S1, unofficial]. The Siddon method [S1, unofficial] and Joseph method [S1, unofficial] are common underlying algorithms for ray-volume intersection.

## Sources
[S1] Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S2] Differentiable Forward Projector for X-ray Computed ..., https://openreview.net/forum?id=dX8khDFGHv, fetched 2026-11-06 (unofficial)
[S3] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S4] 医疗影像开发实战:如何用光线投射算法快速生成DRR图像(附Python代码), https://un.csdn.net/98nhkcg5t04h, fetched 2026-10-26 (unofficial)
[S5] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
