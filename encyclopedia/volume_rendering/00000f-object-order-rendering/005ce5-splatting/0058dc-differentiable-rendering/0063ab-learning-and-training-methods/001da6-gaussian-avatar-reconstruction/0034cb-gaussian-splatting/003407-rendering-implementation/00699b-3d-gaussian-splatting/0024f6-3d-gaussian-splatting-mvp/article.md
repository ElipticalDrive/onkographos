# 3D Gaussian Splatting MVP

Also called: 3DGS MVP, pure PyTorch 3DGS

## Overview
3D Gaussian Splatting MVP is a self-contained, pure-PyTorch implementation of the core 3D Gaussian Splatting algorithm [S1, unofficial]. It solves the problem of providing a readable, educational reference for the differentiable rendering pipeline without relying on custom CUDA kernels or optimized C++ extensions [S1, unofficial]. It sets itself apart from production implementations by using explicit Python loops for rasterization and omitting advanced features like spherical harmonics, adaptive density control, and tile-based sorting, prioritizing mathematical clarity over rendering speed [S1, unofficial].

## How it works
The method implements the forward pass of differentiable volume rendering for 3D Gaussians. It assumes the scene is represented by a set of $N$ 3D Gaussians, each defined by a mean position, a covariance matrix (parameterized by scale and rotation), an opacity, and a color.

### Inputs and Outputs
*   **Inputs:**
    *   $N$ Gaussians, each with parameters: mean $\mu \in \mathbb{R}^3$, scale $s \in \mathbb{R}^3$ (log-space), rotation $q \in \mathbb{R}^4$ (unit quaternion), color $c \in \mathbb{R}^3$, opacity $\alpha \in [0, 1]$.
    *   Camera extrinsics $W \in \mathbb{R}^{4 \times 4}$ (world-to-camera transformation).
    *   Camera intrinsics $K \in \mathbb{R}^{3 \times 3}$ (projection matrix).
    *   Image dimensions $H, W$.
*   **Outputs:**
    *   Rendered image $I \in \mathbb{R}^{H \times W \times 3}$.
    *   (Optional) Depth map and transmittance for auxiliary losses.

### Data Structures
The Gaussian attributes are stored in PyTorch tensors of shape $(N, \text{field})$.
*   `means`: $(N, 3)$, world-space coordinates.
*   `scales`: $(N, 3)$, log-scale factors.
*   `rotations`: $(N, 4)$, unit quaternions $(w, x, y, z)$.
*   `opacities`: $(N, 1)$, scalar opacity values.
*   `colors`: $(N, 3)$, RGB color values.

### Covariance Construction
The 3D covariance matrix $\Sigma$ is constructed from the scale and rotation parameters to ensure it is symmetric positive semi-definite (PSD) [S1, unofficial].

1.  Convert the unit quaternion $q$ to a $3 \times 3$ rotation matrix $R$.
2.  Create a diagonal scale matrix $S = \text{diag}(e^{s_x}, e^{s_y}, e^{s_z})$.
3.  Compute the covariance:
    $$ \Sigma = R S S^T R^T $$

### 2D Screen-Space Covariance Projection
The 3D covariance is projected onto the 2D image plane using the EWA (Elliptical Weighted Average) / Zwicker-style projection [S1, unofficial].

1.  Transform the mean $\mu$ to camera space: $\mu_c = W \mu$.
2.  Transform the covariance to camera space: $\Sigma_c = W \Sigma W^T$.
3.  Compute the projection Jacobian $J$. For a pinhole camera, $J$ relates changes in 3D camera-space coordinates to changes in 2D screen coordinates. It is derived from the perspective projection equations:
    $$ J = \begin{bmatrix} \frac{\partial u}{\partial x} & \frac{\partial u}{\partial y} & \frac{\partial u}{\partial z} \\ \frac{\partial v}{\partial x} & \frac{\partial v}{\partial y} & \frac{\partial v}{\partial z} \end{bmatrix} $$
    Specifically, if $K = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$, then for a point $(X, Y, Z)$ in camera space:
    $$ J = \begin{bmatrix} \frac{f_x}{Z} & 0 & -\frac{f_x X}{Z^2} \\ 0 & \frac{f_y}{Z} & -\frac{f_y Y}{Z^2} \end{bmatrix} $$
4.  Compute the 2D covariance:
    $$ \Sigma_{2d} = J \Sigma_c J^T $$
