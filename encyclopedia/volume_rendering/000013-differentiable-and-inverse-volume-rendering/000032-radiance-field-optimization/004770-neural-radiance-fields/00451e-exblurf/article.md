# ExBluRF

Also called: Efficient Radiance Fields for Extreme Motion Blurred Images

## Overview
ExBluRF is a novel view synthesis method designed to reconstruct sharp 3D scenes from extreme motion-blurred images by jointly optimizing voxel-based radiance fields and 6-DOF camera trajectories [S1]. It distinguishes itself from sibling methods like [NeRF](1185-nerf-neural-radiance-fields.md) and [DeblurNeRF](6202-neural-reconstruction-in-the-wild.md) by replacing the memory-intensive MLP-based radiance field with a voxel grid and modeling motion blur as an accumulation of rays along a Bézier-curve-defined camera trajectory, rather than using 2D spatially varying blur kernels [S1]. This approach allows for a significantly higher number of sampling points along the trajectory without proportional increases in memory or computation, enabling the restoration of high-quality sharp views from severely degraded inputs [S1].

## How it works
**Inputs:** A set of $M$ extremely motion-blurred images $\{I_m\}_{m=1}^M$ with known camera intrinsics $K$ and initial extrinsic poses (e.g., from COLMAP) [S1].
**Outputs:** A sharp voxel-based radiance field (density $\sigma$ and color $\mathbf{c}$) and the optimized 6-DOF camera trajectory parameters for each input image [S1].

**Data Structures:**
1.  **Voxel Grid:** A 3D grid storing volumetric density $\sigma$ and color $\mathbf{c}$ at each voxel. This replaces the MLP $f_{NeRF}$ used in standard NeRF [S1]. The grid supports trilinear interpolation for direct evaluation at any 3D coordinate [S1].
2.  **Bézier Curve Control Points:** For each image $m$, a set of control points $\{\hat{\mathbf{p}}_j\}_{j=0}^M$ in the Lie algebra $\mathfrak{se}(3)$ defining the 6-DOF camera motion trajectory. $M$ is the order of the Bézier curve [S1].
3.  **Ray Batch:** A batch of $B$ rays, where each ray corresponds to a pixel $x$ and an intermediate shutter time $\tau_i$ [S1].

**Parameters:**
*   $N$: Number of sampling points (intermediate sub-frames) along the camera trajectory to approximate the motion blur integral. Typical values range from 5 to 31 [S1].
*   $P$: Number of sample points on a single ray during volume rendering [S1].
*   $L$: Number of layers in the MLP (used for comparison with DeblurNeRF; ExBluRF uses a voxel grid, so $L$ is not applicable to the radiance field itself) [S1].
*   $M$: Order of the Bézier curve. Typical value is 3 for challenging motions [S1].
*   $\phi_{voxel}$: Parameters of the voxel grid [S1].

**Algorithm Steps:**

1.  **Trajectory Reparameterization:** For each input image $m$, define the initial pose $P_{\tau_o}$ and final pose $P_{\tau_c}$. Reparameterize the 6-DOF motion trajectory using a Bézier curve of order $M$ in the Lie algebra $\mathfrak{se}(3)$ [S1]. The pose at normalized time $\tau'_i$ is computed as:
    $$
    \mathbf{p}_{\tau_i} = \sum_{j=0}^{M} \binom{M}{j} (1-\tau'_i)^{M-j} (\tau'_i)^j \cdot \hat{\mathbf{p}}_j
    $$
    where $\tau'_i = \frac{\tau_i - \tau_o}{\tau_c - \tau_o}$ and $\tau_i = \tau_o + \frac{i-1}{N-1}(\tau_c - \tau_o)$ for $i=1 \dots N$ [S1]. The matrix pose $P_{\tau_i}$ is obtained by exponentiating $\mathbf{p}_{\tau_i}$ [S1].

2.  **Ray Generation:** For each blurred input image $m$ and each pixel $x$ in that image, generate $N$ rays $\{\mathbf{r}_{\tau_i}(x)\}_{i=1}^N$ corresponding to the intermediate poses $P_{\tau_i}$ [S1]. The direction $\mathbf{d}_{\tau_i}$ of a ray at pixel $x$ is calculated using the camera intrinsic $K$ and the extrinsic rotation $R_{\tau_i}$ from $P_{\tau_i}$:
    $$
    \mathbf{d}_{\tau_i} = R_{\tau_i} \cdot K^{-1} [\mathbf{x}]_+
    $$
    where $[\mathbf{x}]_+$ is the homogeneous coordinate of pixel $x$ [S1]. The origin of the ray is the translation component of $P_{\tau_i}$ [S1].

