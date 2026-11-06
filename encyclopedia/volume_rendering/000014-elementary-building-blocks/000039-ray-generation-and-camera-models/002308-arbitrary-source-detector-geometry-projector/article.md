# Arbitrary Source-Detector Geometry Projector

Also called: Arbitrary Source-Detector Geometry, Axial cone-beam, Cone-Beam Geometry Projector, Flexible cone-beam, Parallel-Beam Geometry Projector

## Overview
The Arbitrary Source-Detector Geometry Projector is a CT geometry type that allows the user to define the 3D location and orientation of every individual source-detector pair, overriding standard geometric assumptions [S1, unofficial]. It supports flexible specification including arbitrary 3D detector shifts and non-equispaced projection angles, covering planar or curved detectors (axial) and arbitrary source/detector placement (flexible) [S1, unofficial]. It differs from fixed-camera models by explicitly defining every source-detector pair, supporting complex trajectories and parameterized detector surfaces while maintaining exact cell-constant integration [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: 
  - Source position $\mathbf{s} \in \mathbb{R}^3$ for each projection angle.
  - Detector geometry: size (width $W$, height $H$), pixel size (pitch $p_x, p_y$), center offset (SID, $z_{center}$), and orientation (rotation matrix or normal vector).
  - Volume data: 3D voxel grid with physical dimensions.
- **Outputs**: 
  - Projection image (DRR) of size $H \times W$, where each pixel value is the line integral (or attenuated sum) along the ray from source to detector pixel.

**Data Structures**
- **Detector Grid**: A 2D array of detector pixels. Each pixel is represented by its 3D world coordinates.
- **Ray Data**: For each pixel, store ray origin $\mathbf{o}$ (source position) and direction $\mathbf{d}$ (normalized vector from source to detector pixel).

**Steps**
1. **Compute Detector Pixel Positions in World Coordinates**:
   a. Calculate $y\_coords$ and $x\_coords$ using `linspace` based on `detector_size` and `pixel_size`.
   b. Create a meshgrid with 'ij' indexing to form 2D arrays $X$ and $Y$.
   c. Set the $z$ coordinate to `center_offset_z` (SID).
   d. Stack into a $(H, W, 3)$ array with order $[Z, Y, X]$. This gives the detector plane in a local coordinate system aligned with the detector normal.
   e. Apply rotation/translation to transform these local coordinates to world coordinates based on the arbitrary source-detector orientation.

2. **Compute Ray Directions**:
   For each pixel $i,j$:
   $$ \mathbf{d}_{i,j} = \text{detector\_pixel}_{i,j} - \mathbf{s} $$
   where $\mathbf{s}$ is the source position for the current projection.

3. **Normalize Ray Directions**:
   $$ \hat{\mathbf{d}}_{i,j} = \frac{\mathbf{d}_{i,j}}{\|\mathbf{d}_{i,j}\|} $$

4. **Ray Marching/Integration**:
   For each pixel, march along $\hat{\mathbf{d}}_{i,j}$ from the source through the volume, accumulating attenuation or density values using a projector model (e.g., Siddon, Distance Driven) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_rays(source_pos, detector_size, pixel_size, center_offset_z, rotation_matrix):
    """
    Compute ray origins and directions for arbitrary source-detector geometry.
    
    Args:
        source_pos: (3,) array, source position in world coordinates.
        detector_size: (2,) array, [width, height] in world units.
        pixel_size: (2,) array, [pixel_width, pixel_height] in world units.
        center_offset_z: float, distance from source to detector center (SID).
        rotation_matrix: (3, 3) array, rotation matrix for detector orientation.
        
    Returns:
        origins: (H, W, 3) array, ray origins (same as source_pos for each pixel).
        directions: (H, W, 3) array, normalized ray directions.
    """
    # 1. Compute detector pixel positions in local coordinates
    h, w = int(detector_size[1] / pixel_size[1]), int(detector_size[0] / pixel_size[0])
    
    # Create coordinate grids
    y_coords = np.linspace(-detector_size[1]/2, detector_size[1]/2, h)
    x_coords = np.linspace(-detector_size[0]/2, detector_size[0]/2, w)
    X, Y = np.meshgrid(x_coords, y_coords, indexing='ij')
    
    # Z coordinate is the SID (distance from source to detector center)
    Z = np.full((h, w), center_offset_z)
    
    # Stack into (H, W, 3) array with order [Z, Y, X] in local detector frame
    # Note: In local frame, detector is at z = SID, centered at (0,0,SID)
    local_detector_pixels = np.stack([Z, Y, X], axis=-1)  # Shape (H, W, 3)
    
    # 2. Transform to world coordinates
    # Apply rotation and translation. Assuming local origin is at detector center.
    # World position = Rotation * LocalPosition + Translation
    # Translation is such that the center of the detector is at distance SID from source along the optical axis.
    # For simplicity, assume the rotation matrix defines the orientation of the detector plane.
    # The center of the detector in world coordinates is source_pos + rotation_matrix @ [0, 0, SID]^T
    # But usually, the detector plane is defined relative to the source.
    # Let's assume the local coordinate system has Z axis pointing from source to detector center.
    # Then, world_detector_center = source_pos + rotation_matrix @ [0, 0, SID]^T
    # And each local pixel is rotated and translated to world.
    
    # Reshape for matrix multiplication
    local_pixels_flat = local_detector_pixels.reshape(-1, 3)
    
    # Rotate local pixels to world orientation
    # Note: The local Z axis is the optical axis. The rotation matrix aligns the local frame to the world frame.
    # If the local frame is defined such that Z is the optical axis, then:
    world_pixels_flat = (rotation_matrix @ local_pixels_flat.T).T
    
    # Translate so that the center of the detector is at the correct position relative to the source.
    # The center of the local detector is [0, 0, SID].
    # The center of the world detector should be source_pos + rotation_matrix @ [0, 0, SID]^T
    # But our local_pixels already have Z=SID. So we just need to rotate them.
    # However, the origin of the local coordinate system is at the source? No, it's at the detector center?
    # Let's re-read the source: "Set z coordinate to center_offset_z (sid)".
    # This implies the local coordinate system has its origin at the source? No, if Z=SID, then the detector is at Z=SID.
    # So the local origin is at the source.
    # Therefore, world_detector_pixel = source_pos + rotation_matrix @ local_detector_pixel
    
    world_detector_pixels = source_pos + (rotation_matrix @ local_pixels_flat.T).T
    world_detector_pixels = world_detector_pixels.reshape(h, w, 3)
    
    # 3. Compute ray directions
    # Ray direction is from source to detector pixel
    ray_dirs = world_detector_pixels - source_pos
    
    # 4. Normalize ray directions
    norms = np.linalg.norm(ray_dirs, axis=-1, keepdims=True)
    # Avoid division by zero
    norms = np.maximum(norms, 1e-8)
    normalized_dirs = ray_dirs / norms
    
    return np.full((h, w, 3), source_pos), normalized_dirs
```

**Parameters**
- `detector_size`: Typical values depend on the application (e.g., 512x512 pixels).
- `pixel_size`: Depends on the physical detector or desired resolution (e.g., 0.5 mm).
- `center_offset_z` (SID): Typical values range from 500 mm to 1500 mm in CT scanners.
- `rotation_matrix`: Defined by the specific scanner geometry or trajectory.

**Edge Cases and Pitfalls**
- **Precision**: Use double precision for ray direction normalization to avoid artifacts in long rays.
- **Boundaries**: Ensure detector pixels are correctly mapped to the volume bounds. Rays that miss the volume should be handled (e.g., zero contribution).
- **Degenerate Input**: If the source is on the detector plane, the ray direction is undefined. Guard against this by checking the distance between source and detector.

## Subtopics

### [t-beam projectors](3067-t-beam-projectors.md)
t-beam projectors integrate the source-to-pixel segment along a ray for volume rendering or projection in arbitrary source-detector geometries. They differ from standard fixed-camera models by allowing explicit definition of every source-detector pair, supporting complex trajectories (circular, spiral, saddle) and parameterized detector surfaces while maintaining exact cell-constant integration.

### [TIGRE Geometry Setup](5605-tigre-geometry-setup.md)
Configures a TIGRE geometry object from a configuration dictionary, setting detector, voxel, and offset parameters for cone or parallel beam modes. It serves as the foundational step for defining the arbitrary source-detector geometry required by the parent projector, ensuring that ray origins and detector coordinates are mathematically consistent with the chosen scanner mode.

### DRR Geometry and Coordinate Transformation
DRR (Digitally Reconstructed Radiograph) generation requires transforming 3D volume data into 2D X-ray images. This involves defining the source position, detector plane orientation, and pixel grid, then computing the ray paths through the volume to simulate the attenuation of X-rays [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N_{pixels} \times N_{samples})$, where $N_{pixels}$ is the number of detector pixels and $N_{samples}$ is the number of samples per ray.
- **Memory Complexity**: $O(N_{pixels} \times 3)$ for storing ray origins and directions, plus $O(N_{voxels})$ for the volume data.
- **Measured Performance**: Performance depends on the projector model (Siddon, Distance Driven) and hardware (CPU vs. GPU). GPU implementations using CUDA can achieve real-time or near-real-time performance for interactive applications [S1, unofficial][S2, unofficial].
- **Strengths**: Flexibility in defining arbitrary geometries, support for complex trajectories, and compatibility with various projector models.
- **Limitations**: Computational cost can be high for large volumes and high-resolution detectors. Precision issues may arise with long rays or small pixel sizes.
- **When to Use**: When the scanner geometry is non-standard, or when simulating arbitrary source-detector configurations for research or planning purposes.
- **When Not to Use**: For standard CT scanners with fixed geometries, simpler camera models may be more efficient.

## Background
The Arbitrary Source-Detector Geometry Projector is part of a software package that implements three different 3D XCT geometry types: parallel-beam, axial cone-beam, and flexible cone-beam [S1, unofficial]. The flexible cone-beam geometry allows users to place X-ray sources and detectors at any position and orientation in space, covering most XCT applications [S1, unofficial]. The implementation uses both CPU (C/C++) and GPU (CUDA) parallelization, with forward and back projection implementations available for the three CT geometries using both the Siddon and SF projector models [S1, unofficial]. The DRR technique is crucial in medical imaging for comparing 3D CT/MRI data with 2D X-ray images, especially in radiotherapy planning and image-guided surgery [S2, unofficial].

## Sources
[S1] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S2] 医疗影像开发实战:如何用光线投射算法快速生成DRR图像(附Python代码), https://un.csdn.net/98nhkcg5t04h, fetched 2026-10-26 (unofficial)
