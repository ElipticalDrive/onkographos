# Spiral Camera Path Generation

Also called: Forward-Facing Spiral Camera Path, Novel View Synthesis Path, Spiral Path Generation, load_tankstemple_data spiral generation

## Overview
Spiral Camera Path Generation creates a sequence of camera poses along a spiral trajectory for novel view synthesis, defined by a center camera, radius parameters, focal length, vertical offset rate, rotation count, and number of views [S1, unofficial]. It calculates a forward-facing spiral camera path for rendering based on input poses and bounds, where all cameras in the path point towards a world-space focal point [S2, unofficial]. It generates a smooth spiral camera path around the training camera distribution for 360° novel view synthesis videos [S4, unofficial] and for rendering novel views in forward-facing scenes [S5, unofficial]. It generates a spiral camera trajectory around a center point for novel view synthesis [S6, unofficial].

## How it works
**Inputs:**
*   `poses`: Array of $M$ training camera-to-world matrices $\mathbf{C}_i$ [S3, S4, unofficial].
*   `N`: Number of views to generate [S1, S5, unofficial].
*   `rots`: Number of full rotations ($2\pi$) to complete, default 2 [S3, S4, unofficial].
*   `z_rate`: Rate of vertical oscillation in the spiral path, default 0.5 [S3, unofficial].
*   `rads_scale`: Scale factor for camera radius, default 1.0 [S3, unofficial].
*   `focus_distance`: Weight for near/far bound averaging, default 0.75 [S6, unofficial].
*   `spiral_scale_f`: Focal length multiplier, default 1.0 [S6, unofficial].
*   `spiral_scale_r`: Radius multiplier, default 1.0 [S6, unofficial].

**Outputs:**
*   `render_poses`: Array of shape $(N, 4, 4)$ containing the generated camera-to-world matrices [S3, unofficial].

**Data Structures:**
*   `rads`: Array of shape $(3,)$ specifying spiral radius per axis, with 1.0 appended if not present [S1, S3, unofficial].
*   `c2w_avg`: $4 \times 4$ matrix representing the average camera pose of the input training data [S3, S4, unofficial].
*   `up`: $3$-vector representing the average up direction of the input cameras [S3, S4, unofficial].
*   `focal`: Scalar float representing the effective focal length [S3, S6, unofficial].

**Steps:**
1.  **Scene Analysis**:
    *   Compute the average camera-to-world matrix $\mathbf{C}_{avg}$ from the input `poses` [S3, S4, unofficial]. Let $\mathbf{C}_{avg} = [\mathbf{R}_{avg} | \mathbf{t}_{avg}]$ [S3, unofficial].
    *   Compute the average up vector $\mathbf{u}$ by averaging the second column (y-axis) of the rotation matrices of all input poses: $\mathbf{u} = \frac{1}{M} \sum_{i=0}^{M-1} \mathbf{C}_i[:, 1]$ [S3, S4, unofficial].
    *   Extract the camera centers $\mathbf{p}_i = \mathbf{C}_i[:, 3]$ for all $i$ [S3, unofficial].
    *   Compute the radii vector $\mathbf{r}_{raw}$ as the 90th percentile of the absolute values of the camera centers along each axis: $\mathbf{r}_{raw} = \text{percentile}(|\mathbf{p}_i|, 90, \text{axis}=0)$ [S3, S4, unofficial].
    *   Scale the radii: $\mathbf{r}_{scaled} = \mathbf{r}_{raw} \times \text{rads\_scale} \times \text{spiral\_scale\_r}$ [S3, S6, unofficial].
    *   Ensure $\mathbf{rads}$ has 3 elements. If the input `rads` is provided, use it; otherwise, construct $\mathbf{rads} = [\text{rads\_scaled}[0], \text{rads\_scaled}[1], \text{rads\_scaled}[2]]$ [S1, S5, unofficial]. Note: Some implementations append 1.0 to a 2-element list if not present, but standard spiral generation uses 3 axes [S1, S5, unofficial]. We assume $\mathbf{rads}$ is a 3-element array $[r_x, r_y, r_z]$ [S3, S4, unofficial].