5.  Add a small isotropic dilation $\sigma_{blur}^2 I$ to $\Sigma_{2d}$ for numerical stability and to prevent singularities when $Z \to 0$ or for very small Gaussians [S1, unofficial].

### Rasterization Steps
The rasterizer processes Gaussians in a specific order to ensure correct alpha compositing.

1.  **Transform Means:** Transform all Gaussian means from world space to camera space using $W$ [S1, unofficial].
2.  **Transform Covariances:** Transform the 3D covariance matrices into the camera frame as described above [S1, unofficial].
3.  **Project to 2D:** Compute $\Sigma_{2d}$ for each Gaussian [S1, unofficial].
4.  **Depth Sort:** Sort the Gaussians by their depth (z-coordinate in camera space) in front-to-back order. The implementation uses `argsort` on the z-coordinates [S1, unofficial].
5.  **Splat and Composite:** Iterate over the sorted Gaussians and update the image buffers using front-to-back alpha compositing [S1, unofficial].

#### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def quaternion_to_rotation_matrix(q):
    """Convert unit quaternion (w, x, y, z) to 3x3 rotation matrix."""
    w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
    # Standard conversion formula
    R = torch.stack([
        1 - 2*(y**2 + z**2), 2*(x*y - w*z), 2*(x*z + w*y),
        2*(x*y + w*z), 1 - 2*(x**2 + z**2), 2*(y*z - w*x),
        2*(x*z - w*y), 2*(y*z + w*x), 1 - 2*(x**2 + y**2)
    ], dim=1).reshape(-1, 3, 3)
    return R

def compute_jacobian(mean_cam, K):
    """Compute the 2x3 projection Jacobian for a single point in camera space."""
    x, y, z = mean_cam[:, 0], mean_cam[:, 1], mean_cam[:, 2]
    fx, fy = K[0, 0], K[1, 1]
    cx, cy = K[0, 2], K[1, 2]
    
    # Jacobian elements
    # du/dx = fx/z, du/dy = 0, du/dz = -fx*x/z^2
    # dv/dx = 0, dv/dy = fy/z, dv/dz = -fy*y/z^2
    
    J = torch.zeros(mean_cam.shape[0], 2, 3, device=mean_cam.device)
    J[:, 0, 0] = fx / z
    J[:, 0, 2] = -fx * x / (z**2)
    J[:, 1, 1] = fy / z
    J[:, 1, 2] = -fy * y / (z**2)
    return J

