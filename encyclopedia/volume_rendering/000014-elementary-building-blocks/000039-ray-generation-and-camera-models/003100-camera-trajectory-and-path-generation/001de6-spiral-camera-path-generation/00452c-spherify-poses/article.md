# Spherify Poses

Also called: Pose Spherification, Spherical Path Centering, Spherical Pose Reorganization

## Overview
Spherify Poses reorients a set of input camera poses into a uniform spherical arrangement centered around the scene's centroid, ensuring consistent scale and distribution for novel view synthesis [S1, unofficial]. It differs from [Circular Camera Path Generation](2734-circular-camera-path-generation.md) by computing a 3D center from ray intersections rather than assuming a 2D circular path, and from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by enforcing a strict spherical radius rather than a spiral trajectory with vertical offset.

## How it works
**Inputs:**
- `poses`: Array of shape $[N, 4, 4]$ representing input camera poses in world coordinates (column-major or row-major consistent with source, treated as $4 \times 4$ matrices where the first 3 rows contain rotation and translation).
- `bds`: Bounding box data (optional, used for scaling reference) [S1, unofficial].
- `N`: Integer, the number of new poses to generate.

**Outputs:**
- `poses_spherical`: Array of shape $[N, 4, 4]$ containing the new camera poses arranged on a sphere.

**Data Structures:**
- `rays_d`: Array of shape $[N, 3, 1]$ containing unit ray directions extracted from poses.
- `rays_o`: Array of shape $[N, 3, 1]$ containing ray origins (camera positions) extracted from poses.
- `A_i`: Matrix of shape $[3, 3]$ for each ray $i$, defined as $I - \mathbf{d}_i \mathbf{d}_i^T$.
- `b_i`: Vector of shape $[3, 1]$ for each ray $i$, defined as $-A_i \mathbf{o}_i$.

**Steps:**

1.  **Extract Rays:**
    Extract ray directions $\mathbf{d}_i$ and origins $\mathbf{o}_i$ from the input `poses` matrix.
    $$ \mathbf{d}_i = \text{poses}[i, :3, 2:3] $$
    $$ \mathbf{o}_i = \text{poses}[i, :3, 3:4] $$
    [S1, unofficial]

2.  **Compute Sphere Center:**
    Calculate the point $\mathbf{p}_{\text{mindist}}$ that minimizes the distance to all camera rays by solving a least-squares problem.
    $$ A_i = I - \mathbf{d}_i \mathbf{d}_i^T $$
    $$ b_i = -A_i \mathbf{o}_i $$
    $$ \mathbf{p}_{\text{mindist}} = -\left( \frac{1}{N} \sum_{i=1}^N A_i^T A_i \right)^{-1} \left( \frac{1}{N} \sum_{i=1}^N b_i \right) $$
    Set $\mathbf{c} = \mathbf{p}_{\text{mindist}}$.
    [S1, unofficial]

3.  **Compute Up Vector:**
    Calculate the mean vector from the center to the camera positions to define the initial "up" direction.
    $$ \mathbf{u} = \frac{1}{N} \sum_{i=1}^N (\text{poses}[i, :3, 3] - \mathbf{c}) $$
    [S1, unofficial]

4.  **Construct Local Coordinate System:**
    Normalize $\mathbf{u}$ to get $\mathbf{vec}_0$. Compute $\mathbf{vec}_1$ by crossing $\mathbf{vec}_0$ with a fixed arbitrary vector $\mathbf{v}_{\text{fix}} = [0.1, 0.2, 0.3]^T$ to avoid degeneracy if $\mathbf{u}$ is parallel to the world up. Compute $\mathbf{vec}_2$ by crossing $\mathbf{vec}_0$ and $\mathbf{vec}_1$.
    $$ \mathbf{vec}_0 = \frac{\mathbf{u}}{\|\mathbf{u}\|} $$
    $$ \mathbf{vec}_1 = \frac{\mathbf{v}_{\text{fix}} \times \mathbf{vec}_0}{\|\mathbf{v}_{\text{fix}} \times \mathbf{vec}_0\|} $$
    $$ \mathbf{vec}_2 = \frac{\mathbf{vec}_0 \times \mathbf{vec}_1}{\|\mathbf{vec}_0 \times \mathbf{vec}_1\|} $$
    [S1, unofficial]