2.  **Focal Length and Bounds Calculation**:
    *   Compute near and far bounds from the training poses. Let $d_{min} = \min_i \|\mathbf{p}_i\|$ and $d_{max} = \max_i \|\mathbf{p}_i\|$ [S6, unofficial].
    *   Compute the effective focal length $f$ using the weighted harmonic mean:
        $$ f = \frac{1}{\frac{1 - \text{focus\_distance}}{d_{min}} + \frac{\text{focus\_distance}}{d_{max}}} \times \text{spiral\_scale\_f} $$
        [S6, unofficial]. Another simpler method uses the mean norm of camera positions: $f = \text{mean}(\|\mathbf{p}_i\|)$ [S3, unofficial].
3.  **Parameterization and Pose Construction**:
    *   Initialize an empty list `render_poses` [S4, unofficial].
    *   Generate $N$ evenly spaced angles $\theta_j$ for $j = 0, \dots, N-1$:
        $$ \theta_j = \frac{2\pi \cdot j \cdot \text{rots}}{N} $$
        [S1, S3, S4, unofficial]. Note: Some implementations generate $N+1$ points and exclude the last to avoid duplication, effectively creating a closed loop without redundant start/end points [S1, S3, S4, unofficial].
    *   For each $\theta_j$:
        a.  **Compute Offset Vector**: Calculate the offset vector $\mathbf{v}$ in the local coordinate system of the average pose:
            $$ \mathbf{v} = [r_x \cos(\theta_j), -r_y \sin(\theta_j), -r_z \sin(\theta_j \cdot \text{z\_rate})] $$
            [S1, S3, S4, unofficial]. The negative signs on the sine terms are consistent with the coordinate system conventions used in the source implementations (e.g., OpenGL-style camera looking down -Z) [S1, S3, S4, unofficial].
        b.  **Compute Camera Center**: Transform the offset into world space using the average pose's rotation and translation:
            $$ \mathbf{c}_j = \mathbf{C}_{avg}[:, :3] \mathbf{v} + \mathbf{C}_{avg}[:, 3] $$
            [S1, S3, S4, unofficial].
        c.  **Compute Look-at Direction**: Determine the viewing direction $\mathbf{z}_j$. The camera looks towards a focal point defined by the average pose's origin and focal length [S3, unofficial]. The focal point in world space is:
            $$ \mathbf{f}_j = \mathbf{C}_{avg}[:, :3] [0, 0, -f]^T + \mathbf{C}_{avg}[:, 3] $$
            [S1, S4, unofficial]. The viewing direction is the normalized vector from the camera center to the focal point:
            $$ \mathbf{z}_j = \text{normalize}(\mathbf{f}_j - \mathbf{c}_j) $$
            [S4, unofficial].
        d.  **Compute Basis Vectors**:
            i.  Compute the x-axis $\mathbf{x}_j$ using the cross product of the fixed average up vector $\mathbf{u}$ and $\mathbf{z}_j$:
                $$ \mathbf{x}_j = \text{normalize}(\mathbf{u} \times \mathbf{z}_j) $$
                [S4, unofficial].
            ii. Compute the y-axis $\mathbf{y}_j$ to ensure orthogonality:
                $$ \mathbf{y}_j = \mathbf{z}_j \times \mathbf{x}_j $$
                [S1, S4, unofficial].
        e.  **Construct Pose Matrix**: Assemble the $4 \times 4$ matrix $\mathbf{P}_j$:
            $$ \mathbf{P}_j = \begin{bmatrix} \mathbf{x}_j^T & \mathbf{y}_j^T & \mathbf{z}_j^T & \mathbf{c}_j \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
            [S1, S4, unofficial]. Note: In column-major notation (common in OpenGL/NeRF), the rotation matrix columns are $\mathbf{x}_j, \mathbf{y}_j, \mathbf{z}_j$. So $\mathbf{P}_j = [\mathbf{x}_j, \mathbf{y}_j, \mathbf{z}_j, \mathbf{c}_j]$ with the bottom row $[0,0,0,1]$ [S1, S4, unofficial].