def render_gaussians_mvp(means, scales, rotations, opacities, colors, W, K, H, W_img):
    """
    Pure PyTorch MVP rasterizer.
    """
    N = means.shape[0]
    
    # 1. Transform means to camera space
    # Homogeneous coordinates
    means_h = torch.cat([means, torch.ones(N, 1, device=means.device)], dim=1)
    means_cam = (W @ means_h.T).T[:, :3]
    
    # 2. Construct 3D covariance and transform to camera space
    R = quaternion_to_rotation_matrix(rotations)
    S = torch.diag_embed(torch.exp(scales)) # S is 3x3 diagonal
    Sigma_3d = torch.bmm(torch.bmm(R, S), torch.bmm(S, R.transpose(1, 2))) # R S S^T R^T
    
    # Transform covariance: Sigma_c = W Sigma W^T
    # Note: W is 4x4. We only need the upper 3x3 part for covariance transformation
    W_3x3 = W[:3, :3]
    Sigma_cam = torch.bmm(torch.bmm(W_3x3, Sigma_3d), W_3x3.transpose(1, 2))
    
    # 3. Project to 2D
    J = compute_jacobian(means_cam, K)
    Sigma_2d = torch.bmm(torch.bmm(J, Sigma_cam), J.transpose(1, 2))
    
    # Add isotropic blur for stability
    blur = 0.3 # Typical small value
    Sigma_2d[:, 0, 0] += blur
    Sigma_2d[:, 1, 1] += blur
    
    # 4. Depth Sort
    depths = means_cam[:, 2]
    sort_indices = torch.argsort(depths, dim=0)
    
    # Gather sorted attributes
    means_cam_sorted = means_cam[sort_indices]
    Sigma_2d_sorted = Sigma_2d[sort_indices]
    opacities_sorted = opacities[sort_indices]
    colors_sorted = colors[sort_indices]
    
    # 5. Splat and Composite
    # Initialize buffers
    out_color = torch.zeros(H, W_img, 3, device=means.device)
    out_alpha = torch.zeros(H, W_img, 1, device=means.device)
    
    # Precompute bounding boxes for each Gaussian to limit pixel updates
    # This is a simplification; a full MVP might iterate all pixels or use a sparse grid
    # For readability, we iterate Gaussians and update pixels within their bounding box
    
    for i in range(N):
        mu = means_cam_sorted[i]
        cov2d = Sigma_2d_sorted[i]
        alpha = opacities_sorted[i]
        color = colors_sorted[i]
        
        # Inverse 2D covariance
        cov2d_inv = torch.inverse(cov2d)
        
        # Determine bounding box in screen space
        # Eigenvalues of cov2d give the spread
        eigvals, eigvecs = torch.linalg.eigh(cov2d)
        # Radius is typically 3 standard deviations
        radius_x = 3.0 * torch.sqrt(eigvals[0])
        radius_y = 3.0 * torch.sqrt(eigvals[1])
        
        # Project mean to screen
        u = K[0, 0] * mu[0] / mu[2] + K[0, 2]
        v = K[1, 1] * mu[1] / mu[2] + K[1, 2]
        
        u_min = max(0, int(u - radius_x))
        u_max = min(W_img, int(u + radius_x))
        v_min = max(0, int(v - radius_y))
        v_max = min(H, int(v + radius_y))
        
        # Create grid of pixel coordinates
        ys, xs = torch.meshgrid(torch.arange(v_min, v_max), torch.arange(u_min, u_max), indexing='ij')
        pixel_coords = torch.stack([xs.float(), ys.float()], dim=-1) # (H', W', 2)
        
        # Compute 2D Gaussian weights
        # delta = pixel - mean
        delta = pixel_coords - torch.tensor([u, v], device=means.device)
        # exponent = -0.5 * delta^T Sigma_inv delta
        exponent = -0.5 * torch.sum(delta @ cov2d_inv * delta, dim=-1)
        weight = torch.exp(exponent)
        
        # Alpha contribution
        # a = alpha * weight
        a = alpha * weight
        
        # Transmittance from previous layers (accumulated)
        # T_i = product(1 - a_j) for j < i
        # We need to track T for each pixel. 
        # Since we are in a loop, we can maintain a global T buffer.
        
        # Update buffers
        # T_current = out_alpha[v_min:v_max, u_min:u_max]
        # new_alpha = a * (1 - T_current)
        # out_color += color * new_alpha
        # out_alpha += new_alpha
        
        # Vectorized update for the bounding box
        T_current = out_alpha[v_min:v_max, u_min:u_max] # (H', W', 1)
        new_alpha = a.unsqueeze(-1) * (1 - T_current)
        
        out_color[v_min:v_max, u_min:u_max] += color * new_alpha
        out_alpha[v_min:v_max, u_min:u_max] += new_alpha

    return out_color
