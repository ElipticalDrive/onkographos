# 3D Gaussian Splatting MVP

Also called: 3DGS MVP, Adaptive Surface Splatting Using Elliptical Gaussians, Differentiable 3D Gaussian Splatting, pure PyTorch 3DGS

## Overview
3D Gaussian Splatting MVP is a self-contained, pure-PyTorch implementation of the core 3D Gaussian Splatting algorithm, designed as a teaching reference rather than a production renderer [S1, unofficial]. It renders 3D Gaussians onto screen space and rasterizes them using alpha blending to produce smooth, hole-free images from arbitrary viewpoints [S2]. It sets itself apart from production implementations by intentionally omitting tile-based CUDA rasterization, spherical harmonics for view-dependent color, and adaptive density control, relying instead on a readable Python loop for the forward pass [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians defined by their parameters; a pinhole camera model (look_at, OpenCV/COLMAP convention) [S1, unofficial].
*   **Outputs:** A rendered RGB image of resolution $H \times W$ [S1, unofficial].

**Data Structures:**
Each Gaussian $k$ is parameterized by 11 values [S2]:
1.  **Mean:** $\mathbf{p}_k = (x, y, z)$ in world space.
2.  **Scale:** $\mathbf{s}_k = (\sigma_x, \sigma_y, \sigma_z)$ stored in log-space [S1, unofficial].
3.  **Rotation:** $\mathbf{q}_k = (q_w, q_x, q_y, q_z)$ as a unit quaternion [S1, unofficial].
4.  **Color:** $\mathbf{c}_k = (r, g, b)$ per-Gaussian RGB [S1, unofficial].
5.  **Opacity:** $o_k$ (scalar) [S1, unofficial].

The covariance matrix $\Sigma_k$ is constructed from scale and rotation to ensure it is symmetric positive semi-definite (PSD) [S1, unofficial]:
$$ \Sigma_k = R S S^T R^T $$
where $R$ is the rotation matrix derived from $\mathbf{q}_k$, and $S = \text{diag}(\mathbf{s}_k)$ [S1, unofficial].

**Algorithm Steps:**

1.  **World-to-Camera Transformation:**
    Transform the Gaussian mean $\mathbf{p}_k$ from world space to camera space using the viewing transformation $W$ [S1, unofficial]. The viewing transformation $W$ is defined by camera rotation $T$ and translation $t$ [S2]:
    $$ W = \begin{bmatrix} T & t \\ 0 & 1 \end{bmatrix} $$
    Let $\mathbf{p}'_k$ be the transformed mean in camera space [S1, unofficial].

2.  **3D Covariance Transformation:**
    Transform the 3D covariance matrix $\Sigma_k$ into the camera frame [S1, unofficial]:
    $$ \Sigma'_k = W_{3\times3} \Sigma_k W_{3\times3}^T $$
    where $W_{3\times3}$ is the upper-left $3\times3$ rotation submatrix of $W$ [S1, unofficial].

3.  **2D Screen-Space Projection:**
    Project $\Sigma'_k$ to 2D screen space using the EWA / Zwicker-style projection Jacobian $J$ [S1, unofficial]. The screen-space covariance $\Sigma_k^{(s)}$ is approximated as [S2]:
    $$ \Sigma_k^{(s)} = J \Sigma'_k J^T $$
    where $J$ is the Jacobian of the perspective projection [S2]. For a point with camera-space coordinates $(x, y, z)$ and focal lengths $f_x, f_y$, $J$ is given by [S2]:
    $$ J = \begin{bmatrix} f_x/z & 0 & -f_x x / z^2 \\ 0 & f_y/z & -f_y y / z^2 \end{bmatrix} $$
    A small isotropic dilation/blur is added to $\Sigma_k^{(s)}$ for numerical stability [S1, unofficial].

4.  **Splat Boundary Determination:**
    Determine the bounding box for each splat in screen space. A threshold of three times the standard deviation of the Gaussian is used to define the boundaries [S2].

5.  **Depth Sorting:**
    Sort the set of splats $U$ covering the image by depth (the $z$ coordinate in camera space) in front-to-back order [S2]. Note: The implementation uses `argsort`, which is non-differentiable across order changes; gradients flow through the values given the fixed ordering [S1, unofficial].

6.  **Per-Pixel Alpha Compositing:**
    For each pixel $\mathbf{x}$, identify the set of splats $U = \{ \langle G(\mathbf{p}'_k + \delta_k, \Sigma_k^{(s)}), o_k, \mathbf{c}_k \rangle \mid k=1,\dots,K \}$ that cover the pixel [S2].
    
    Calculate the opacity $\alpha_k$ of the $k$-th splat at pixel $\mathbf{x}$ by evaluating the 2D Gaussian density, weighted by the estimated opacity $o_k$ [S2]:
    $$ \alpha_k = o_k \cdot e^{-\frac{1}{2} \mathbf{d}_k^T (\Sigma_k^{(s)})^{-1} \mathbf{d}_k} $$
    where $\mathbf{d}_k$ is the distance vector between the pixel center and the projected Gaussian center in screen space [S2].

    Compute the final color $C(\mathbf{x})$ using front-to-back alpha blending [S2]:
    $$ C(\mathbf{x}) = \sum_{k=1}^{K} T_k \alpha_k \mathbf{c}_k $$
    where $T_k = \prod_{j=1}^{k-1} (1 - \alpha_j)$ is the accumulated transmittance [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_gaussians_mvp(gaussians, camera_params, image_h, image_w):
    """
    Pure PyTorch implementation of 3D Gaussian Splatting MVP.
    
    Args:
        gaussians: Dict with keys 'mean' (N,3), 'scale' (N,3), 'rot' (N,4), 
                   'color' (N,3), 'opacity' (N,1). All in world space.
        camera_params: Dict with 'R_cam' (3,3), 'T_cam' (3,). 
                       Defines world-to-camera transformation.
        image_h, image_w: Output image dimensions.
        
    Returns:
        rendered_image: (H, W, 3) tensor.
    """
    N = gaussians['mean'].shape[0]
    
    # 1. World-to-Camera Transformation
    # p_cam = R_cam @ p_world + T_cam
    means_cam = (camera_params['R_cam'] @ gaussians['mean'].T).T + camera_params['T_cam']
    
    # 2. Construct 3D Covariance Matrices
    # Sigma = R * S * S^T * R^T
    # R from quaternion
    Rs = []
    for i in range(N):
        q = gaussians['rot'][i] # qw, qx, qy, qz
        R = quaternion_to_rotation_matrix(q)
        Rs.append(R)
    Rs = torch.stack(Rs) # (N, 3, 3)
    
    # S from log-scale
    scales = torch.exp(gaussians['scale']) # (N, 3)
    S = torch.zeros(N, 3, 3)
    for i in range(N):
        S[i] = torch.diag(scales[i])
        
    # Sigma_3d = R S S^T R^T
    Sigma_3d = torch.bmm(torch.bmm(Rs, S), torch.bmm(S, Rs.transpose(1, 2)))
    
    # 3. Transform Covariance to Camera Frame
    # Sigma_cam = R_cam @ Sigma_3d @ R_cam^T
    Sigma_cam = torch.bmm(torch.bmm(camera_params['R_cam'].unsqueeze(0).expand(N, -1, -1), Sigma_3d),
                          camera_params['R_cam'].unsqueeze(0).expand(N, -1, -1).transpose(1, 2))
                          
    # 4. Project to Screen Space
    # Jacobian J depends on camera-space coordinates (x, y, z) and focal lengths
    # Assume focal lengths fx, fy are part of camera_params or derived from FOV
    fx = camera_params.get('fx', 1.0)
    fy = camera_params.get('fy', 1.0)
    
    xs = means_cam[:, 0]
    ys = means_cam[:, 1]
    zs = means_cam[:, 2]
    
    # J matrix construction
    # J = [[fx/z, 0, -fx*x/z^2],
    #      [0, fy/z, -fy*y/z^2]]
    J = torch.zeros(N, 2, 3)
    J[:, 0, 0] = fx / zs
    J[:, 0, 2] = -fx * xs / (zs**2)
    J[:, 1, 1] = fy / zs
    J[:, 1, 2] = -fy * ys / (zs**2)
    
    # Sigma_2d = J @ Sigma_cam @ J^T
    Sigma_2d = torch.bmm(torch.bmm(J, Sigma_cam), J.transpose(1, 2))
    
    # Add small isotropic blur for stability
    Sigma_2d[:, 0, 0] += 0.3
    Sigma_2d[:, 1, 1] += 0.3
    
    # 5. Depth Sorting
    # Sort by z-coordinate (means_cam[:, 2])
    depths = means_cam[:, 2]
    sort_indices = torch.argsort(depths, dim=0)
    
    # Gather sorted parameters
    sorted_means = means_cam[sort_indices]
    sorted_Sigma_2d = Sigma_2d[sort_indices]
    sorted_colors = gaussians['color'][sort_indices]
    sorted_opacities = gaussians['opacity'][sort_indices]
    
    # 6. Rasterization
    # Initialize output image
    image = torch.zeros(image_h, image_w, 3, device=gaussians['mean'].device)
    transmittance = torch.ones(image_h, image_w, 1, device=gaussians['mean'].device)
    
    # Iterate over Gaussians (front to back)
    for k in range(N):
        mean_k = sorted_means[k]
        sigma_k = sorted_Sigma_2d[k]
        color_k = sorted_colors[k]
        opac_k = sorted_opacities[k]
        
        # Determine bounding box for this splat
        # Threshold: 3 * std_dev. Eigenvalues of Sigma_2d give variances.
        eigvals = torch.linalg.eigvals(sigma_k)
        std_x = torch.sqrt(torch.max(eigvals.real[0], torch.tensor(1e-6)))
        std_y = torch.sqrt(torch.max(eigvals.real[1], torch.tensor(1e-6)))
        
        # Project mean to screen space
        # Assuming standard pinhole projection: x_screen = fx * X_cam / Z_cam
        # This part requires mapping camera coords to pixel coords.
        # For MVP, we assume a simple projection or that the Jacobian logic 
        # implicitly handles the coordinate mapping for the density evaluation.
        # To be precise, we need the pixel coordinates of the center.
        # Let's assume camera_params includes projection matrix or we compute pixel coords.
        # px = fx * X / Z + cx
        # py = fy * Y / Z + cy
        cx = camera_params.get('cx', image_w / 2)
        cy = camera_params.get('cy', image_h / 2)
        
        px = fx * mean_k[0] / mean_k[2] + cx
        py = fy * mean_k[1] / mean_k[2] + cy
        
        # Define bounding box in pixels
        min_x = int(max(0, px - 3 * std_x))
        max_x = int(min(image_w, px + 3 * std_x))
        min_y = int(max(0, py - 3 * std_y))
        max_y = int(min(image_h, py + 3 * std_y))
        
        if min_x >= max_x or min_y >= max_y:
            continue
            
        # Create grid for bounding box
        y_coords, x_coords = torch.meshgrid(torch.arange(min_y, max_y), 
                                            torch.arange(min_x, max_x), 
                                            indexing='ij')
        y_coords = y_coords.to(gaussians['mean'].device)
        x_coords = x_coords.to(gaussians['mean'].device)
        
        # Distance vector d_k from pixel to center in screen space
        # Note: Sigma_2d is in pixel units if J was constructed with pixel units?
        # The formula for J uses fx, fy. If fx is in pixels, Sigma_2d is in pixels^2.
        # d_k should be in the same units as Sigma_2d.
        dx = x_coords - px
        dy = y_coords - py
        
        # Stack d_k
        d_k = torch.stack([dx, dy], dim=-1) # (H', W', 2)
        
        # Compute exponent: -0.5 * d^T Sigma^{-1} d
        # Sigma_inv = inverse of Sigma_2d
        Sigma_inv = torch.linalg.inv(sigma_k) # (2, 2)
        
        # d^T Sigma_inv d
        # For each pixel: sum_i sum_j d_i * Sigma_inv_ij * d_j
        # Using einsum for efficiency
        exponent = -0.5 * torch.einsum('...i,ij,...j->...', d_k, Sigma_inv, d_k)
        
        # Alpha
        alpha = opac_k * torch.exp(exponent)
        
        # Update image and transmittance
        # T_k = product(1 - alpha_j) for j < k
        # Current transmittance is T_k
        # New color contribution: T_k * alpha * color
        # New transmittance: T_k * (1 - alpha)
        
        alpha_clamped = torch.clamp(alpha, 0, 1)
        
        # Update image
        contribution = transmittance[:, :, 0] * alpha_clamped * color_k
        image[min_y:max_y, min_x:max_x] += contribution
        
        # Update transmittance
        transmittance[min_y:max_y, min_x:max_x, 0] *= (1 - alpha_clamped)
        
    return image

def quaternion_to_rotation_matrix(q):
    """Converts a quaternion (w, x, y, z) to a 3x3 rotation matrix."""
    w, x, y, z = q.unbind(dim=-1)
    N = w**2 + x**2 + y**2 + z**2
    w, x, y, z = w/torch.sqrt(N), x/torch.sqrt(N), y/torch.sqrt(N), z/torch.sqrt(N)
    
    return torch.stack([
        1 - 2*(y**2 + z**2), 2*(x*y - w*z), 2*(x*z + w*y),
        2*(x*y + w*z), 1 - 2*(x**2 + z**2), 2*(y*z - w*x),
        2*(x*z - w*y), 2*(y*z + w*x), 1 - 2*(x**2 + y**2)
    ], dim=-1).reshape(3, 3)
```

**Parameters:**
*   **Camera Model:** Pinhole camera with `look_at` function using OpenCV/COLMAP convention [S1, unofficial].
*   **Splat Threshold:** 3 times the standard deviation of the Gaussian [S2].
*   **Stability Blur:** Small isotropic dilation added to screen-space covariance [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-differentiable Sort:** The depth sort uses `argsort`, which is non-differentiable across order changes. Gradients flow through the values given the ordering, which is accepted for the fit-from-scratch demo [S1, unofficial].
*   **Performance:** The implementation uses a Python loop over Gaussians rather than vectorizing over all $(H, W, N)$ at once for readability. End-to-end run takes ~30 seconds on CPU for a 64x64 image [S1, unofficial].
*   **Omissions:** The MVP intentionally skips tile-based CUDA rasterizer, spherical-harmonics view-dependent color, adaptive density control, real captures, and SSIM losses [S1, unofficial].

## Subtopics

### [Relightable 3DGS](7851-relightable-3dgs.md)
Relightable 3DGS extends standard 3D Gaussian Splatting to handle complex appearances such as reflections, transparency, and glossy materials, which standard rasterization approximates poorly. It embeds physically-based material properties into each Gaussian for deferred shading, decomposing outgoing radiance into a BRDF integration that respects material parameters. This approach enables relighting the entire scene under novel illumination conditions while preserving real-time performance during view synthesis.

### HiFi4G
HiFi4G is a 3D Gaussian Splatting variant designed for high-fidelity reconstruction of dynamic humans from monocular video. It combines standard 3D Gaussian Splatting with a dual-graph mechanism to enable non-rigid tracking, ensuring spatial-temporal consistency while maintaining a compact representation. It distinguishes itself from static 3DGS by explicitly modeling temporal deformation and from other dynamic methods by its specific dual-graph topology for tracking.

### PRoGS
PRoGS is a post-training technique for 3D Gaussian Splatting that approximates each Gaussian's contribution to the final scene to construct a prioritization order, enabling progressive rendering or streaming by dynamically selecting splats to display or transmit [S1, unofficial]. Unlike existing web-viewers that require full scene retrieval before rendering, PRoGS loads a representative subset early, significantly reducing time-to-first-paint while maintaining perceptual quality [S1, unofficial]. It is designed for on-device rendering applications constrained by memory and streaming applications requiring minimal bandwidth [S1, unofficial].

### [3D Gaussian Splatting Rendering Pipeline (RViz Plugin)](10297-3d-gaussian-splatting-rendering-pipeline-rviz-plugin.md)
This article describes the OpenGL-based rendering pipeline for visualizing 3D Gaussian Splatting data within the ROS RViz environment. It solves the problem of efficiently rendering millions of 2D-projected Gaussian splats as a single instanced draw call while handling depth sorting and view-dependent shading. It differs from pure software implementations by leveraging GPU-based sorting and texture buffer objects (TBOs) for data storage, and from other RViz plugins by supporting both exact back-to-front sorting and Order-Independent Transparency (WBOIT) approximations.

### [CPU-based Gaussian Splatting Thumbnailer with JIT Optimization](14069-cpu-based-gaussian-splatting-thumbnailer-with-jit-optimizati.md)
This method provides a lightweight, CPU-only pipeline for generating rapid visual thumbnails from 3D Gaussian Splatting datasets, prioritizing responsiveness and portability over high-fidelity rendering. It distinguishes itself from standard GPU-accelerated splatting by omitting rotation parameters, approximating Gaussians as spheres, and leveraging Python's Numba JIT compiler to achieve near-real-time performance on constrained hardware.

### OmniGS
OmniGS is a method for fast radiance field reconstruction and real-time omnidirectional novel view synthesis using calibrated monocular 360-degree images and sparse SfM point clouds [S1, unofficial]. It replaces the standard perspective camera model with an omnidirectional equirectangular projection model, allowing 3D Gaussians to be splatted directly onto the equirectangular screen space [S1, unofficial]. Unlike sibling methods such as 360-GS, which rely on tangent-plane approximations and two-stage projection, OmniGS uses direct screen-space splatting without requiring scene assumptions or deep networks [S1, unofficial].

### [TR-Gaussians](17588-tr-gaussians.md)
TR-Gaussians is a 3D Gaussian Splatting-based representation designed for the high-fidelity, real-time rendering of planar transmission and reflection effects, such as those found in indoor glass panes. Unlike vanilla 3D Gaussian Splatting, which overfits training images by mixing transmission and reflection into low-opacity artifacts, TR-Gaussians explicitly decomposes the scene into real geometry (transmission) and virtual geometry (reflection) using a learnable reflection plane. This separation prevents the entanglement of real scene geometry with virtual reflections, ensuring that reflections remain coherent and artifact-free in novel views where the real scene is occluded or unobserved.

### [DOF-GS](18284-dof-gs.md)
DOF-GS is a 3D Gaussian Splatting framework that incorporates a finite-aperture camera model to explicitly model and render depth-of-field (DOF) effects, enabling post-capture control of aperture and focal distance. Unlike previous NeRF-based techniques that simulate defocus via expensive per-ray blur kernels or recent 3DGS methods that use black-box neural networks for shape offsets, DOF-GS blurs individual Gaussians based on learnable Circle-of-Confusion (CoC) cues before compositing, allowing efficient, differentiable, and intuitive DOF adjustment.

### [Temporal Gaussian Splatting](18651-temporal-gaussian-splatting.md)
Temporal Gaussian Splatting (TemporalGS) is a training-free, plug-and-play acceleration method for 3D Gaussian Splatting (3DGS) that leverages temporal priors from previous frames to reduce computational redundancy in Gaussian preprocessing, sorting, and rasterization. It solves the challenge of high FPS and low latency rendering by culling unnecessary Gaussians and approximating static regions via warped buffers, achieving up to 1.48× acceleration over vanilla 3DGS while maintaining competitive quality. Unlike sibling methods that may require post-training compression or hardware-specific optimizations, TemporalGS operates as a software rasterization plugin that interleaves dynamic culling and selective rendering intervals.

### [Geometry and Representation Adaptation](21351-geometry-and-representation-adaptation.md)
This family of methods adapts the volumetric 3D Gaussian Splatting representation to better handle geometric constraints, sparse inputs, or surface-oriented rendering tasks. While standard 3DGS relies on dense multi-view photometric consistency to implicitly define geometry, these variants introduce explicit geometric supervision, surface parameterizations, or thickness regularization to resolve ambiguities inherent in sparse or deformable scenarios. They differ from standard 3DGS by decoupling or constraining the spatial distribution of Gaussians to align with known geometric priors (meshes, depth maps, or height fields) rather than relying solely on appearance optimization.

### 3D Gaussian Splatting with Vector Clustering and Entropy Encoding
A method that utilizes vector clustering to encode color and geometric attributes into two codebooks, using a sensitivity-aware KMeans method [S3]. For disk data storage, SASCGS utilizes the entropy encoding method DEFLATE, which combines the LZ77 algorithm and Huffman coding, to compress the data [S3]. On the MipNeRF360 dataset, SASCGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S3].

### Vol3DGS
Enforces that alpha blending exactly satisfies the front-to-back absorption-emission model by deriving discrete compositing as an unbiased Monte Carlo estimator of the continuous integral [S4, unofficial]. Introduces a continuous density field along the ray to resolve artifacts in scenes with semi-transparent objects [S4, unofficial]. Volumetric alpha computation: $\alpha_i = 1 - \exp(-\int G_i(x) dx)$, accumulating density along the ray in accordance with volume rendering equation [S4, unofficial].

### Modified Alpha-Blending
A modified alpha-blending function designed to perform image rendering over re-sketched Gaussian lines and Gaussian triangles [S5, unofficial]. Render the re-sketched primitives using the modified alpha-blending function in a similar manner as original Gaussian ellipses [S5, unofficial].

### Depth Generation via Linear Interpolation
A method to compute accurate depth $\hat{D}$ for a pixel by treating it as a linear interpolation of distances from 3D Gaussians to the image plane, avoiding the floating problem and disc aliasing associated with peak selection in forward mapping splatting [S6]. For each pixel, render the view using 3D Gaussians to obtain $T_i$, $\alpha_i$, $d_i$, and $n_i$ for all contributing Gaussians $i$ [S6]. Formula: $\hat{D} = \sum_{i=1}^{N} \hat{w}_i d_i$, where $\hat{w}_i = \frac{T_i \alpha_i}{\sum_{i=1}^{N} T_i \alpha_i}$, $d_i$ is the distance from the i-th Gaussian to the image plane, $T_i$ is accumulated transmittance, and $\alpha_i$ is opacity [S6].

## Operating characteristics
*   **Time Complexity:** The MVP implementation uses a Python loop over Gaussians, making it significantly slower than the tile-based CUDA rasterizer used in production 3DGS. End-to-end run takes ~30 seconds on CPU for a 64x64 image [S1, unofficial].
*   **Memory Complexity:** Stores $N$ Gaussians with 11 parameters each. No additional large buffers for tile-based sorting or hierarchical binning are used [S1, unofficial].
*   **Strengths:** High readability and ease of implementation; suitable for educational purposes and prototyping; pure PyTorch allows for easy integration with standard deep learning workflows without custom CUDA kernels [S1, unofficial].
*   **Limitations:** Lacks performance optimizations (tile-based rasterization, vectorization); does not support view-dependent color (spherical harmonics); lacks adaptive density control (splitting/merging Gaussians); not suitable for real-time rendering or large-scale scenes [S1, unofficial].
*   **When to use:** Teaching, understanding the mathematical foundations of 3DGS, or small-scale experiments where performance is not critical [S1, unofficial].
*   **When not to use:** Production rendering, real-time applications, or large-scale scene reconstruction [S1, unofficial].

## Background
3D Gaussian Splatting MVP is based on the core algorithm of 3D Gaussian Splatting (Kerbl et al., SIGGRAPH 2023) [S1, unofficial]. It was introduced as a self-contained Jupyter notebook implementation in pure PyTorch, intended as a teaching reference to "grind on the math" without the complexity of custom CUDA kernels [S1, unofficial]. The implementation demonstrates the forward rasterization pipeline, including covariance construction, projection, sorting, and alpha compositing, using a simple pinhole camera model [S1, unofficial].

## Sources
[S1] Add 3D Gaussian Splatting MVP notebook, https://github.com/fangyuan-ksgk/diffuse-experiment/pull/3, fetched 2026-10-26 (unofficial)
[S2] Low Latency Point Cloud Rendering with Learned Splatting, https://doi.org/10.1109/cvprw63382.2024.00584, fetched 2026-10-26
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S5] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S6] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