4.  **Return**: Stack all $\mathbf{P}_j$ into an array of shape $(N, 4, 4)$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spiral_camera_path_generation(poses, N, rots=2, z_rate=0.5, rads_scale=1.0, 
                                  focus_distance=0.75, spiral_scale_f=1.0, 
                                  spiral_scale_r=1.0):
    """
    Generate a spiral camera path.
    
    Args:
        poses: (M, 4, 4) array of camera-to-world matrices.
        N: Number of poses to generate.
        rots: Number of full rotations.
        z_rate: Vertical oscillation rate.
        rads_scale: Scale factor for radii.
        focus_distance: Weight for near/far bound averaging.
        spiral_scale_f: Focal length multiplier.
        spiral_scale_r: Radius multiplier.
        
    Returns:
        render_poses: (N, 4, 4) array of generated camera-to-world matrices.
    """
    # 1. Scene Analysis
    # Compute average pose (S3, S4)
    c2w_avg = np.mean(poses, axis=0)
    
    # Compute average up vector (S3, S4)
    up = np.mean(poses[:, :3, 1], axis=0)
    
    # Compute radii (S3, S4)
    positions = poses[:, :3, 3]
    radii_raw = np.percentile(np.abs(positions), 90, axis=0)
    rads = radii_raw * rads_scale * spiral_scale_r
    
    # Ensure rads has 3 elements (S1, S5)
    if len(rads) == 2:
        rads = np.append(rads, 1.0)
    elif len(rads) != 3:
        raise ValueError("rads must have 2 or 3 elements")

    # 2. Focal Length and Bounds (S6)
    dists = np.linalg.norm(positions, axis=1)
    near_bound = np.min(dists)
    far_bound = np.max(dists)
    
    # Avoid division by zero (G)
    if near_bound < 1e-6:
        near_bound = 1e-6
        
    focal = (1.0 / ((1.0 - focus_distance) / near_bound + focus_distance / far_bound)) * spiral_scale_f

    # 3. Pose Construction
    render_poses = []
    
    # Generate angles, excluding last point to avoid duplication (S1, S3, S4)
    thetas = np.linspace(0.0, 2.0 * np.pi * rots, N + 1)[:-1]
    
    for theta in thetas:
        # a. Compute Offset Vector (S1, S3, S4)
        offset = np.array([
            rads[0] * np.cos(theta),
            -rads[1] * np.sin(theta),
            -rads[2] * np.sin(theta * z_rate)
        ])
        
        # b. Compute Camera Center (S1, S3, S4)
        center = c2w_avg[:3, 3] + c2w_avg[:3, :3] @ offset
        
        # c. Compute Look-at Direction (S4)
        # Focal point in world space
        focal_point = c2w_avg[:3, :3] @ np.array([0.0, 0.0, -focal]) + c2w_avg[:3, 3]
        # Viewing direction from center to focal point
        z = focal_point - center
        norm_z = np.linalg.norm(z)
        if norm_z > 1e-6:
            z = z / norm_z
        else:
            z = np.array([0.0, 0.0, -1.0]) # Fallback (G)
        
        # d. Compute Basis Vectors (S4)
        x = np.cross(up, z)
        norm_x = np.linalg.norm(x)
        if norm_x < 1e-6:
            # Degenerate case: up and z are parallel (G)
            x = np.array([1.0, 0.0, 0.0])
            x = x - np.dot(x, z) * z
            x = x / np.linalg.norm(x)
        else:
            x = x / norm_x
            
        y = np.cross(z, x)
        
        # e. Construct Pose Matrix (S1, S4)
        pose = np.eye(4)
        pose[:3, 0] = x
        pose[:3, 1] = y
        pose[:3, 2] = z
        pose[:3, 3] = center
        
        render_poses.append(pose)
        
    return np.array(render_poses)