```

### Parameters
*   **Camera Model:** Pinhole camera with `look_at` function using OpenCV/COLMAP convention [S1, unofficial].
*   **Optimization:** Adam optimizer is used for fitting Gaussians to images [S1, unofficial].
*   **Loss:** Photometric L1 loss and MSE [S1, unofficial].
*   **Iterations:** 600 iterations for the demo [S1, unofficial].

### Edge Cases and Pitfalls
*   **Depth Sort Non-Differentiability:** The depth sort uses `argsort`, which is non-differentiable across order changes. Gradients flow through the values given the ordering, which is accepted for the fit-from-scratch demo [S1, unofficial].
*   **Precision:** A small isotropic dilation is added to the 2D covariance to prevent numerical instability when the Gaussian is far from the camera or very small [S1, unofficial].
*   **Boundaries:** Pixels outside the image bounds or behind the camera ($Z \le 0$) should be discarded or handled to avoid division by zero in projection [G].

## Subtopics

### [3D Gaussian Splatting with Gaussian Unpooling](6133-3d-gaussian-splatting-with-gaussian-unpooling.md)
FSGS is the first method to explore reconstructing 3D scenes from sparse view input. It addresses insufficient initialization data by using Gaussian unpooling to expand sparse Structure-from-Motion (SfM) points into a denser set of Gaussians, enabling faithful geometry reconstruction where standard methods fail due to lack of initial coverage.

### [Relightable 3DGS](7851-relightable-3dgs.md)
Relightable 3DGS extends standard 3D Gaussian Splatting by embedding physically-based material properties into each Gaussian, enabling deferred shading and realistic rendering of complex appearances such as reflections, transparency, and glossy surfaces. Unlike standard rasterization which approximates view-dependent colors via spherical harmonics and fails to represent specular reflections or global illumination, this method decomposes outgoing radiance into a BRDF integration that respects material parameters. It sets itself apart from its siblings by decoupling geometry processing from lighting and material computations, adapting a two-pass deferred pipeline to Gaussian Splatting for efficient rendering of high-frequency, view-dependent effects.

### HiFi4G
HiFi4G is a 3D Gaussian Splatting variant designed for high-fidelity reconstruction of dynamic humans from monocular video. It combines standard 3D Gaussian Splatting with a dual-graph mechanism to enable non-rigid tracking, ensuring spatial-temporal consistency while maintaining a compact representation. It distinguishes itself from static 3DGS by explicitly modeling temporal deformation and from other dynamic methods by its specific dual-graph topology for tracking.

### [3D Gaussian Splatting with Mesh UV Parameterization](8046-3d-gaussian-splatting-with-mesh-uv-parameterization.md)
This method enables real-time and high-quality rendering of animated humans by projecting 3D Gaussian parameters into a 2D texture space via mesh UV parameterization. It distinguishes itself from standard 3D Gaussian Splatting by decoupling appearance storage from 3D spatial coordinates, allowing for compact representation and efficient texture-based rendering of dynamic cloth and accessories.

### PRoGS
PRoGS is a post-training technique for 3D Gaussian Splatting that approximates each Gaussian's contribution to the final scene to construct a prioritization order, enabling progressive rendering or streaming by dynamically selecting splats to display or transmit [S1, unofficial]. Unlike existing web-viewers that require full scene retrieval before rendering, PRoGS loads a representative subset early, significantly reducing time-to-first-paint while maintaining perceptual quality [S1, unofficial]. It is designed for on-device rendering applications constrained by memory and streaming applications requiring minimal bandwidth [S1, unofficial].

### [3D Gaussian Splatting Rendering Pipeline (RViz Plugin)](10297-3d-gaussian-splatting-rendering-pipeline-rviz-plugin.md)
This article describes the OpenGL-based rendering pipeline for visualizing 3D Gaussian Splatting data within the ROS RViz environment. It solves the problem of efficiently rendering millions of 2D-projected Gaussian splats as a single instanced draw call, handling depth sorting and view-dependent shading. It differs from pure software implementations by leveraging GPU-based sorting and texture buffer objects (TBOs) for data storage, and from other RViz plugins by supporting both exact back-to-front sorting and Order-Independent Transparency (WBOIT) approximations.

### [CPU-based Gaussian Splatting Thumbnailer with JIT Optimization](14069-cpu-based-gaussian-splatting-thumbnailer-with-jit-optimizati.md)
This method provides a lightweight, CPU-only pipeline for generating rapid visual thumbnails from 3D Gaussian Splatting datasets, prioritizing responsiveness and portability over high-fidelity rendering. It distinguishes itself from standard GPU-accelerated splatting by omitting rotation parameters, approximating Gaussians as spheres, and leveraging Python's Numba JIT compiler to achieve near-real-time performance on constrained hardware.

### OmniGS
OmniGS is a method for fast radiance field reconstruction and real-time omnidirectional novel view synthesis using calibrated monocular 360-degree images and sparse SfM point clouds [S1, unofficial]. It replaces the standard perspective camera model with an omnidirectional equirectangular projection model, allowing 3D Gaussians to be splatted directly onto the equirectangular screen space [S1, unofficial]. Unlike sibling methods such as 360-GS, which rely on tangent-plane approximations and two-stage projection, OmniGS uses direct screen-space splatting without requiring scene assumptions or deep networks [S1, unofficial].

### [TR-Gaussians](17588-tr-gaussians.md)
TR-Gaussians is a 3D Gaussian Splatting-based representation designed for the high-fidelity, real-time rendering of planar transmission and reflection effects, such as those found in indoor glass panes. Unlike vanilla 3D Gaussian Splatting, which overfits training images by mixing transmission and reflection into low-opacity artifacts, TR-Gaussians explicitly decomposes the scene into real geometry (transmission) and virtual geometry (reflection) using a learnable reflection plane. This separation prevents the entanglement of real scene geometry with virtual reflections, ensuring that reflections remain coherent and artifact-free in novel views where the real scene is occluded or unobserved.

### [DOF-GS](18284-dof-gs.md)
DOF-GS is a 3D Gaussian Splatting framework that incorporates a finite-aperture camera model to explicitly model and render depth-of-field (DOF) effects, enabling post-capture control of aperture and focal distance. Unlike previous NeRF-based techniques that simulate defocus via expensive per-ray blur kernels or recent 3DGS methods that use black-box neural networks for shape offsets, DOF-GS blurs individual Gaussians based on learnable Circle-of-Confusion (CoC) cues before compositing, allowing efficient, differentiable, and intuitive DOF adjustment.

### 3D Gaussian Splatting with Vector Clustering and Entropy Encoding
A method that utilizes vector clustering to encode color and geometric attributes into two codebooks, using a sensitivity-aware KMeans method [S2]. For disk data storage, SASCGS utilizes the entropy encoding method DEFLATE, which combines the LZ77 algorithm and Huffman coding, to compress the data [S2]. On the MipNeRF360 dataset, SASCGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S2].

### Vol3DGS
Enforces that alpha blending exactly satisfies the front-to-back absorption-emission model by deriving discrete compositing as an unbiased Monte Carlo estimator of the continuous integral [S3, unofficial]. It introduces a continuous density field along the ray to resolve artifacts in scenes with semi-transparent objects [S3, unofficial]. Volumetric alpha computation is defined as $\alpha_i = 1 - \exp(-\int G_i(x) dx)$, accumulating density along the ray in accordance with the volume rendering equation [S3, unofficial].

### Modified Alpha-Blending
A modified alpha-blending function designed to perform image rendering over re-sketched Gaussian lines and Gaussian triangles [S4, unofficial]. It renders the re-sketched primitives using the modified alpha-blending function in a similar manner as original Gaussian ellipses [S4, unofficial].

### Depth Generation via Linear Interpolation
A method to compute accurate depth $\hat{D}$ for a pixel by treating it as a linear interpolation of distances from 3D Gaussians to the image plane, avoiding the floating problem and disc aliasing associated with peak selection in forward mapping splatting [S5]. For each pixel, it renders the view using 3D Gaussians to obtain $T_i$, $\alpha_i$, $d_i$, and $n_i$ for all contributing Gaussians $i$ [S5]. The depth is calculated as $\hat{D} = \sum_{i=1}^{N} \hat{w}_i d_i$, where $\hat{w}_i = \frac{T_i \alpha_i}{\sum_{i=1}^{N} T_i \alpha_i}$, $d_i$ is the distance from the i-th Gaussian to the image plane, $T_i$ is accumulated transmittance, and $\alpha_i$ is opacity [S5].

## Operating characteristics
*   **Time Complexity:** The MVP implementation uses a Python loop over Gaussians, making it $O(N \cdot H \cdot W)$ in the worst case if bounding boxes are not used, or $O(N \cdot A)$ where $A$ is the average area of a Gaussian's bounding box. It is significantly slower than the tile-based CUDA rasterizer used in production 3DGS.
*   **Memory Complexity:** $O(N)$ to store Gaussian attributes and $O(H \cdot W)$ for the output buffers.
*   **Measured Performance:** End-to-end run takes ~30 seconds on CPU for a 64x64 image [S1, unofficial].
*   **Strengths:** High readability, no external dependencies beyond PyTorch/Numpy, suitable for educational purposes and debugging the mathematical pipeline.
*   **Limitations:** Slow performance, lacks optimizations like tile-based sorting, spherical harmonics for view-dependent color, and adaptive density control [S1, unofficial].
*   **When to use:** For learning the core algorithm, debugging, or when GPU acceleration is unavailable.
*   **When not to use:** For production rendering, large-scale scenes, or real-time applications.

## Background
The 3D Gaussian Splatting MVP notebook was introduced as a contribution to the `diffuse-experiment` repository [S1, unofficial]. It implements the core algorithm from the original 3D Gaussian Splatting paper (Kerbl et al., SIGGRAPH 2023) but in pure PyTorch [S1, unofficial]. The implementation is intended as a teaching reference, allowing users to "grind on the math" without the complexity of custom CUDA kernels [S1, unofficial]. It was developed by Devin AI integration and committed to the repository in May 2026 [S1, unofficial].

## Sources
[S1] Add 3D Gaussian Splatting MVP notebook, https://github.com/fangyuan-ksgk/diffuse-experiment/pull/3, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S5] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
