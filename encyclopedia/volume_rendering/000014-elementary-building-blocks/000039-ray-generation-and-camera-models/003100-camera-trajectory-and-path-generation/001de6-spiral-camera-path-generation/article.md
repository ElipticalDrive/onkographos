# Spiral Camera Path Generation

Also called: Novel View Synthesis Path, load_tankstemple_data spiral generation

## Overview
Spiral Camera Path Generation creates a sequence of camera poses along a helical trajectory around a scene's centroid for novel view synthesis [S1, S3, S4, unofficial]. It solves the problem of generating smooth, continuous fly-through or orbiting viewpoints that cover 360° horizontal rotation while introducing vertical variation, which prevents the "flat" appearance of pure circular orbits [S3, S6, unofficial]. It differs from [Circular Camera Path Generation](2734-circular-camera-path-generation.md) by adding a vertical oscillation component (z-rate), making it a non-planar helix rather than a degenerate planar circle [S1, unofficial]. It differs from [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md) by assuming a central focal point and isotropic or axis-aligned radii rather than following a specific linear or curved structural axis [S1, S3, unofficial].

## How it works
The method generates a sequence of $4 \times 4$ camera-to-world transformation matrices. The core mechanism involves analyzing the input training poses to determine the scene's spatial extent and orientation, then parameterizing a 3D spiral path using radius, rotation count, and vertical oscillation rate.

### Inputs and Outputs
*   **Inputs**:
    *   `poses`: Array of $N$ training camera poses, each a $4 \times 4$ matrix (camera-to-world) [S1, S3, S4, unofficial].
    *   `n_poses` (or `N`): Number of output camera poses to generate [S1, S3, S4, unofficial].
    *   `n_rots` (or `rots`): Number of full horizontal rotations ($2\pi$) to complete [S1, S3, S4, unofficial].
    *   `z_rate`: Vertical oscillation rate parameter [S1, S3, S4, unofficial].
    *   `rads_scale` (optional): Scaling factor for the computed radii [S3, unofficial].
    *   `focus_depth` (optional): Depth of the focal plane for forward-facing scenes [S2, unofficial].
    *   `pitch_deg` (optional): Pitch angle for the camera up-vector [S6, unofficial].
    *   `shift` (optional): Centroid shift vector $[dx, dy, dz]$ [S6, unofficial].
*   **Outputs**:
    *   `render_poses`: List or array of $M$ camera poses, where $M = n\_poses$. Each pose is a $4 \times 4$ matrix [S1, S4, unofficial].

### Data Structures
*   **Pose Matrix**: $4 \times 4$ matrix. The upper-left $3 \times 3$ submatrix contains the orthonormal basis vectors $[\mathbf{x}, \mathbf{y}, \mathbf{z}]$ (right, up, forward/backward) of the camera in world coordinates. The fourth column contains the translation vector $\mathbf{c}$ (camera center). The fourth row is typically $[0, 0, 0, 1]$ [S1, S4, unofficial].
*   **Radii Vector**: $\mathbf{rads} = [r_x, r_y, r_z]$, a 3-element array defining the spiral's extent along each world axis [S1, S3, S4, unofficial].

### Algorithm Steps

1.  **Scene Analysis**:
    *   Compute the scene centroid $\mathbf{c}_{avg}$ as the mean of all training camera positions:
        $$ \mathbf{c}_{avg} = \frac{1}{N} \sum_{i=0}^{N-1} \mathbf{poses}_i[:3, 3] $$
        [S3, S6, unofficial].
    *   Compute the average up vector $\mathbf{u}_{avg}$ from training poses:
        $$ \mathbf{u}_{avg} = \frac{1}{N} \sum_{i=0}^{N-1} \mathbf{poses}_i[:3, 1] $$
        [S1, S3, S4, unofficial].
    *   Compute the radii $\mathbf{rads}$. Two common methods exist:
        *   **Method A (Percentile)**: Calculate the 90th percentile of the absolute camera positions relative to the centroid along each axis, then scale by `rads_scale` [S3, unofficial].
        *   **Method B (Default)**: If not provided, use a default radius derived from the mean distance of cameras to the centroid, scaled by `scale_r` (default 1.0) [S6, unofficial].
        *   Ensure $\mathbf{rads}$ has 3 elements. If only 2 are provided, append $1.0$ for the z-component [S1, S4, unofficial].