3.  **Volume Rendering:** For each ray $\mathbf{r}_{\tau_i}(x)$, compute the rendered color $\hat{\mathbf{C}}(\mathbf{r}_{\tau_i}(x))$ using the standard volume rendering equation with the voxel-based radiance field [S1]:
    $$
    \hat{\mathbf{C}}(\mathbf{r}) = \sum_{k=1}^{P} T_k (1 - \exp(-\sigma_k \delta_k)) \mathbf{c}_k
    $$
    where $\sigma_k$ and $\mathbf{c}_k$ are the density and color sampled at the $k$-th point along the ray, $\delta_k$ is the distance between sample points, and $T_k = \exp(-\sum_{j=1}^{k-1} \sigma_j \delta_j)$ is the transmittance [S1]. The density and color are obtained by trilinear interpolation from the voxel grid [S1].

4.  **Blur Accumulation:** Approximate the motion blur integral for pixel $x$ by averaging the rendered colors of the $N$ rays along the trajectory [S1]:
    $$
    \hat{\mathbf{C}}_B(x) \approx \frac{1}{N} \sum_{i=1}^{N} \hat{\mathbf{C}}(\mathbf{r}_{\tau_i}(x))
    $$
    This $\hat{\mathbf{C}}_B(x)$ is the predicted blurred color for pixel $x$ [S1].