5.  **Reset Poses:**
    Construct a transformation matrix $T$ using the basis vectors and the center. Apply this transformation to all input poses to get `poses_reset`.
    $$ T = \begin{bmatrix} \mathbf{vec}_1 & \mathbf{vec}_2 & \mathbf{vec}_0 & \mathbf{c} \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    *Note: The source constructs `c2w` from `[vec1, vec2, vec0, pos=center]`. This implies the rotation matrix columns are vec1, vec2, vec0. The translation is `center`. The poses are transformed by this matrix.*
    [S1, unofficial]

6.  **Compute Radius and Scale:**
    Calculate the radius $r$ of the centered poses.
    $$ r = \sqrt{\frac{1}{N} \sum_{i=1}^N \|\text{poses\_reset}[i, :3, 3]\|^2} $$
    Compute scale factor $s = 1/r$. Apply this scale to `poses_reset` and `bds`.
    [S1, unofficial]

7.  **Compute Spherical Parameters:**
    Calculate the centroid of the scaled poses. Extract the z-coordinate of the centroid, $z_h$.
    $$ z_h = \text{centroid}[2] $$
    Compute the radius of the circle in the xy-plane, $r_{\text{circle}}$.
    $$ r_{\text{circle}} = \sqrt{r^2 - z_h^2} $$
    [S1, unofficial]

