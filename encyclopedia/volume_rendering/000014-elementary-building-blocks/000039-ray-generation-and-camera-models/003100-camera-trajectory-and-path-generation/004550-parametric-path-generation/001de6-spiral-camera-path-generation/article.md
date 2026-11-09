# Spiral Camera Path Generation

Also called: Forward-Facing Spiral Camera Path, Novel View Synthesis Path, load_tankstemple_data spiral generation

## Overview
Spiral Camera Path Generation creates a sequence of camera poses along a spiral trajectory for novel view synthesis, primarily designed for forward-facing scenes where the camera orbits around a central point while oscillating vertically [S1, S3, S4, S5, S6, unofficial]. It differs from [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) by introducing a vertical z-axis oscillation component, and from [Circular Camera Path Generation](2734-circular-camera-path-generation.md) by varying the height over time rather than maintaining a fixed plane [S1, S3, unofficial]. It is distinct from [Spherify Poses](11414-spherify-poses.md) as it enforces a specific spiral geometry with vertical offset rather than a uniform spherical distribution [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `poses`: A list or array of training camera poses, each a $4 \times 4$ camera-to-world transformation matrix [S1, S3, S4, unofficial].
    *   `n_poses` (or `N`, `n_frames`): The number of camera poses to generate [S1, S3, S4, unofficial].
    *   `rots` (or `n_rots`): The number of full rotations ($2\pi$) to complete in the spiral [S1, S3, S4, unofficial].
    *   `zrate` (or `z_rate`): The rate of vertical oscillation in the spiral path [S1, S3, S4, unofficial].
    *   `focus_distance`: Weight for near/far bound averaging to compute focal length (default 0.75) [S6, unofficial].
    *   `spiral_scale_f`: Focal length multiplier (default 1.0) [S6, unofficial].
    *   `spiral_scale_r` (or `rads_scale`): Radius multiplier (default 1.0) [S6, S1, unofficial].
*   **Outputs**:
    *   `render_poses`: A list or array of generated $4 \times 4$ camera-to-world transformation matrices [S1, S3, S4, unofficial].

**Data Structures**
*   **Pose Matrix**: A $4 \times 4$ matrix where the upper-left $3 \times 3$ submatrix represents rotation, the rightmost column (first 3 elements) represents translation (camera center), and the bottom row is typically $[0, 0, 0, 1]$ [S1, S4, unofficial].
*   **Radii Array**: A 3-element array $[r_x, r_y, r_z]$ defining the spiral dimensions along each axis [S1, S3, S4, unofficial].

**Steps**
1.  **Compute Average Pose**: Calculate the average camera-to-world matrix $\mathbf{C}_{avg}$ from the input `poses` [S1, S3, S4, unofficial]. This serves as the center of the spiral [S5, unofficial].
    *   *Note*: Some implementations compute the centroid of camera positions and an average up vector separately [S8, unofficial].
2.  **Compute Radii**:
    *   Extract camera positions $\mathbf{p}_i$ from `poses` (the translation components of $\mathbf{C}_{avg}$ or individual poses) [S1, S3, S4, unofficial].
    *   Compute the radius vector $\mathbf{r}$ as the 90th percentile of the absolute values of these positions along each axis, scaled by `spiral_scale_r` [S1, S3, S4, unofficial].
    *   Append $1.0$ to $\mathbf{r}$ to form a homogeneous scaling vector $\mathbf{r}_{hom} = [r_x, r_y, r_z, 1.0]$ [S1, S3, S4, unofficial].
3.  **Compute Focal Length**:
    *   Determine near and far bounds from the scene geometry (e.g., minimum and maximum depth of training cameras) [S6, unofficial].
    *   Calculate the focal length $f$ using the harmonic mean weighted by `focus_distance`:
        $$ f = \frac{1}{\frac{1 - \text{focus\_distance}}{\text{near\_bound}} + \frac{\text{focus\_distance}}{\text{far\_bound}}} \times \text{spiral\_scale\_f} $$
        [S6, unofficial].
    *   *Alternative*: Some sources compute focal length as the mean norm of camera positions or extract it from the input camera intrinsics [S4, S8, unofficial].
4.  **Generate Poses**:
    *   Initialize an empty list `render_poses` [S1, S3, S4, unofficial].
    *   Iterate $\theta$ from $0$ to $2\pi \times \text{rots}$ with `n_poses` steps, excluding the final point to avoid duplication with the start [S1, S3, S4, unofficial].
    *   For each $\theta$, compute the offset vector $\mathbf{t}$ in the local coordinate system of $\mathbf{C}_{avg}$:
        $$ \mathbf{t} = \mathbf{r}_{hom} \odot [\cos(\theta), -\sin(\theta), -\sin(\theta \times \text{zrate}), 1.0] $$
        [S1, S3, S4, S5, unofficial].
        *   Note: The sign of the sine terms may vary by implementation (e.g., $-\sin(\theta)$ vs $\sin(\theta)$) to match the desired handedness [S1, S3, unofficial].
    *   Compute the camera center $\mathbf{c}$ in world coordinates:
        $$ \mathbf{c} = \mathbf{C}_{avg} \times \mathbf{t} $$
        [S1, S3, S4, S5, unofficial].
        *   Specifically, $\mathbf{c} = \mathbf{C}_{avg}[:3, :3] \times \mathbf{t}[:3] + \mathbf{C}_{avg}[:3, 3]$ [S5, unofficial].
    *   Compute the look-at direction (z-axis). The camera looks towards a focal point defined by the average pose's origin and focal length [S1, S3, S4, unofficial].
        *   Define the focal point $\mathbf{f}_{pt}$ in world coordinates:
            $$ \mathbf{f}_{pt} = \mathbf{C}_{avg} \times [0, 0, -f, 1]^T $$
            [S1, S3, S4, unofficial].
        *   Compute the viewing direction $\mathbf{z}$:
            $$ \mathbf{z} = \text{normalize}(\mathbf{f}_{pt} - \mathbf{c}) $$
            [S1, S3, S4, S5, unofficial].
            *   *Note*: Some sources define $\mathbf{z}$ as $\text{normalize}(\mathbf{c} - \mathbf{f}_{pt})$ and negate later, or compute it as $\text{normalize}(\mathbf{C}_{avg}[:3, 3] + \mathbf{C}_{avg}[:3, 2] \times f - \mathbf{c})$ [S4, S5, unofficial].
    *   Compute the up vector $\mathbf{u}$. This is typically the mean of the up vectors (second column) of the input `poses` [S1, S3, S4, unofficial].
        $$ \mathbf{u} = \text{mean}(\text{poses}[:, :3, 1]) $$
        [S1, S3, S4, unofficial].
    *   Construct the rotation matrix for the new pose:
        *   Compute $\mathbf{x}$-axis:
            $$ \mathbf{x} = \text{normalize}(\mathbf{u} \times \mathbf{z}) $$
            [S1, S3, S4, S5, unofficial].
            *   *Note*: Some implementations use a fixed world up vector $[0, 1, 0]$ to compute $\mathbf{x}$ if the camera up is unstable [S9, unofficial].
        *   Compute $\mathbf{y}$-axis:
            $$ \mathbf{y} = \mathbf{z} \times \mathbf{x} $$
            [S1, S3, S4, S5, unofficial].
    *   Construct the $4 \times 4$ pose matrix $\mathbf{P}$:
        $$ \mathbf{P} = \begin{bmatrix} \mathbf{x} & \mathbf{y} & \mathbf{z} & \mathbf{c} \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
        [S1, S3, S4, S5, unofficial].
    *   Append $\mathbf{P}$ to `render_poses` [S1, S3, S4, unofficial].
5.  **Return**: Stack `render_poses` into a single array and return [S1, S3, S4, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_spiral_path(poses, n_poses, rots, zrate, focus_distance=0.75, spiral_scale_f=1.0, spiral_scale_r=1.0):
    """
    Generate a spiral camera path for novel view synthesis.
    
    Args:
        poses: np.ndarray of shape (N, 4, 4), training camera poses.
        n_poses: int, number of poses to generate.
        rots: int, number of rotations.
        zrate: float, vertical oscillation rate.
        focus_distance: float, weight for near/far bound averaging.
        spiral_scale_f: float, focal length multiplier.
        spiral_scale_r: float, radius multiplier.
        
    Returns:
        render_poses: np.ndarray of shape (n_poses, 4, 4).
    """
    # Step 1: Compute average pose
    # Average rotation and translation
    avg_rot = np.mean(poses[:, :3, :3], axis=0)
    # Normalize rotation to ensure it is a valid rotation matrix (optional but recommended)
    # For simplicity, we assume inputs are well-formed or use SVD if needed.
    # Here we just use the mean translation for the center.
    avg_trans = np.mean(poses[:, :3, 3], axis=0)
    
    # Construct average pose matrix
    c2w_avg = np.eye(4)
    c2w_avg[:3, :3] = avg_rot
    c2w_avg[:3, 3] = avg_trans
    
    # Step 2: Compute Radii
    positions = poses[:, :3, 3]
    radii = np.percentile(np.abs(positions), 90, axis=0) * spiral_scale_r
    rads = np.append(radii, 1.0)  # [rx, ry, rz, 1.0]
    
    # Step 3: Compute Focal Length
    # Estimate near and far bounds from training camera depths
    # Assuming 'positions' are relative to some origin, or we use the norm of positions
    # A common heuristic in NeRF implementations:
    # near_bound = min distance from origin to planes? 
    # Source [S6, unofficial] uses bounds.min() and bounds.max(). Let's assume we have scene bounds.
    # If not given, we approximate using the mean distance or min/max of positions.
    # For implementation completeness, we use the norm of positions as a proxy for depth if bounds are not explicit.
    # However, [S6, unofficial] explicitly computes focal from near/far bounds.
    # Let's assume we compute near/far from the projection of positions onto the average view direction.
    # Simplified: Use the mean norm of positions as a scale, or specific bounds if available.
    # To be precise to [S6, unofficial]:
    # near_bound = np.min(np.linalg.norm(positions, axis=1)) # Placeholder if bounds not provided
    # far_bound = np.max(np.linalg.norm(positions, axis=1))
    # But [S6, unofficial] says bounds.min() and bounds.max(). Let's assume we have a 'bounds' object or compute it.
    # Since 'bounds' is not an input, we derive it from positions.
    # A robust way: project positions onto the average forward vector.
    avg_forward = c2w_avg[:3, 2] # Assuming -z is forward in camera space, but c2w maps cam->world.
    # Actually, c2w[:3, 2] is the world direction of the camera's Z axis.
    # Let's use the mean distance from the centroid as a scale factor for focal if bounds are ambiguous.
    # Source [S4, unofficial] uses mean norm. Source [S6, unofficial] uses near/far.
    # We will follow [S6, unofficial] logic but estimate bounds from positions.
    # Let's assume the scene is centered at 0.
    dists = np.linalg.norm(positions, axis=1)
    near_bound = np.min(dists)
    far_bound = np.max(dists)
    
    focal = (1.0 / ((1 - focus_distance) / near_bound + focus_distance / far_bound)) * spiral_scale_f
    
    # Step 4: Generate Poses
    render_poses = []
    
    # Compute average up vector
    up = np.mean(poses[:, :3, 1], axis=0)
    up = up / np.linalg.norm(up)
    
    theta = np.linspace(0, 2 * np.pi * rots, n_poses, endpoint=False)
    
    for t in theta:
        # Compute offset vector t_vec
        # [cos(t), -sin(t), -sin(t * zrate), 1.0]
        t_vec = np.array([np.cos(t), -np.sin(t), -np.sin(t * zrate), 1.0])
        
        # Scale by radii
        t_scaled = t_vec * rads
        
        # Compute camera center c
        # c = c2w_avg @ t_scaled
        c = c2w_avg[:3, :3] @ t_scaled[:3] + c2w_avg[:3, 3]
        
        # Compute look-at direction z
        # Focal point in world space: c2w_avg @ [0, 0, -focal, 1]
        focal_point_world = c2w_avg[:3, :3] @ np.array([0, 0, -focal]) + c2w_avg[:3, 3]
        
        z = focal_point_world - c
        z = z / np.linalg.norm(z)
        
        # Compute x axis
        x = np.cross(up, z)
        x = x / np.linalg.norm(x)
        
        # Compute y axis
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

**Edge Cases and Pitfalls**
*   **Precision**: The focal length calculation involves division by near/far bounds. If near bound is close to zero, precision issues may arise. Clamping near bound to a small epsilon is recommended [G].
*   **Boundaries**: The spiral path should not include the final point if it duplicates the start point (i.e., use `endpoint=False` in linspace) [S1, S3, S4, unofficial].
*   **Degenerate Input**: If training poses are collinear or have zero variance in one axis, the 90th percentile radius may be zero, leading to a degenerate spiral. Adding a small epsilon to radii or using a minimum radius threshold is recommended [G].
*   **Up Vector**: The average up vector may not be orthogonal to the viewing direction. The cross product step ensures orthogonality, but if the up vector is parallel to the viewing direction, the cross product will be zero. Using a fixed world up vector (e.g., $[0, 1, 0]$) as a fallback is a common safeguard [S9, G, unofficial].

## Subtopics

### Spiral Camera Pose Generation
A variant that generates camera poses in a circular path with optional vertical offset, often used in differentiable rendering frameworks [S9, unofficial]. It differs from the standard spiral by using a simpler circular trajectory in the x-z plane with a fixed or sinusoidal y-offset, and constructs poses using `look_at_view_transform` [S9, unofficial]. It is chosen when a simpler, planar or slightly elevated orbit is sufficient, or when integrating with libraries like PyTorch3D [S9, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of poses to generate, as each pose is computed in constant time [S1, S3, S4, unofficial].
*   **Memory Complexity**: $O(N)$ to store the generated poses [S1, S3, S4, unofficial].
*   **Performance**: The generation is extremely fast, typically taking milliseconds for hundreds of poses [S1, S3, S4, unofficial].
*   **Strengths**: Provides smooth, continuous camera motion suitable for video rendering; adapts to the scale of the input scene via percentile-based radii [S1, S3, S4, unofficial].
*   **Limitations**: Assumes a roughly central scene distribution; may produce poor results for highly asymmetric or elongated scenes without adjustment [S1, S3, unofficial]. The vertical oscillation rate (`zrate`) is a hyperparameter that may need tuning for specific scenes [S1, S3, S4, unofficial].
*   **When to Use**: Forward-facing novel view synthesis, especially for datasets like LLFF or Tank-And-Temple where the camera orbits the scene [S1, S3, S4, unofficial].
*   **When Not to Use**: Scenes with significant depth variation where a fixed focal plane is inappropriate, or when a strict spherical or elliptical path is required [S1, S3, S4, S8, unofficial].

## Background
Spiral camera paths were introduced in the context of NeRF (Neural Radiance Fields) for novel view synthesis, particularly for forward-facing scenes [S1, S3, S4, unofficial]. The method was popularized by the original NeRF paper and subsequent implementations in repositories like `bmild/nerf` and `kwea123/nerf_pl` [S1, S2, S3, S4, unofficial]. It is widely used in datasets such as LLFF and Tank-And-Temple for generating training and evaluation views [S1, S3, S4, unofficial].

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