5.  **Loss Computation and Optimization:** Compute the photo-consistency loss (e.g., $L_1$ or $L_2$) between the predicted blurred image $\hat{\mathbf{C}}_B$ and the input blurred image $I_m$ [S1]. Backpropagate the gradients to optimize both the voxel grid parameters $\phi_{voxel}$ and the Bézier control points $\{\hat{\mathbf{p}}_j\}$ for all input images [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_exblurf(voxel_grid, camera_intrinsics, control_points, pixel_coords, num_samples_trajectory):
    """
    Renders a single blurred image from a set of blurred inputs using ExBluRF.
    
    Args:
        voxel_grid: Voxel grid storing density and color. Supports trilinear interpolation.
        camera_intrinsics: Matrix K.
        control_points: List of M+1 control points in se(3) Lie algebra for the Bézier curve.
        pixel_coords: Array of pixel coordinates (u, v) for the image.
        num_samples_trajectory: N, number of intermediate sub-frames.
        
    Returns:
        blurred_image: Rendered blurred image of shape (H, W, 3).
    """
    H, W = pixel_coords.shape[:2]
    blurred_image = np.zeros((H, W, 3))
    
    # 1. Generate intermediate poses using Bézier curve
    # Normalize time for N samples
    tau_prime = np.linspace(0, 1, num_samples_trajectory)
    
    # Compute poses for each tau_prime
    # Note: In practice, this involves Lie algebra exponential map
    poses = []
    for t in tau_prime:
        pose_se3 = np.zeros(6)
        for j in range(len(control_points)):
            # Bernstein polynomial coefficient
            coeff = comb(len(control_points)-1, j) * ((1-t)**(len(control_points)-1-j)) * (t**j)
            pose_se3 += coeff * control_points[j]
        # Exponential map from se(3) to SE(3) to get 4x4 matrix
        pose_matrix = exp_se3_to_se3(pose_se3)
        poses.append(pose_matrix)
        
    # 2. For each pixel, accumulate rays along the trajectory
    for y in range(H):
        for x in range(W):
            pixel = pixel_coords[y, x]
            accumulated_color = np.zeros(3)
            
            for pose in poses:
                # Extract R and t from pose matrix
                R = pose[:3, :3]
                t = pose[:3, 3]
                
                # 3. Ray direction
                # d = R * K^-1 * [x, y, 1]^T
                hom_coord = np.array([x, y, 1.0])
                inv_K_hom = np.linalg.inv(camera_intrinsics) @ hom_coord
                d = R @ inv_K_hom
                d = d / np.linalg.norm(d) # Normalize direction
                
                # 4. Ray origin
                origin = t
                
                # 5. Volume Rendering along ray
                # Sample P points along the ray
                # Assume near and far planes are defined or inferred
                # For simplicity, assume standard NeRF sampling strategy
                samples = sample_ray(origin, d, num_samples_ray=P)
                
                density_sum = 0.0
                color_acc = np.zeros(3)
                
                for k, (pos, dist_to_next) in enumerate(samples):
                    # Interpolate density and color from voxel grid
                    sigma, c = voxel_grid.interpolate(pos)
                    
                    # Alpha and transmittance
                    alpha = 1.0 - np.exp(-sigma * dist_to_next)
                    T = np.exp(-density_sum)
                    
                    color_acc += T * alpha * c
                    density_sum += sigma * dist_to_next
                    
                accumulated_color += color_acc
                
            # Average over N samples
            blurred_image[y, x] = accumulated_color / num_samples_trajectory
            
    return blurred_image

def train_exblurf(blurred_images, voxel_grid_init, camera_intrinsics, num_samples_trajectory, num_iterations):
    """
    Jointly optimizes voxel grid and camera trajectories.
    """
    # Initialize voxel grid
    voxel_grid = voxel_grid_init
    
    # Initialize control points for each image (e.g., from initial poses)
    # Each image has its own set of control points
    control_points_list = [init_control_points_from_pose(initial_pose) for initial_pose in initial_poses]
    
    optimizer = Adam([voxel_grid.parameters(), control_points_list])
    
    for iteration in range(num_iterations):
        total_loss = 0.0
        
        for i, blurred_img in enumerate(blurred_images):
            # Render the blurred image for current view
            rendered_blurred = render_exblurf(
                voxel_grid, 
                camera_intrinsics, 
                control_points_list[i], 
                get_pixel_coords(blurred_img.shape), 
                num_samples_trajectory
            )
            
            # Compute loss
            loss = L1Loss(rendered_blurred, blurred_img)
            total_loss += loss
            
        # Backpropagate
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
    return voxel_grid, control_points_list
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on accurate Lie algebra operations for pose interpolation. Incorrect exponential maps can lead to non-smooth trajectories [S1].
*   **Boundaries:** The voxel grid must cover the entire scene extent. If the camera moves significantly outside the initial bounding box, extrapolation errors may occur [S1].
*   **Degenerate Input:** If the blur is too extreme or the motion is too fast, the assumption of a smooth Bézier curve may fail. Using a low-order (1st order) Bézier curve for complex hand-shake motions leads to poor performance [S1].
*   **Memory:** While ExBluRF is more memory-efficient than DeblurNeRF, increasing $N$ (number of trajectory samples) still increases computation time linearly, though not memory [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** ExBluRF has $O(1)$ memory complexity per ray because it sequentially evaluates sample points on the ray [S1]. The total memory usage is constant with respect to the number of trajectory samples $N$, unlike DeblurNeRF which has $O(P L)$ memory cost per ray due to synchronous feed-forwarding and activation storage [S1]. The base memory cost is determined by the voxel grid size $\phi_{voxel}$ [S1].
*   **Time:** Training time and computation increase linearly with $N$ (number of trajectory samples) because $N$ rays are rendered per pixel [S1]. However, the constant factor is significantly lower than MLP-based methods [S1].

**Measured Performance:**
*   On the 'Camellia' dataset (ExBlur), ExBluRF ($N=5$) achieves 27.50 PSNR with 4.31 GB GPU memory and 4.41 hours training time [S1].
*   DeblurNeRF ($N=5$) achieves 23.95 PSNR with 3.55 GB GPU memory and 5.17 hours training time [S1].
*   DeblurNeRF ($N=21$) achieves 27.76 PSNR with 13.89 GB GPU memory and 27.48 hours training time [S1].
*   ExBluRF ($N=21$) achieves 28.83 PSNR with 4.31 GB GPU memory and 4.41 hours training time [S1].
*   ExBluRF is approximately 10x faster and uses 10x less GPU memory than DeblurNeRF for comparable or better quality [S1].
*   On the 'Postbox' scene, increasing $N$ from 5 to 31 increases PSNR from 28.62 to 30.23 [S1].
*   On the 'Pool' scene, increasing the Bézier curve order from 1 to 3 increases PSNR from 25.30 to 29.65 [S1].

**Strengths and Limitations:**
*   **Strengths:** Handles extreme motion blur efficiently; allows large $N$ for sharper results; joint optimization of geometry and camera pose; robust to hand-shake motions with higher-order Bézier curves [S1].
*   **Limitations:** Requires per-scene optimization; sensitive to the order of the Bézier curve (overfitting possible with very high orders); initial pose estimation is still required [S1].

**When to Use:**
*   Use ExBluRF when dealing with extreme motion blur where 2D deblurring priors fail or when high-quality sharp novel views are required from severely degraded inputs [S1].
*   Use when computational resources are limited, but high accuracy is needed, as it scales better with blur intensity than MLP-based methods [S1].

**When Not to Use:**
*   Do not use if the motion is simple and linear, as a 1st-order Bézier curve may suffice, but ExBluRF's complexity might be overkill compared to simpler methods [S1].
*   Do not use if real-time rendering is required, as it is an optimization-based method [S1].

## Background
ExBluRF was introduced by Dongwoo Lee, Sunghyun Cho, Kyoung Mu Lee, and Jin Han Rim in 2023 [S1]. It addresses the limitations of previous NeRF-based deblurring methods like DeblurNeRF, which struggled with extreme motion blur due to memory and computation bottlenecks associated with MLPs and 2D blur kernels [S1]. The method builds on the principles of differentiable volume rendering [44-radiance-field-optimization.md] and voxel-based radiance fields [10852-voxel-based-representation.md]. It was validated on the ExBlur dataset, which provides multi-view blurred images with ground truth sharp pairs and trajectories [S1].

## Sources
[S1] ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images, https://arxiv.org/abs/2309.08957v3, fetched 2026-10-27