```

**Edge Cases and Pitfalls:**
*   **Degenerate Up Vector**: If the average up vector is parallel to the viewing direction $\mathbf{z}$, the cross product for $\mathbf{x}$ is zero. A guard is required to select an alternative up vector [G].
*   **Precision**: Floating point errors in normalization can accumulate. Re-orthogonalization (Gram-Schmidt) is recommended [G].
*   **Boundary Conditions**: The spiral should not include the final duplicate point to avoid redundancy in rendering [S1, S3, S4, unofficial].
*   **Focal Length**: If `near_bound` is zero, division by zero occurs. A small epsilon should be added [G].

## Subtopics

### [Circular Camera Path Generation](2734-circular-camera-path-generation.md)
Circular Camera Path Generation creates a sequence of camera poses arranged in a 360-degree circular trajectory around a central origin at a fixed height and radius. It differs from Spiral Camera Path Generation by having a constant z-coordinate (no vertical oscillation, `z_rate` = 0) and a fixed radius, making it suitable for flat scenes or when vertical coverage is not required. It is simpler than the spiral path and computationally cheaper. Link: [Circular Camera Path Generation](2734-circular-camera-path-generation.md).

### [Spherify Poses](11414-spherify-poses.md)
Spherify Poses reorients a set of input camera poses into a uniform spherical arrangement centered around the scene's centroid, ensuring consistent scale and distribution for novel view synthesis. It differs from Spiral Camera Path Generation by enforcing a strict spherical radius rather than a spiral trajectory with vertical offset, and by computing a 3D center from ray intersections rather than assuming a 2D circular path. It is used when a full 360° spherical coverage is needed, unlike the spiral which is often forward-facing or limited in elevation. Link: [Spherify Poses](11414-spherify-poses.md).

### Spiral Camera Pose Generation
Spiral Camera Pose Generation is a variant that generates camera poses in a circle with optional pitch rotation and up-vector flipping. It differs from the standard spiral by using a fixed radius and potentially flipping the up vector for specific rendering requirements. It is used for fly-through rendering where a simple circular path with orientation adjustments is sufficient. Link: [Spiral Camera Pose Generation](1635-spiral-camera-path-generation.md).

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of poses to generate. The computation per pose is constant time (vector operations) [G].
*   **Memory Complexity**: $O(N)$ to store the output poses [G].
*   **Measured Performance**: Not given in the sources [G].
*   **Scaling**: Scales linearly with the number of poses. The radius and focal length depend on the scene statistics, so larger scenes require larger radii [G].
*   **Strengths**: Provides smooth, continuous coverage of a scene with vertical variation. Good for forward-facing and 360° novel view synthesis [S4, unofficial].
*   **Limitations**: Assumes the scene is roughly centered at the origin. May not cover scenes with large vertical extent if `z_rate` is not tuned [G]. Degenerate cases (parallel up and view vectors) require handling.
*   **When to Use**: When smooth, continuous camera motion with vertical variation is desired for novel view synthesis.
*   **When Not to Use**: When the scene is not centered, or when a fixed-height orbit is sufficient.

## Background
Spiral Camera Path Generation originated in the context of NeRF (Neural Radiance Fields) for novel view synthesis. It was popularized by the original NeRF paper and subsequent implementations in repositories like `bmild/nerf` and `kwea123/nerf_pl` [S1, S2, unofficial]. It is commonly used for rendering novel views in forward-facing scenes (e.g., LLFF dataset) and 360° capture scenarios [S4, S3, unofficial]. The method maps a scalar parameter to a 4x4 transformation matrix, building on the principles of parametric path generation.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S3] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S4] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S6] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S7] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
[S8] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S9] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S10] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