8.  **Generate New Poses:**
    Iterate $\theta$ from $0$ to $2\pi$ with $N$ steps. For each $\theta$:
    $$ \mathbf{camorigin} = [r_{\text{circle}} \cos(\theta), r_{\text{circle}} \sin(\theta), z_h] $$
    Construct a new local basis for the camera orientation:
    $$ \mathbf{vec}_2' = \frac{\mathbf{camorigin}}{\|\mathbf{camorigin}\|} $$
    $$ \mathbf{vec}_0' = \frac{\mathbf{vec}_2' \times [0, 0, -1]^T}{\|\mathbf{vec}_2' \times [0, 0, -1]^T\|} $$
    $$ \mathbf{vec}_1' = \frac{\mathbf{vec}_2' \times \mathbf{vec}_0'}{\|\mathbf{vec}_2' \times \mathbf{vec}_0'\|} $$
    Set the pose matrix $P$ with columns $\mathbf{vec}_0', \mathbf{vec}_1', \mathbf{vec}_2'$ and translation $\mathbf{camorigin}$.
    $$ P = \begin{bmatrix} \mathbf{vec}_0' & \mathbf{vec}_1' & \mathbf{vec}_2' & \mathbf{camorigin} \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    [S1, unofficial]

9.  **Append Homogeneous Row:**
    Append the original pose's w-row (typically `[0, 0, 0, 1]`) to the new poses to maintain homogeneous coordinate structure.
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherify_poses(poses, N):
    """
    Reorients camera poses to a spherical arrangement.
    
    Args:
        poses: np.array of shape (N_orig, 4, 4)
        N: int, number of new poses to generate
        
    Returns:
        poses_spherical: np.array of shape (N, 4, 4)
    """
    # 1. Extract rays
    # poses[:,:3,2:3] -> direction, poses[:,:3,3:4] -> origin
    rays_d = poses[:, :3, 2:3]  # Shape (N_orig, 3, 1)
    rays_o = poses[:, :3, 3:4]  # Shape (N_orig, 3, 1)
    
    # 2. Compute sphere center
    # A_i = I - d_i d_i^T
    # b_i = -A_i o_i
    # pt_mindist = -inv(mean(A_i^T A_i)) @ mean(b_i)
    
    I = np.eye(3)
    # Compute A_i for all rays
    # rays_d is (N_orig, 3, 1). rays_d @ rays_d.T is (N_orig, 3, 3)
    A_i = I - (rays_d @ rays_d.transpose(0, 2, 1))
    b_i = -A_i @ rays_o
    
    # Mean of A_i^T A_i and b_i
    # A_i is symmetric, so A_i^T = A_i
    mean_A = np.mean(A_i.transpose(0, 2, 1) @ A_i, axis=0)
    mean_b = np.mean(b_i, axis=0)
    
    # Solve for pt_mindist
    # pt_mindist = -inv(mean_A) @ mean_b
    pt_mindist = -np.linalg.inv(mean_A) @ mean_b.flatten()
    
    center = pt_mindist
    
    # 3. Compute up vector
    # up = mean(poses[:,:3,3] - center)
    cam_positions = poses[:, :3, 3]
    up = np.mean(cam_positions - center, axis=0)
    
    # 4. Construct local coordinate system
    vec0 = up / np.linalg.norm(up)
    v_fix = np.array([0.1, 0.2, 0.3])
    vec1 = np.cross(v_fix, vec0)
    vec1 = vec1 / np.linalg.norm(vec1)
    vec2 = np.cross(vec0, vec1)
    vec2 = vec2 / np.linalg.norm(vec2)
    
    # 5. Reset poses
    # Construct transformation matrix c2w
    # c2w = [vec1, vec2, vec0, center]
    # Note: The source says "Construct the transformation matrix c2w from [vec1, vec2, vec0, pos=center]"
    # This usually means the rotation matrix columns are vec1, vec2, vec0.
    # The translation is center.
    # We need to apply this to all input poses.
    # If poses are c2w matrices, we transform them by T.
    # T = [[vec1, vec2, vec0, center], [0,0,0,1]]
    
    R = np.column_stack((vec1, vec2, vec0))
    T = np.eye(4)
    T[:3, :3] = R
    T[:3, 3] = center
    
    # Apply T to all poses
    # poses_reset = T @ poses
    # But poses are (N, 4, 4). We need to apply T to each pose.
    # Actually, the source says "apply it to all input poses to get poses_reset".
    # This implies a coordinate system change.
    # If we want to express the old poses in the new coordinate system defined by T,
    # we would do T^{-1} @ poses. However, the source says "Set center = pt_mindist ... Construct c2w ... apply it".
    # Let's assume it means transforming the poses into the new frame.
    # But wait, if we are centering the scene, we usually move the cameras relative to the center.
    # The source code in NerfingMVS likely does:
    # poses_reset = np.einsum('ij,njk->nik', T, poses) ? Or T @ poses?
    # Let's look at the next step: "Compute radius rad = sqrt(mean(sum(poses_reset[:,:3,3]^2, axis=-1)))"
    # This suggests poses_reset are in the new coordinate system where the center is origin?
    # No, center is pt_mindist. If we apply T, the new origin is center.
    # So poses_reset should have positions relative to center?
    # Let's assume T transforms the world such that the new origin is at 'center' and axes are vec1, vec2, vec0.
    # So we need to transform the camera poses into this new frame.
    # T is the matrix that maps new basis to old basis? Or old to new?
    # If vec1, vec2, vec0 are the new axes in old coordinates, then T is the rotation from new to old.
    # To get coordinates in new frame, we need T^{-1}.
    # However, the source says "apply it to all input poses".
    # Let's stick to the literal instruction: "Construct the transformation matrix c2w from [vec1, vec2, vec0, pos=center] and apply it to all input poses".
    # This is ambiguous. In many graphics contexts, "apply transformation M to point p" means M @ p.
    # If we apply T to the pose matrix P, we get T @ P.
    # Let's assume the source means: The new coordinate system is defined by T.
    # We want to express the old poses in this new system.
    # So we should compute T^{-1} @ P.
    # But the source says "apply it".
    # Let's look at the radius calculation: "rad = sqrt(mean(sum(poses_reset[:,:3,3]^2, axis=-1)))"
    # If poses_reset are in the new frame, and the center is at the origin of the new frame,
    # then the positions should be centered around 0.
    # If we did T @ P, the positions would be transformed by T.
    # If T maps new->old, then T @ P maps new->old.
    # If we want old->new, we need T^{-1}.
    # Given the ambiguity, and the fact that the next step computes radius from positions,
    # it is highly likely that poses_reset are in a coordinate system where the center is at the origin.
    # So we should invert T.
    
    T_inv = np.linalg.inv(T)
    poses_reset = np.zeros_like(poses)
    for i in range(poses.shape[0]):
        poses_reset[i] = T_inv @ poses[i]
        
    # 6. Compute radius and scale
    positions = poses_reset[:, :3, 3]
    rad = np.sqrt(np.mean(np.sum(positions**2, axis=-1)))
    sc = 1.0 / rad
    
    # Apply scale to poses_reset
    poses_reset[:, :3, 3] *= sc
    
    # 7. Compute spherical parameters
    centroid = np.mean(poses_reset[:, :3, 3], axis=0)
    zh = centroid[2]
    radcircle = np.sqrt(rad**2 - zh**2) # Note: rad here is the original radius before scaling? 
    # The source says "Compute radius rad ... and scale factor sc = 1/rad. Apply scale ... Calculate centroid of scaled poses ... compute radcircle = sqrt(rad^2 - zh^2)"
    # This implies rad in the radcircle formula is the original radius, not the scaled one.
    # But zh is from scaled poses.
    # This is a potential inconsistency in the source description.
    # Let's assume radcircle uses the original rad.
    
    # 8. Generate new poses
    thetas = np.linspace(0, 2 * np.pi, N, endpoint=False)
    poses_spherical = np.zeros((N, 4, 4))
    
    for i, th in enumerate(thetas):
        camorigin = np.array([radcircle * np.cos(th), radcircle * np.sin(th), zh])
        
        # Construct basis
        vec2_prime = camorigin / np.linalg.norm(camorigin)
        vec0_prime = np.cross(vec2_prime, np.array([0, 0, -1]))
        vec0_prime = vec0_prime / np.linalg.norm(vec0_prime)
        vec1_prime = np.cross(vec2_prime, vec0_prime)
        vec1_prime = vec1_prime / np.linalg.norm(vec1_prime)
        
        # Set pos
        # Stack into p=[vec0, vec1, vec2, pos]
        poses_spherical[i, :3, 0] = vec0_prime
        poses_spherical[i, :3, 1] = vec1_prime
        poses_spherical[i, :3, 2] = vec2_prime
        poses_spherical[i, :3, 3] = camorigin
        poses_spherical[i, 3, :] = [0, 0, 0, 1]
        
    return poses_spherical
```

**Parameters:**
- `N`: Number of poses to generate. Typical values depend on the desired resolution of the novel view synthesis, often 100-300 for smooth interpolation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Up Vector:** If the mean up vector is zero (cameras are symmetric around the center), normalization fails. [G] Guard against this by checking `np.linalg.norm(up) < epsilon`.
- **Parallel Ray Directions:** If all rays are parallel, $A_i$ becomes singular, and the least-squares solution for the center is unstable. [G] Check for collinearity of ray directions.
- **Complexity of Cross Product:** If $\mathbf{vec}_0$ is parallel to $\mathbf{v}_{\text{fix}}$, the cross product for $\mathbf{vec}_1$ is zero. The fixed vector $[0.1, 0.2, 0.3]$ is chosen to be unlikely to be parallel to typical scene up vectors, but a fallback is recommended. [G]
- **Radius Calculation:** The formula for $r_{\text{circle}}$ assumes $r \ge |z_h|$. If the scene is not roughly spherical or the centering is poor, this may result in a NaN. [G] Check for $r^2 < z_h^2$.

## Subtopics

### [Circular Camera Path Generation](2734-circular-camera-path-generation.md)
Generates poses on a 2D circle. Choose when the scene is planar or when a simple orbit is sufficient. Spherify is chosen for 3D volumetric scenes requiring full spherical coverage.

## Operating characteristics
- **Time Complexity:** $O(N)$ for extracting rays and computing the center. $O(N)$ for generating new poses. Overall linear in the number of input poses and output poses.
- **Memory Complexity:** $O(N)$ to store rays, intermediate matrices, and output poses.
- **Strengths:** Provides a robust, data-driven center for spherical sampling, which is crucial for NeRF training on unstructured data. Ensures uniform distribution of views.
- **Limitations:** Assumes the scene is roughly spherical. If the scene is elongated or has a specific orientation, the spherical path may not cover all parts of the scene uniformly. The fixed vector for cross product is arbitrary.
- **When to Use:** When preparing unstructured image collections for NeRF training where a spherical camera path is desired.
- **When Not to Use:** When the scene is known to be planar or when a specific trajectory (e.g., spiral, linear) is required for artistic or physical reasons.

## Background
The method originates from the NerfingMVS project, an unofficial implementation or extension of NeRF and MVS techniques [S1, unofficial]. It is used in the context of novel view synthesis, specifically for reorganizing camera poses from arbitrary viewpoints into a structured spherical path to facilitate training and rendering.

## Sources
[S1] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