2.  **Path Parameterization**:
    *   Define the number of frames $M = n\_poses$ [S3, S4, unofficial].
    *   Generate a sequence of angles $\theta$ from $0$ to $2\pi \cdot n\_rots$ (exclusive of the endpoint to avoid duplication):
        $$ \theta_j = \frac{j}{M} \cdot 2\pi \cdot n\_rots, \quad j = 0, \dots, M-1 $$
        [S1, S3, S4, unofficial].
    *   Alternatively, parameterize by time $t \in [0, 1)$:
        $$ t_j = \frac{j}{M} $$
        [S5, unofficial].

3.  **Pose Construction**:
    For each angle $\theta_j$ (or time $t_j$):

    *   **Compute Camera Center $\mathbf{c}$**:
        Calculate the offset vector in the local spiral frame:
        $$ \mathbf{c}_{offset} = \begin{bmatrix} r_x \cos(\theta_j) \\ -r_y \sin(\theta_j) \\ -r_z \sin(\theta_j \cdot z\_rate) \end{bmatrix} $$
        [S1, S3, S4, unofficial].
        Transform this offset to world space using the average pose's rotation matrix $\mathbf{R}_{avg} = \mathbf{c}_{avg\_pose}[:3, :3]$:
        $$ \mathbf{c} = \mathbf{c}_{avg} + \mathbf{R}_{avg} \mathbf{c}_{offset} $$
        [S1, S3, S4, unofficial].
        *Note: Some implementations use a fixed world frame for the spiral (e.g., $x = r \cos \theta, y = -r \sin \theta, z = -r \sin(\theta \cdot z\_rate)$) relative to the centroid, effectively assuming $\mathbf{R}_{avg}$ is identity or aligning the spiral axes with world axes [S2, S6, unofficial]. The most general form uses $\mathbf{R}_{avg}$ to align the spiral with the scene's principal axes [S1, S3, S4, unofficial].*

    *   **Compute Viewing Direction $\mathbf{z}$**:
        *   **Forward-Facing (LLFF) Style**: Look towards a focal plane at depth $d$ along the negative z-axis of the average pose:
            $$ \mathbf{z} = \text{normalize}(\mathbf{c}_{avg} + \mathbf{R}_{avg} \begin{bmatrix} 0 \\ 0 \\ -focal \end{bmatrix} - \mathbf{c}) $$
            [S2, S4, unofficial].
        *   **General/Orbiting Style**: Look towards the scene centroid:
            $$ \mathbf{z} = \text{normalize}(\mathbf{c}_{avg} - \mathbf{c}) $$
            [S6, unofficial].

    *   **Compute Up Vector $\mathbf{u}$**:
        *   Use the average up vector $\mathbf{u}_{avg}$ [S1, S3, S4, unofficial].
        *   Apply pitch adjustment if `pitch_deg` is specified:
            $$ \text{target\_y} = r_{circle} \tan(\text{pitch\_deg} \cdot \frac{\pi}{180}) $$
            Adjust the look-at target's y-component accordingly [S6, unofficial].
        *   Handle up-vector flipping if `flip_up_vec` is True [S6, unofficial].

    *   **Compute Right Vector $\mathbf{x}$**:
        $$ \mathbf{x} = \text{normalize}(\mathbf{u}_{avg} \times \mathbf{z}) $$
        [S2, S6, unofficial].
        *Note: Some implementations use a fixed world up vector $[0, 1, 0]$ for the cross product if the scene is forward-facing [S2, unofficial].*

    *   **Compute True Up Vector $\mathbf{y}$**:
        $$ \mathbf{y} = \mathbf{z} \times \mathbf{x} $$
        [S2, S6, unofficial].

    *   **Construct Pose Matrix**:
        $$ \mathbf{P}_j = \begin{bmatrix} \mathbf{x} & \mathbf{y} & \mathbf{z} & \mathbf{c} \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
        [S1, S2, S4, S6, unofficial].

