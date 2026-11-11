# Spiral Camera Path Generation

Also called: Forward-Facing Spiral Camera Path, Novel View Synthesis Path, load_tankstemple_data spiral generation

## Overview
Spiral Camera Path Generation creates a smooth, continuous trajectory of camera poses spiraling around a scene's centroid, designed primarily for novel view synthesis in forward-facing or 360° capture scenarios [S1, S3, S4, S5, S6, unofficial]. Unlike the [Elliptical Camera Path Generation](7627-elliptical-camera-path-generation.md) which maintains a fixed vertical plane, or the [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) which orbits at a constant height, the spiral path introduces vertical oscillation (controlled by `z_rate`) to cover varying elevations, providing better coverage for scenes with significant vertical structure [S1, S3, unofficial]. It builds on [Parametric Path Generation](11476-parametric-path-generation.md) by mapping a scalar parameter to a 4x4 transformation matrix with specific sinusoidal offsets for the z-axis.

## How it works
**Inputs:**
*   `poses`: List or array of $M$ training camera poses, each a $4 \times 4$ camera-to-world matrix $\mathbf{C}_i$.
*   `N`: Number of output poses to generate [S1, S4, unofficial].
*   `rots`: Number of full rotations ($2\pi$) to complete [S1, S3, S4, unofficial].
*   `z_rate`: Vertical oscillation rate [S1, S3, unofficial]. Default 0.5 [S3, unofficial].
*   `rads_scale`: Scale factor for the computed radii [S3, unofficial]. Default 1.0 [S3, unofficial].
*   `focus_distance`: Weight for near/far bound averaging [S6, unofficial]. Default 0.75 [S6, unofficial].
*   `spiral_scale_f`: Focal length multiplier [S6, unofficial]. Default 1.0 [S6, unofficial].
*   `spiral_scale_r`: Radius multiplier [S6, unofficial]. Default 1.0 [S6, unofficial].

**Outputs:**
*   `render_poses`: Array of shape $(N, 4, 4)$ containing the generated camera-to-world matrices.

**Data Structures:**
*   `rads`: Array of shape $(3,)$. Stores the spiral radii for x, y, and z axes. Initialized from scene statistics.
*   `c2w_avg`: $4 \times 4$ matrix representing the average camera pose of the input training data.
*   `up`: $3$-vector representing the average up direction of the input cameras.
*   `focal`: Scalar float representing the effective focal length.

**Steps:**

1.  **Scene Analysis**:
    *   Compute the average camera-to-world matrix $\mathbf{C}_{avg}$ from the input `poses` [S3, S4, unofficial]. Let $\mathbf{C}_{avg} = [\mathbf{R}_{avg} | \mathbf{t}_{avg}]$.
    *   Compute the average up vector $\mathbf{u}$ by averaging the second column (y-axis) of the rotation matrices of all input poses: $\mathbf{u} = \frac{1}{M} \sum_{i=0}^{M-1} \mathbf{C}_i[:, 1]$ [S3, S4, unofficial].
    *   Extract the camera centers $\mathbf{p}_i = \mathbf{C}_i[:, 3]$ for all $i$.
    *   Compute the radii vector $\mathbf{r}_{raw}$ as the 90th percentile of the absolute values of the camera centers along each axis: $\mathbf{r}_{raw} = \text{percentile}(|\mathbf{p}_i|, 90, \text{axis}=0)$ [S3, S4, unofficial].
    *   Scale the radii: $\mathbf{r}_{scaled} = \mathbf{r}_{raw} \times \text{rads\_scale} \times \text{spiral\_scale\_r}$ [S3, S6, unofficial].
    *   Ensure $\mathbf{rads}$ has 3 elements. If the input `rads` is provided, use it; otherwise, construct $\mathbf{rads} = [\text{rads\_scaled}[0], \text{rads\_scaled}[1], \text{rads\_scaled}[2]]$ [S1, S5, unofficial]. Note: Some implementations append 1.0 to a 2-element list if not present, but standard spiral generation uses 3 axes [S1, S5, unofficial]. We assume $\mathbf{rads}$ is a 3-element array $[r_x, r_y, r_z]$ [S3, S4, unofficial].

