# Synthetic Camera Trajectory Generation

Also called: Colmap Dataset Generation, Sobol Camera Sampling

## Overview
Synthetic Camera Trajectory Generation creates a sequence of camera poses for volume rendering or novel view synthesis when real-world capture data is unavailable or insufficient. It solves the problem of defining valid viewing geometries by sampling positions and orientations from a structured distribution, typically using low-discrepancy sequences like Sobol samplers to ensure uniform coverage of the scene volume. It differs from spiral or orbit generation by decoupling position and direction sampling to allow arbitrary spherical coverage, and from random pose selection by providing deterministic, space-filling trajectories rather than stochastic samples [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: Volume bounding box (AABB), target number of views ($N_{VIEWS}$), renderer resolution, focal length calculation method.
- **Output**: A list of camera poses (position, orientation quaternion, translation) and metadata compatible with standard photogrammetry formats (e.g., COLMAP).

**Data Structures**
1.  **Volume AABB**: Represented as a single `Point3D` object (id=0) containing position (center) and RGB values derived from the AABB bounds [S1, unofficial].
2.  **Camera Definition**: A single camera object (id=0) with model `SIMPLE_PINHOLE`. The focal length is computed via `renderer.colmap_focal_length()` based on the renderer resolution [S1, unofficial].
3.  **Pose List**: A sequence of entries, each containing:
    -   Image metadata (filename/path).
    -   Rotation quaternion (format: $[w, x, y, z]$).
    -   Translation vector (position).

**Steps**
1.  **Initialize Volume Metadata**: Write the world-space AABB of the volume as a single `Point3D` (id=0). The position is the center of the AABB; the RGB values are derived from the AABB bounds [S1, unofficial].
2.  **Initialize Camera Model**: Write a single camera (id=0) with model `SIMPLE_PINHOLE`. Set the resolution to the renderer's output dimensions. Calculate the focal length using the provided `renderer.colmap_focal_length()` function [S1, unofficial].
3.  **Sample Positions**: For each view index $i$ from $0$ to $N_{VIEWS}-1$:
    -   Use a Sobol sequence sampler (`samplerOut`) to generate a 3D point on a sphere centered at the volume center. This point becomes the camera position $\mathbf{p}_i$ [S1, unofficial].
4.  **Sample Directions**:
    -   Use a Sobol sequence sampler (`samplerIn`) to generate a small random offset vector $\mathbf{r}_i$.
    -   Define the target direction as the vector from $\mathbf{p}_i$ to the volume center $\mathbf{c}$.
    -   Apply the offset $\mathbf{r}_i$ to this direction to determine the final viewing direction $\mathbf{d}_i$ [S1, unofficial].
5.  **Compute Orientation**:
    -   Construct a rotation matrix or quaternion that aligns the camera's local Z-axis (or forward vector) with $\mathbf{d}_i$.
    -   Convert the orientation to a quaternion $q_i = [w, x, y, z]$.
    -   Reorder the quaternion components from the internal representation $[x, y, z, w]$ to $[w, x, y, z]$ (specifically noted as reordering from $[3,0,1,2]$ in the source implementation, implying a specific memory layout or library convention) [S1, unofficial].
6.  **Store Pose**: Store the image metadata, the reordered quaternion $q_i$, and the translation vector $\mathbf{p}_i$ for view $i$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_synthetic_trajectory(aabb_center, aabb_size, n_views, resolution, focal_length_func):
    """
    Generates synthetic camera trajectories using Sobol sampling.
    
    Args:
        aabb_center: np.array, shape (3,), center of the volume.
        aabb_size: np.array, shape (3,), dimensions of the volume.
        n_views: int, number of camera poses to generate.
        resolution: tuple, (width, height) of the rendered image.
        focal_length_func: callable, returns focal length from resolution.
        
    Returns:
        poses: list of dicts, each containing 'position', 'quaternion', 'camera_id'.
    """
    # Step 1: Initialize Volume Metadata (Conceptual, stored externally)
    # volume_metadata = {
    #     'id': 0,
    #     'position': aabb_center,
    #     'rgb': aabb_size # Simplified representation per source
    # }
    
    # Step 2: Initialize Camera Model (Conceptual, stored externally)
    # camera_id = 0
    # camera_model = 'SIMPLE_PINHOLE'
    # focal_length = focal_length_func(resolution)
    
    poses = []
    
    # Initialize Sobol samplers
    # Note: In practice, use a library like `sobol_seq` or `scipy.stats.qmc.Sobol`
    # sampler_out for positions on sphere
    # sampler_in for direction offsets
    # The source implies two separate samplers or streams.
    
    for i in range(n_views):
        # Step 3: Sample position on sphere
        # Sobol sequence generates points in [0,1]^3. Map to sphere.
        # Common mapping: Generate uniform point in cube, normalize to sphere, scale by radius.
        # Source says "sample camera position on a sphere around the volume center".
        # Let's assume a standard Sobol-to-Sphere mapping.
        raw_pos = sobol_sample_3d() # Returns [0,1]^3
        # Map to sphere surface
        # Using inverse CDF or rejection sampling is standard for Sobol on sphere.
        # For simplicity, we assume a helper `sobol_sphere_sample` exists.
        offset_from_center = sobol_sphere_sample() # Unit vector
        radius = np.linalg.norm(aabb_size) / 2.0 * 1.5 # Heuristic radius > volume
        position = aabb_center + offset_from_center * radius
        
        # Step 4: Sample direction
        # Direction towards center
        direction_to_center = aabb_center - position
        direction_to_center = direction_to_center / np.linalg.norm(direction_to_center)
        
        # Small random offset using Sobol
        raw_offset = sobol_sample_3d()
        # Map to small sphere or cube for offset
        offset = (raw_offset - 0.5) * 0.1 # Small perturbation
        
        # Apply offset
        final_direction = direction_to_center + offset
        final_direction = final_direction / np.linalg.norm(final_direction)
        
        # Step 5: Compute Quaternion
        # Construct rotation matrix aligning camera forward (e.g., -Z) with final_direction
        # Assuming camera looks down -Z axis in local space
        forward = final_direction
        # Up vector approximation (avoid gimbal lock if forward is parallel to world up)
        world_up = np.array([0, 1, 0])
        if np.abs(np.dot(forward, world_up)) > 0.999:
            world_up = np.array([1, 0, 0])
        
        right = np.cross(forward, world_up)
        right = right / np.linalg.norm(right)
        up = np.cross(right, forward)
        
        # Rotation matrix from world to camera (columns are right, up, -forward)
        # Or camera to world? COLMAP usually stores R as world-to-camera or camera-to-world?
        # Source says "quaternion rotation". COLMAP format typically stores rotation vector or quaternion.
        # Let's construct the rotation matrix that rotates the camera local frame to world.
        # R_cam_to_world = [right, up, -forward]
        R = np.column_stack((right, up, -forward))
        
        # Convert to quaternion [w, x, y, z]
        quat = rotation_matrix_to_quaternion(R)
        
        # Step 6: Store Pose
        # Source mentions reordering from [3,0,1,2]. This likely refers to 
        # converting from a library's internal [x,y,z,w] or similar to [w,x,y,z].
        # Assuming quat is already [w,x,y,z] from standard conversion.
        # If the source implies a specific internal format, we note the reordering.
        # Here we assume standard [w,x,y,z].
        
        poses.append({
            'id': i,
            'position': position,
            'quaternion': quat, # [w, x, y, z]
            'camera_id': 0
        })
        
    return poses

def sobol_sample_3d():
    """Placeholder for Sobol sequence generation in [0,1]^3."""
    # Implementation depends on specific Sobol library.
    # Returns a 3D vector with components in [0,1].
    pass

def sobol_sphere_sample():
    """Placeholder for mapping Sobol sequence to unit sphere."""
    # Standard method: Generate 3D Gaussian or normalize uniform cube.
    # For low-discrepancy, often rejection sampling or specific mapping is used.
    pass

def rotation_matrix_to_quaternion(R):
    """Converts 3x3 rotation matrix to [w, x, y, z] quaternion."""
    trace = np.trace(R)
    if trace > 0:
        S = np.sqrt(trace + 1.0) * 2
        w = 0.25 * S
        x = (R[2, 1] - R[1, 2]) / S
        y = (R[0, 2] - R[2, 0]) / S
        z = (R[1, 0] - R[0, 1]) / S
    elif (R[0, 0] > R[1, 1]) and (R[0, 0] > R[2, 2]):
        S = np.sqrt(1.0 + R[0, 0] - R[1, 1] - R[2, 2]) * 2
        w = (R[2, 1] - R[1, 2]) / S
        x = 0.25 * S
        y = (R[0, 1] + R[1, 0]) / S
        z = (R[0, 2] + R[2, 0]) / S
    elif R[1, 1] > R[2, 2]:
        S = np.sqrt(1.0 + R[1, 1] - R[0, 0] - R[2, 2]) * 2
        w = (R[0, 2] - R[2, 0]) / S
        x = (R[0, 1] + R[1, 0]) / S
        y = 0.25 * S
        z = (R[1, 2] + R[2, 1]) / S
    else:
        S = np.sqrt(1.0 + R[2, 2] - R[0, 0] - R[1, 1]) * 2
        w = (R[1, 0] - R[0, 1]) / S
        x = (R[0, 2] + R[2, 0]) / S
        y = (R[1, 2] + R[2, 1]) / S
        z = 0.25 * S
    return np.array([w, x, y, z])
```

**Parameters**
-   $N_{VIEWS}$: Number of views. Not given in the sources.
-   Radius of sampling sphere: Not given in the sources. The source implies "around the volume center" but does not specify the radius relative to the AABB.
-   Offset magnitude for direction: Not given in the sources. The source mentions "small random offset" but does not quantify it.

**Edge Cases and Pitfalls**
-   **Precision**: Sobol sequence generators must be initialized with the correct dimension and seed to avoid repetition. [G] Ensure the Sobol sampler handles the mapping to the sphere correctly to avoid clustering at poles if using simple normalization of uniform samples.
-   **Boundaries**: The camera position must be outside the volume AABB to avoid occlusion issues, unless the volume is transparent and the renderer supports interior views. The source does not specify a minimum distance.
-   **Degenerate Input**: If the volume center is not well-defined (e.g., empty AABB), the trajectory generation fails. [G] Check for valid AABB bounds before sampling.

## Subtopics

### [Random Pose Selection](8187-random-pose-selection.md)
Random Pose Selection generates intermediate camera poses by interpolating between two randomly chosen training poses, ensuring that rendered views remain within the observed regions of the training data. This approach prevents rendering failures on unobserved parts of the scene that occur when attempting to render frames significantly outside the training view coverage. It differs from deterministic trajectory generation by relying on stochastic sampling constrained by proximity thresholds, and from simple linear interpolation by using rotation-aware interpolation for the orientation component. See [Random Pose Selection](8187-random-pose-selection.md).

### Inpainting Trajectory Sampling
Inpainting Trajectory Sampling generates camera positions for inpainting tasks by defining a closed curve on the horizontal plane based on depth values at zero elevation angle from a single panoramic depth map. It applies 1D Gaussian filtering to smooth the distance values, creating a closed curve $C$ that defines the maximum movement space. It then progressively samples on shrunk curves $\{C_i\}$ scaled by factors $\{s_i\}$. Specifically, it uses 4 shrunk curves with scale factors 0.15, 0.3, 0.45, and 0.6, uniformly sampling 8 positions on each, resulting in 32 total positions. See [Inpainting Trajectory Sampling](8187-random-pose-selection.md) (Note: No specific article exists for this variant in the provided tree, so it is listed here as a minor variant without a link).

## Operating characteristics
-   **Time Complexity**: $O(N_{VIEWS})$ for sampling and pose generation, assuming constant-time Sobol sampling and quaternion conversion.
-   **Memory Complexity**: $O(N_{VIEWS})$ to store the pose list.
-   **Performance**: The method is computationally lightweight compared to optimization-based trajectory planning. It relies on fast low-discrepancy sequence generation.
-   **Strengths**: Provides uniform coverage of the spherical viewing space. Deterministic and reproducible. Simple to implement.
-   **Limitations**: Does not account for scene-specific occlusions or lighting conditions. The fixed radius and offset parameters may not be optimal for all scene scales.
-   **When to use**: When generating synthetic datasets for training neural radiance fields or volume renderers where uniform sampling is desired.
-   **When not to use**: When specific viewpoints are required (e.g., looking at a specific object) or when the scene geometry is highly irregular and requires adaptive sampling.

## Background
The method originates from the need to generate training data for volume rendering and neural radiance fields when real-world captures are unavailable. The specific implementation using Sobol sequences and COLMAP format is documented in the `volren` repository [S1, unofficial]. The inpainting variant is described in the context of panoramic neural radiance fields from single panoramas [S2].

## Sources
[S1] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S2] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
