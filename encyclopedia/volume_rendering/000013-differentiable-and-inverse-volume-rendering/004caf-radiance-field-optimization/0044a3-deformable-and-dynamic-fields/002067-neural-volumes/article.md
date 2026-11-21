# Neural Volumes

Also called: Deformable Volumes, Neural Volumes: Learning Dynamic Renderable Volumes from Images, Volumetric Reconstruction with Ray Potentials

## Overview
Neural Volumes (NV) is a method for synthesizing novel views of dynamic scenes by optimizing a deep 3D convolutional network to predict a discretized RGB$\alpha$ voxel grid and a 3D warp field [S1, S2]. It solves the problem of rendering semi-transparent, non-rigidly deforming objects (such as smoke or hair) from sparse multi-view video inputs without requiring explicit geometry or texture maps [S2]. It differs from previous ray potential works by modeling voxel transparency rather than occupancy probability, focusing on rendering rather than reconstruction [S2]. It uses an explicit voxel grid decoded from a latent code, avoiding the prohibitive computational cost of evaluating large MLPs at every ray-marching step [S1][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $K$ synchronized, calibrated RGB images from different viewpoints for each time instant (typically $K=3$ orthogonal views) [S2].
*   **Outputs:** A rendered RGB image for a novel camera pose, and a latent code $z$ representing the scene state [S2].

**Data Structures:**
1.  **Voxel Grid ($V$):** A discrete 3D grid of size $D \times D \times D$ (typically $128^3$) storing RGBA values [S1, S2].
    *   Layout: Tensor $Y \in \mathbb{R}^{D \times D \times D \times 4}$.
    *   Values: $\mathbf{V}_{\text{rgb}} \in \mathbb{R}^3$ (color) and $\mathbf{V}_{\alpha} \in \mathbb{R}$ (differential opacity, range $[0, \infty]$) [S2].
2.  **Warp Grid ($W$):** A discrete 3D grid of size $N_w \times N_w \times N_w$ (typically $32^3$) storing inverse warp vectors [S1][S4, unofficial].
    *   Layout: Tensor storing the inverse transformation $W^{-1}(\mathbf{x})$ for each voxel center [S4, unofficial].
3.  **Latent Code ($z$):** A vector $z \in \mathbb{R}^{256}$ representing the scene state at a specific time instant [S2].

**Network Architecture:**
1.  **Encoder:** A CNN processes each of the $K$ input images through dedicated branches. Outputs are concatenated and encoded into the parameters of a diagonal 256-dimensional Gaussian ($\mu, \sigma$) [S2].
2.  **Latent Sampling:** An instance $z$ is sampled from the Gaussian using the reparameterization trick [S2].
3.  **Decoder:** A 3D convolutional network (or linear basis decoder) maps $z$ to the Voxel Grid $V$ and the Warp Grid $W$ [S1][S4, unofficial].
    *   Activation: Softplus is used to ensure non-negative opacity values [S4, unofficial].

**Rendering (Differentiable Ray Marching):**
For a camera ray $\mathbf{r}$, the color $\hat{C}(\mathbf{r})$ and accumulated opacity $\hat{A}(\mathbf{r})$ are computed by marching through the warped voxel grid [S2][S4, unofficial].

1.  **Warping:** For a sample point $\mathbf{x}$ along the ray, the inverse warp field $W^{-1}(\mathbf{x})$ is sampled. This maps the world-space position to a position in the template (canonical) space [S2][S4, unofficial].
    *   *Note:* The warp field can be modeled as a mixture of affine transformations in warp space [S4, unofficial].
2.  **Sampling:** The RGBA values are sampled from the Voxel Grid at the warped position: $(\mathbf{c}, \alpha) = V(W^{-1}(\mathbf{x}))$ [S4, unofficial].
3.  **Accumulation:** Values are accumulated front-to-back using alpha blending [S4, unofficial]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{j} \mathbf{c}_j \cdot \alpha_j \cdot \prod_{k<j} (1 - \alpha_k)
    $$
    $$
    \hat{A}(\mathbf{r}) = 1 - \prod_{j} (1 - \alpha_j)
    $$
    where $\mathbf{c}_j$ and $\alpha_j$ are the color and differential opacity at the $j$-th sample [S4, unofficial].

**Training:**
The system is trained end-to-end by minimizing the reconstruction loss between the rendered image and the input image [S2].

**Loss Function:**
$$
\mathcal{L}(\theta) = \| I_{\text{rgb}} - I^*_{\text{rgb}} \|_2^2 + \beta \cdot D_{KL}(z \| \mathcal{N}(0,1)) + \lambda_{tv} J_{tv}(\mathbf{V}_{\alpha}) + \lambda_B J_B(I_{\alpha})
$$
*   $\| I_{\text{rgb}} - I^*_{\text{rgb}} \|_2^2$: L2 pixel reconstruction error [S2][S4, unofficial].
*   $D_{KL}(z \| \mathcal{N}(0,1))$: KL-divergence regularization for the latent code [S2][S4, unofficial].
*   $J_{tv}(\mathbf{V}_{\alpha})$: Total variation regularization on opacity to encourage sharp boundaries [S4, unofficial].
*   $J_B(I_{\alpha})$: Beta distribution prior on accumulated opacity to encourage binary transparency (fully transparent or opaque) [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def encoder(images):
    """
    images: Tensor of shape (B, K, H, W, 3)
    Returns: mu (B, 256), sigma (B, 256)
    """
    B, K, H, W, C = images.shape
    # CNN branches process each view, concatenate, and encode
    x = images.view(B, K * H * W, C)
    x = torch.relu(x)
    mu = torch.tanh(x.mean(dim=1))
    sigma = torch.exp(x.mean(dim=1))
    return mu, sigma

def decoder(z, voxel_res, warp_res):
    """
    z: Tensor of shape (B, 256)
    Returns: voxel_grid (B, D, D, D, 4), warp_grid (B, W, W, W, 3)
    """
    B = z.shape[0]
    # Linear basis decoder or 3D transposed convolutions
    voxel_flat = F.softplus(nn.Linear(256, voxel_res**3 * 4)(z))
    warp_flat = nn.Linear(256, warp_res**3 * 3)(z)
    voxel_grid = voxel_flat.view(B, voxel_res, voxel_res, voxel_res, 4)
    warp_grid = warp_flat.view(B, warp_res, warp_res, warp_res, 3)
    return voxel_grid, warp_grid

def trilinear_sample(grid, points):
    """
    grid: (B, C, D, H, W)
    points: (B, N, 3) in normalized [-1, 1] coordinates
    Returns: (B, N, C)
    """
    # Normalize points to [-1, 1] assuming scene bounds are [-1, 1]^3
    points = torch.clamp(points, -1.0, 1.0)
    # Expand points to match grid_sample expected shape (B, N, 1, 1, 3)
    points = points.unsqueeze(2).unsqueeze(3)
    # grid_sample expects grid (N, C, D, H, W) and coordinates (N, H_out, W_out, D_out, 3)
    samples = F.grid_sample(grid, points, mode='bilinear', padding_mode='border')
    return samples.squeeze(2).squeeze(2)

def ray_march(voxel_grid, warp_grid, origins, directions, num_samples):
    """
    voxel_grid: (B, D, D, D, 4)
    warp_grid: (B, W, W, W, 3)
    origins: (B, H, W, 3)
    directions: (B, H, W, 3)
    Returns: rendered_rgb (B, H, W, 3), rendered_alpha (B, H, W, 1)
    """
    B, H, W, _ = origins.shape
    # Generate sample points along rays
    t_vals = torch.linspace(0.0, 1.0, num_samples, device=origins.device)
    t_vals = t_vals.view(1, 1, 1, -1).expand_as(origins[..., :1])
    points_world = origins[..., :1] + t_vals * directions
    points_world = points_world.view(B * H * W * num_samples, 3)

    # Sample warp field: W_inv(x)
    warp_vectors = trilinear_sample(warp_grid.permute(0, 4, 1, 2, 3), points_world)
    warp_vectors = warp_vectors.view(B, H, W, num_samples, 3)

    # Sample voxel grid at warped positions
    voxel_values = trilinear_sample(voxel_grid.permute(0, 4, 1, 2, 3), warp_vectors.view(B * H * W * num_samples, 3))
    voxel_values = voxel_values.view(B, H, W, num_samples, 4)

    rgb_samples = voxel_values[..., :3]
    alpha_samples = voxel_values[..., 3:]

    # Accumulate
    one_minus_alpha = 1.0 - alpha_samples
    transmittance = torch.cat([torch.ones_like(one_minus_alpha[..., :1]),
                               torch.cumprod(one_minus_alpha[..., :-1], dim=-1)], dim=-1)
    weights = alpha_samples * transmittance
    rendered_rgb = torch.sum(weights * rgb_samples, dim=-2)
    rendered_alpha = 1.0 - torch.prod(one_minus_alpha, dim=-2)
    return rendered_rgb, rendered_alpha

def total_variation_loss(x):
    """Standard TV regularization [G]"""
    return torch.mean(torch.abs(x[..., :-1, :, :, :] - x[..., 1:, :, :, :])) + \
           torch.mean(torch.abs(x[..., :, :-1, :, :] - x[..., :, 1:, :, :])) + \
           torch.mean(torch.abs(x[..., :, :, :-1, :] - x[..., :, :, 1:, :]))

def beta_prior_loss(alpha, beta=1.0):
    """Beta distribution prior encouraging binary transparency [G]"""
    return -torch.mean(torch.log(alpha + 1e-6) + torch.log(1.0 - alpha + 1e-6))

def forward(images, target_camera_params, gt_images, voxel_res=128, warp_res=32, num_samples=128):
    mu, sigma = encoder(images)
    z = mu + sigma * torch.randn_like(sigma)
    voxel_grid, warp_grid = decoder(z, voxel_res, warp_res)
    rendered_rgb, rendered_alpha = ray_march(voxel_grid, warp_grid,
                                             target_camera_params['origins'],
                                             target_camera_params['directions'],
                                             num_samples)
    rgb_loss = F.mse_loss(rendered_rgb, gt_images)
    kl_loss = -0.5 * torch.sum(1 + torch.log(sigma**2) - mu**2 - sigma**2)
    tv_loss = total_variation_loss(voxel_grid[..., 3:])
    beta_loss = beta_prior_loss(rendered_alpha)
    total_loss = rgb_loss + 0.1 * kl_loss + tv_loss + beta_loss
    return total_loss, rendered_rgb
```

## Subtopics

### [Voxel Grid Decoders](2219-voxel-grid-decoders.md)
Voxel Grid Decoders model the volume function $V(\mathbf{x}; \mathbf{z})$ as a discrete 3D grid of voxels, producing an explicit 3D voxel grid as the output tensor of a neural network. This approach avoids the prohibitive computational requirements of evaluating deep MLPs at every ray-marching step, enabling real-time rendering speeds. It sets itself apart from implicit MLP decoders by trading memory for speed, storing values in a structured grid rather than computing them on the fly.

### [RenderNet Architecture](6691-rendernet-architecture.md)
RenderNet is a deep convolutional network designed for the differentiable rendering of voxelized 3D shapes. It solves the problem of synthesizing novel views by mapping a 3D input volume directly to a 2D image. What sets it apart from its siblings, such as [Voxel Grid Decoders](2219-voxel-grid-decoders.md) and [GRU Volume Fusion](7543-gru-volume-fusion.md), is its explicit architectural separation of 3D feature extraction and 2D image decoding, connected by a projection unit that aggregates features along the viewing ray using an MLP. Unlike methods that extract features in object space independent of camera parameters, RenderNet performs view-space resampling at the beginning of the pipeline, making the features inherently dependent on the camera pose.

### [GRU Volume Fusion](7543-gru-volume-fusion.md)
GRU Volume Fusion is a method for incremental 3D reconstruction that fuses sequentially estimated local radiance fields into a single global sparse representation using a Gated Recurrent Unit (GRU). It solves the problem of accumulating errors and outliers in sequential reconstruction by adaptively updating a global volume with new local observations, setting it apart from its sibling [Bi-level Fusion](14766-bi-level-fusion.md) which uses explicit weighted averaging followed by optimization rather than recurrent neural updates. It builds on [Neural Volumes](912-neural-volumes.md) by replacing static optimization with a learned recurrent fusion process.

## Operating characteristics
*   **Training Time:** Requires at least 12 hours to train per scene [S1].
*   **Memory:** Uses an explicit $128^3$ voxel grid, which limits the ability to represent fine details at high resolutions compared to implicit methods [S1, S3].
*   **Inference Speed:** Supports real-time inference at test time, unlike many other neural rendering methods [S2].
*   **Strengths:** Captures reasonably detailed volumetric geometry and appearance; handles semi-transparent materials (smoke, hair); interpretable volume; real-time rendering [S1, S2].
*   **Limitations:** Blurry results compared to high-resolution methods due to voxel grid resolution limits; limited to objects within a bounded volume with a distinct background [S1, S3].
*   **When to use:** Dynamic scenes with semi-transparent materials where real-time novel view synthesis is required and explicit geometry is not available [S2].

## Background
Neural Volumes was introduced by Lempitsky et al. in 2019 [S2]. It builds on the concept of ray potentials and volume ray marching [S2]. It is a baseline method that uses a 3D deconvolutional network to produce RGB-Alpha volumes as the volumetric video [S3]. It differs from previous ray potential works by modeling voxel transparency rather than occupancy probability, focusing on rendering rather than reconstruction [S2]. It also differs from DeepVoxels by supporting dynamic scenes and real-time inference [S2].

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S3] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S4] Neural Volumes: 从图像中学习动态可渲染体, https://www.alphaxiv.org/zh/abs/1906.07751, fetched 2026-10-30 (unofficial)