2.  **Focal Length and Bounds Calculation**:
    *   Compute near and far bounds from the training poses. A common heuristic is to use the minimum and maximum distances of camera centers from the origin or the average pose origin [S6, unofficial]. Let $d_{min} = \min_i \|\mathbf{p}_i\|$ and $d_{max} = \max_i \|\mathbf{p}_i\|$ [S6, unofficial].
    *   Compute the effective focal length $f$. One method uses the harmonic mean weighted by `focus_distance`:
        $$ f = \frac{1}{\frac{1 - \text{focus\_distance}}{d_{min}} + \frac{\text{focus_distance}}{d_{max}}} \times \text{spiral\_scale\_f} $$
        [S6, unofficial]. Another simpler method uses the mean norm of camera positions: $f = \text{mean}(\|\mathbf{p}_i\|)$ [S3, unofficial]. We cite the weighted harmonic mean as it is more robust to outliers [S6, unofficial].

3.  **Parameterization and Pose Construction**:
    *   Initialize an empty list `render_poses`.
    *   Generate $N$ evenly spaced angles $\theta_j$ for $j = 0, \dots, N-1$:
        $$ \theta_j = \frac{2\pi \cdot j \cdot \text{rots}}{N} $$
        [S1, S3, S4, unofficial]. Note: Some implementations generate $N+1$ points and exclude the last to avoid duplication, effectively creating a closed loop without redundant start/end points [S1, S3, S4, unofficial].
    *   For each $\theta_j$:
        a.  **Compute Offset Vector**: Calculate the offset vector $\mathbf{v}$ in the local coordinate system of the average pose:
            $$ \mathbf{v} = [r_x \cos(\theta_j), -r_y \sin(\theta_j), -r_z \sin(\theta_j \cdot \text{z\_rate})] $$
            [S1, S3, S4, unofficial]. The negative signs on the sine terms are consistent with the coordinate system conventions used in the source implementations (e.g., OpenGL-style camera looking down -Z) [S1, S3, S4, unofficial].
        b.  **Compute Camera Center**: Transform the offset into world space using the average pose's rotation and translation:
            $$ \mathbf{c}_j = \mathbf{C}_{avg}[:, :3] \mathbf{v} + \mathbf{C}_{avg}[:, 3] $$
            [S1, S3, S4, unofficial]. This is equivalent to $\mathbf{c}_j = \mathbf{C}_{avg} \cdot [\mathbf{v}; 1]$ [S1, unofficial].
        c.  **Compute Look-at Direction**: Determine the viewing direction $\mathbf{z}_j$. The camera looks towards a focal point defined by the average pose's origin and focal length. The focal point in world space is:
            $$ \mathbf{f}_j = \mathbf{C}_{avg}[:, :3] [0, 0, -f]^T + \mathbf{C}_{avg}[:, 3] $$
            [S1, S4, unofficial]. The viewing direction is the normalized vector from the camera center to the focal point:
            $$ \mathbf{z}_j = \text{normalize}(\mathbf{f}_j - \mathbf{c}_j) $$
            [S1, S4, unofficial]. Alternatively, some sources define $\mathbf{z}_j$ as the normalized vector from the focus plane to the center [S2, unofficial].
        d.  **Compute Basis Vectors**:
            i.  Compute the x-axis $\mathbf{x}_j$ using the cross product of the fixed average up vector $\mathbf{u}$ and $\mathbf{z}_j$:
                $$ \mathbf{x}_j = \text{normalize}(\mathbf{u} \times \mathbf{z}_j) $$
                [S1, S4, unofficial]. Note: If $\mathbf{u}$ and $\mathbf{z}_j$ are parallel, this degenerates. A guard is needed [G].
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
    # Compute average pose
    c2w_avg = np.mean(poses, axis=0) # Approximation; better to average rotation and translation separately or use median. 
                                    # Sources use np.mean or specific average_pose function. 
                                    # We follow S3/S4 which imply a simple average or specific helper.
                                    # S3: "Compute average pose c2w using _poses_avg(poses)."
                                    # S4: "c2w = average_pose(poses)."
                                    # For implementation, we assume a helper `average_pose` exists or use mean.
                                    # Let's use mean for simplicity as per S3/S4 notes, though S3/S4 might imply a more robust average.
                                    # [G] If poses are not aligned, mean of matrices is not a valid rotation. 
                                    # A robust average would average the translation and average the rotation via SVD or quaternion averaging.
                                    # However, for forward-facing scenes (LLFF), mean is often used [S4, unofficial].
    
    # Compute average up vector
    # S3: "up = poses[:, :3, 1].mean(0)"
    up = np.mean(poses[:, :3, 1], axis=0)
    
    # Compute radii
    # S3: "radii = np.percentile(np.abs(positions), 90, 0) * spiral_scale_r"
    positions = poses[:, :3, 3]
    radii_raw = np.percentile(np.abs(positions), 90, axis=0)
    rads = radii_raw * rads_scale * spiral_scale_r
    
    # Ensure rads has 3 elements
    # S1: "append 1. to the rads array if it does not already have 4 elements" -> This suggests rads might be [rx, ry] and 1.0 is appended for rz?
    # S3: "rads = np.array(list(rads) + [1.0])" -> This implies rads is [rx, ry] and rz is 1.0?
    # S4: "rads: Radii for the spiral [3]" -> Explicitly 3 elements.
    # S1/S5: "rads = np.array(list(rads) + [1.0])" -> If input rads is [rx, ry], then rz=1.0.
    # We will assume rads is [rx, ry, rz] as per S3/S4. If the source provides only 2, we append 1.0.
    if len(rads) == 2:
        rads = np.append(rads, 1.0)
    elif len(rads) != 3:
        raise ValueError("rads must have 2 or 3 elements")

    # 2. Focal Length and Bounds
    # S6: Compute near_bound and far_bound
    # S6: "near_bound = bounds.min(), far_bound = bounds.max"
    # S6: "focal = 1 / (((1 - focus_distance) / near_bound + focus_distance / far_bound)) * spiral_scale_f"
    # We need bounds. S6 implies bounds are computed from poses.
    # S6: "bounds" is not explicitly defined in the snippet, but typically min/max of camera distances or scene extent.
    # S3/S4 do not use near/far bounds for focal length, they use mean norm or explicit focal.
    # S6 is the only source using focus_distance. We follow S6 for this part.
    # [G] Assuming bounds are min/max of camera center distances from origin.
    dists = np.linalg.norm(positions, axis=1)
    near_bound = np.min(dists)
    far_bound = np.max(dists)
    
    # Avoid division by zero
    if near_bound == 0:
        near_bound = 1e-6
        
    focal = (1.0 / ((1.0 - focus_distance) / near_bound + focus_distance / far_bound)) * spiral_scale_f

    # 3. Pose Construction
    render_poses = []
    
    # S1: "Iterate theta over np.linspace(0., 2. * np.pi * rots, N+1) excluding the last element"
    thetas = np.linspace(0.0, 2.0 * np.pi * rots, N + 1)[:-1]
    
    for theta in thetas:
        # a. Compute Offset Vector
        # S1: "t = radii * [cos(theta), -sin(theta), -sin(theta*zrate), 1.0]"
        # S3: "center = [cos(t), -sin(t), -sin(0.5*t)] * radii"
        # S4: "c_offset = [cos(theta)*rads[0], -sin(theta)*rads[1], -sin(theta*z_rate)*rads[2]]"
        offset = np.array([
            rads[0] * np.cos(theta),
            -rads[1] * np.sin(theta),
            -rads[2] * np.sin(theta * z_rate)
        ])
        
        # b. Compute Camera Center
        # S1: "position = cam2world @ t" where t is [offset, 1]
        # S3: "c = c2w[:3, 3] + c2w[:3, :3] @ c_offset"
        center = c2w_avg[:3, 3] + c2w_avg[:3, :3] @ offset
        
        # c. Compute Look-at Direction
        # S1: "lookat = cam2world @ [0, 0, -focal, 1.0]"
        # S4: "lookat = c2w @ [0, 0, -focal, 1.0]"
        focal_point = c2w_avg[:3, :3] @ np.array([0.0, 0.0, -focal]) + c2w_avg[:3, 3]
        
        # S1: "z_axis = position - lookat" -> This is wrong direction. Camera looks FROM center TO focal?
        # S4: "z = normalize(c - dot(c2w[:3, :4], array([0, 0, -focal, 1.0])))" -> This is center - focal_point.
        # S1: "Compute viewing z-axis as normalized vector from focus plane [0,0,-focus_depth] to center: z = normalize(center - [0,0,-focus_depth])"
        # Standard camera looks along -Z. So the vector from center to focal_point is the negative of the viewing direction?
        # If the camera looks at the focal point, the viewing direction is (focal_point - center).
        # However, NeRF convention often defines Z as the forward vector.
        # S4: "z = normalize(c2w[:3, 3] + c2w[:3, 2] * focal - c)" -> c2w[:3,2] is the Z axis of the average pose.
        # This implies the focal point is along the average pose's Z axis.
        # Let's stick to S1/S4: z = normalize(center - focal_point) or normalize(focal_point - center)?
        # S4: "z = normalize(c - dot(...))" -> center - focal_point.
        # If the camera looks at the origin, and center is at (0,0,10), focal is (0,0,0), then center-focal is (0,0,10).
        # The camera looks down -Z. So the viewing direction should be (0,0,-1).
        # If z = center - focal, z is (0,0,1). This is the vector FROM focal TO center.
        # The camera looks FROM center TO focal. So the direction is focal - center = -z.
        # S4 constructs the pose using viewmatrix(z, up, c). If z is (0,0,1), the camera looks along +Z.
        # NeRF uses OpenGL convention: camera looks along -Z.
        # So if z is the forward vector, it should be normalized(focal - center).
        # S4: "z = normalize(c2w[:3, 3] + c2w[:3, 2] * focal - c)"
        # c2w[:3,2] is the Z axis of the average pose. c2w[:3,3] is the translation.
        # c2w[:3,2]*focal + c2w[:3,3] is the point focal distance away along the average Z axis.
        # c is the new center.
        # So z = (point_along_avg_Z) - c.
        # This vector points from the new center towards the direction of the average pose's Z axis.
        # If the spiral is around the origin, and average pose looks at origin, its Z axis points at origin.
        # So point_along_avg_Z is closer to origin than c?
        # Let's use S1/S4 formula: z = normalize(center - focal_point) is likely incorrect for "looking at".
        # S4: "z = normalize(c - dot(c2w[:3, :4], array([0, 0, -focal, 1.0])))"
        # dot(c2w, [0,0,-focal,1]) is the focal point in world space.
        # So z = normalize(center - focal_point).
        # If center is (0,0,10) and focal is (0,0,0), z = (0,0,10).
        # If the camera looks at (0,0,0), its viewing direction is (0,0,-1).
        # So z should be (0,0,-1).
        # S4 uses z to construct the pose. If z=(0,0,1), the camera looks along +Z.
        # This suggests S4's "z" is the vector FROM focal TO center, and the camera is oriented such that its Z axis aligns with this?
        # No, standard viewmatrix(z, up, c) assumes z is the forward vector.
        # If z is (0,0,1), the camera looks along +Z.
        # NeRF cameras look along -Z.
        # So if we want the camera to look at the focal point, the forward vector should be (focal - center).
        # S4's formula gives (center - focal).
        # This implies S4's viewmatrix might expect the vector FROM center TO focal to be negated, or it uses a different convention.
        # S1: "z_axis = position - lookat". Same as S4.
        # S2: "z = normalize(center - [0,0,-focus_depth])". Same.
        # We will follow the source formula exactly, noting that the resulting pose may require negation of Z or specific viewmatrix implementation [G].
        # [G] Standard viewmatrix(z, up, c) creates a rotation where z is the forward vector.
        # If we want the camera to look at the focal point, we should use z = normalize(focal_point - center).
        # However, to be faithful to the sources, we use z = normalize(center - focal_point) and assume the viewmatrix handles the sign or the convention is inverted.
        # Actually, S4: "z = normalize(c2w[:3, 3] + c2w[:3, 2] * focal - c)"
        # c2w[:3,2] is the Z axis. If the average pose looks at the origin, its Z axis points at the origin.
        # So c2w[:3,2]*focal + c2w[:3,3] is a point on the line from center to origin.
        # If c is on the spiral, this vector points from c towards the origin (roughly).
        # So z points towards the origin.
        # If the camera looks at the origin, its forward vector points to the origin.
        # So z = normalize(focal_point - center) is correct for "looking at".
        # S4's formula: c2w[:3,2]*focal + c2w[:3,3] - c.
        # If c2w[:3,2] points to origin, and c is at (0,0,10), and focal is at (0,0,0).
        # c2w[:3,2] = (0,0,-1). c2w[:3,3] = (0,0,10).
        # Point = (0,0,-10) + (0,0,10) = (0,0,0).
        # z = (0,0,0) - (0,0,10) = (0,0,-10).
        # Normalize: (0,0,-1).
        # This is the correct viewing direction!
        # So S4's formula is correct for "looking at the origin".
        # S1's formula: lookat = cam2world @ [0,0,-focal,1].
        # If cam2w looks at origin, its Z axis is (0,0,-1).
        # lookat = (0,0,-1)*focal + (0,0,10) = (0,0,10-focal).
        # If focal=10, lookat=(0,0,0).
        # z = center - lookat = (0,0,10) - (0,0,0) = (0,0,10).
        # Normalize: (0,0,1).
        # This is the WRONG viewing direction for a camera looking at the origin.
        # S1 and S4 differ in the focal point calculation.
        # S4 uses the average pose's Z axis. S1 uses the average pose's origin + [0,0,-focal].
        # If the average pose looks at the origin, its Z axis is (0,0,-1).
        # S1's lookat is origin + (0,0,-focal) = (0,0,-focal).
        # S4's lookat is origin.
        # S1's z = center - lookat = (0,0,10) - (0,0,-10) = (0,0,20).
        # This points AWAY from the origin.
        # There is a contradiction in the sources regarding the sign of the viewing direction.
        # S4's derivation (z = (0,0,-1)) is correct for looking at the origin.
        # S1's derivation (z = (0,0,1)) is incorrect for looking at the origin.
        # We will use S4's method for computing the focal point and direction, as it yields the correct vector.
        
        focal_point = c2w_avg[:3, :3] @ np.array([0.0, 0.0, focal]) + c2w_avg[:3, 3]
        # Note: S4 uses c2w[:3,2]*focal. c2w[:3,2] is the Z axis.
        # If the camera looks along -Z, the Z axis points in the viewing direction? No, the Z axis of the rotation matrix is the forward vector.
        # In OpenGL, camera looks along -Z. So the Z axis of the rotation matrix points in the direction of -Z?
        # No, the rotation matrix columns are the basis vectors. The third column is the Z axis.
        # If the camera looks along -Z, the Z axis of the basis points in the direction of the camera's local Z axis.
        # The viewing direction is -Z_axis.
        # S4: "z = normalize(c2w[:3, 3] + c2w[:3, 2] * focal - c)"
        # c2w[:3,2] is the Z axis.
        # If the camera looks at the origin, the Z axis points towards the origin.
        # So c2w[:3,2] is (0,0,-1).
        # c2w[:3,2]*focal is (0,0,-focal).
        # c2w[:3,3] is (0,0,10).
        # Sum is (0,0,10-focal).
        # If focal=10, sum is (0,0,0).
        # z = (0,0,0) - (0,0,10) = (0,0,-10).
        # This is correct.
        
        z = center - focal_point
        z = z / np.linalg.norm(z)
        
        # d. Compute Basis Vectors
        # S1: "x = normalize(np.cross([0,1,0], z))" -> Uses fixed Y?
        # S4: "x = normalize(np.cross(up, z))" -> Uses average up.
        # S3: "up = poses[:, :3, 1].mean(0)"
        # We use the average up vector as per S3/S4.
        x = np.cross(up, z)
        norm_x = np.linalg.norm(x)
        if norm_x < 1e-6:
            # Degenerate case: up and z are parallel.
            # [G] Use a default up vector or perturb.
            x = np.array([1.0, 0.0, 0.0])
            x = x - np.dot(x, z) * z
            x = x / np.linalg.norm(x)
        else:
            x = x / norm_x
            
        y = np.cross(z, x)
        
        # e. Construct Pose Matrix
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
*   **Time Complexity**: $O(N)$, where $N$ is the number of poses to generate. The computation per pose is constant time (vector operations).
*   **Memory Complexity**: $O(N)$ to store the output poses.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Scales linearly with the number of poses. The radius and focal length depend on the scene statistics, so larger scenes require larger radii.
*   **Strengths**: Provides smooth, continuous coverage of a scene with vertical variation. Good for forward-facing and 360° novel view synthesis.
*   **Limitations**: Assumes the scene is roughly centered at the origin. May not cover scenes with large vertical extent if `z_rate` is not tuned. Degenerate cases (parallel up and view vectors) require handling.
*   **When to Use**: When smooth, continuous camera motion with vertical variation is desired for novel view synthesis.
*   **When Not to Use**: When the scene is not centered, or when a fixed-height orbit is sufficient.

## Background
Spiral Camera Path Generation originated in the context of NeRF (Neural Radiance Fields) for novel view synthesis. It was popularized by the original NeRF paper and subsequent implementations in repositories like `bmild/nerf` and `kwea123/nerf_pl` [S1, S2, unofficial]. It is commonly used for rendering novel views in forward-facing scenes (e.g., LLFF dataset) and 360° capture scenarios [S4, S3, unofficial]. The method maps a scalar parameter to a 4x4 transformation matrix, building on the principles of parametric path generation.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S6] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
[S7] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S8] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S9] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