4.  **Validation**:
    *   Ensure $\mathbf{x}, \mathbf{y}, \mathbf{z}$ are orthonormal. Re-normalize if necessary [S6, unofficial].
    *   Check for singularities if $\mathbf{z}$ is parallel to the up vector (gimbal lock). In such cases, fallback to a fixed up vector or adjust the target [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_spiral_path(poses, n_poses, n_rots, z_rate=0.5, rads_scale=1.0, 
                         focus_depth=None, pitch_deg=0.0, shift=np.zeros(3)):
    """
    Generate a spiral camera path.
    
    Args:
        poses: np.ndarray of shape (N, 4, 4), training camera poses (c2w).
        n_poses: int, number of output poses.
        n_rots: int, number of full rotations.
        z_rate: float, vertical oscillation rate.
        rads_scale: float, scale factor for radii.
        focus_depth: float, focal depth for forward-facing scenes.
        pitch_deg: float, pitch angle in degrees.
        shift: np.ndarray of shape (3,), centroid shift.
        
    Returns:
        np.ndarray of shape (n_poses, 4, 4), generated camera poses.
    """
    # 1. Scene Analysis
    # Compute centroid
    centroids = poses[:, :3, 3]
    c_avg = np.mean(centroids, axis=0) + shift
    
    # Compute average rotation and up vector
    R_avg = np.mean(poses[:, :3, :3], axis=0)
    # Normalize R_avg to ensure it's a valid rotation matrix (simple approximation)
    # In practice, use SVD or quaternion averaging for robustness [G]
    U, _, Vt = np.linalg.svd(R_avg)
    R_avg = U @ Vt
    
    u_avg = np.mean(poses[:, :3, 1], axis=0)
    
    # Compute radii
    # Method: 90th percentile of distances from centroid
    dists = centroids - c_avg
    rads = np.percentile(np.abs(dists), 90, axis=0) * rads_scale
    
    # Ensure 3 radii
    if len(rads) < 3:
        rads = np.append(rads, 1.0)
        
    # 2. Path Parameterization
    thetas = np.linspace(0, 2 * np.pi * n_rots, n_poses, endpoint=False)
    
    render_poses = []
    
    for theta in thetas:
        # 3. Pose Construction
        
        # Compute camera center
        # Offset in local spiral frame
        c_offset = np.array([
            rads[0] * np.cos(theta),
            -rads[1] * np.sin(theta),
            -rads[2] * np.sin(theta * z_rate)
        ])
        
        # Transform to world space
        c = c_avg + R_avg @ c_offset
        
        # Compute viewing direction z
        if focus_depth is not None:
            # Forward-facing: look at focal plane
            focal_point = c_avg + R_avg @ np.array([0, 0, -focus_depth])
            z = focal_point - c
        else:
            # General: look at centroid
            z = c_avg - c
            
        z = z / np.linalg.norm(z)
        
        # Compute right vector x
        # Use u_avg, but handle potential alignment issues
        x = np.cross(u_avg, z)
        norm_x = np.linalg.norm(x)
        if norm_x < 1e-6:
            # Fallback if z is parallel to u_avg
            x = np.array([1.0, 0.0, 0.0])
            # Re-cross to get y, then recompute x if needed, or just use fixed up
            # Standard fallback: use world up [0,1,0] if u_avg is ambiguous
            x = np.cross(np.array([0.0, 1.0, 0.0]), z)
            norm_x = np.linalg.norm(x)
            
        x = x / norm_x
        
        # Compute true up vector y
        y = np.cross(z, x)
        
        # Construct pose matrix
        pose = np.eye(4)
        pose[:3, 0] = x
        pose[:3, 1] = y
        pose[:3, 2] = z
        pose[:3, 3] = c
        
        render_poses.append(pose)
        
    return np.stack(render_poses, axis=0)
```

### Parameters and Typical Values
*   `n_poses`: 120–200 frames for smooth video [S3, S6, unofficial].
*   `n_rots`: 1–2 full rotations [S3, S4, unofficial].
*   `z_rate`: 0.5 (vertical oscillation half-cycle per rotation) [S1, S3, S4, unofficial].
*   `rads_scale`: 1.0 [S3, unofficial].
*   `focus_depth`: Scene-dependent, often derived from mean camera distance [S2, S4, unofficial].

### Edge Cases and Pitfalls
*   **Singularities**: If the viewing direction $\mathbf{z}$ aligns with the up vector $\mathbf{u}$, the cross product for $\mathbf{x}$ becomes zero. A fallback up vector (e.g., world $[0,1,0]$) is required [G].
*   **Precision**: Use double precision for trigonometric calculations if high accuracy is needed over many rotations [G].
*   **Boundary**: The last pose should not duplicate the first if the path is closed, hence `endpoint=False` in `linspace` [S1, S3, S4, unofficial].

## Subtopics

### [Circular Camera Path Generation](2734-circular-camera-path-generation.md)
Circular Camera Path Generation creates a sequence of camera poses arranged in a 360-degree circular trajectory around a central origin at a fixed height and radius. It solves the problem of generating consistent, orbiting viewpoints for novel view synthesis or 360-degree visualization. It is a degenerate case of Spiral Camera Path Generation where the vertical offset rate ($z\_rate$) is zero, resulting in a planar orbit rather than a helical one. Choose this when vertical variation is undesirable or when the scene is strictly planar. Link: [Circular Camera Path Generation](2734-circular-camera-path-generation.md).

### Spiral Camera Pose Generation
A variant that generates camera poses in a circle with optional up-vector handling and perspective camera construction using specific libraries (e.g., PyTorch3D's `look_at_view_transform`) [S7, unofficial]. It computes the eye position based on the up vector's y-component to handle vertical alignment, then constructs a `PerspectiveCameras` object with specified focal length and principal point [S7, unofficial]. Choose this when integrating with differentiable rendering pipelines that require explicit camera objects rather than raw transformation matrices. Link: [Circular Camera Path Generation](2734-circular-camera-path-generation.md) (as it is a circular variant, often grouped with spiral methods).

## Operating characteristics
*   **Time Complexity**: $O(N + M)$, where $N$ is the number of training poses (for centroid/radius calculation) and $M$ is the number of output poses [S1, S3, unofficial].
*   **Memory Complexity**: $O(N + M)$ to store input poses and output path [S1, unofficial].
*   **Performance**: Generation is instantaneous on CPU for typical $N, M < 1000$ [S1, S3, unofficial].
*   **Strengths**: Smooth, continuous motion; covers 360°; adaptable to scene scale via automatic radius estimation [S1, S3, unofficial].
*   **Limitations**: Assumes a central scene structure; may not follow elongated or complex geometries well [S3, S6, unofficial]; vertical oscillation may cause unnatural motion for flat scenes if $z\_rate$ is not tuned [S1, unofficial].
*   **When to Use**: Novel view synthesis for central objects, fly-through videos, 360° visualization [S1, S3, S4, unofficial].
*   **When Not to Use**: Elongated structures (use [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md)), or when strict planar orbit is required (use [Circular Camera Path Generation](2734-circular-camera-path-generation.md)) [S1, S3, unofficial].

## Background
Spiral camera paths are widely used in NeRF (Neural Radiance Fields) novel view synthesis, particularly for datasets like Blender synthetic scenes and LLFF forward-facing scenes [S1, S2, S4, unofficial]. The method was popularized by the original NeRF implementation [S1, unofficial] and subsequent variants like NeRF-Pl [S2, unofficial] and various from-scratch tutorials [S3, S4, unofficial]. It is a standard component in rendering pipelines for generating smooth fly-throughs of learned 3D scenes [S3, S6, unofficial].

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S6] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S7] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
