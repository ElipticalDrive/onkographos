# TensoRF

Also called: TensoRF (Tensorial Radiance Field Representation), TensoRF Representation, TensoRF Representation and Reconstruction, Tensorial Radiance Field Representation, Tensorial Radiance Fields

## Overview
TensoRF models a radiance field as a factorized 4D tensor, replacing the multi-layer perceptrons (MLPs) used in [NeRF](4972-neural-radiance-fields.md) with compact vector and matrix factors. It achieves significantly faster reconstruction times and smaller model sizes by exploiting the low-rank structure of the scene's geometry and appearance. Unlike [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md) which stores dense voxel grids, TensoRF uses tensor decomposition (CP or VM) to correlate and regularize the entire field, reducing space complexity from $O(n^3)$ to $O(n)$ or $O(n^2)$ [S1, S3].

## How it works
**Inputs:** Multi-view images with known camera poses.
**Outputs:** Optimized tensor factors representing the scene's volume density $\sigma$ and view-dependent color $c$.

### Data Structures
The method maintains two factorized grids:
1.  **Geometry Grid ($G_\sigma$):** A 3D tensor of size $I \times J \times K$ representing volume density. It is factorized into vectors $\mathbf{v}$ and matrices $\mathbf{M}$.
2.  **Appearance Grid ($G_c$):** A 4D tensor of size $I \times J \times K \times P$ representing appearance features. It is factorized into vectors $\mathbf{v}$, matrices $\mathbf{M}$, and a global appearance dictionary matrix $\mathbf{B}$.

**Factor Layout:**
*   **CP Decomposition:**
    *   $G_\sigma$: $R_\sigma$ components. Each component $r$ consists of three vectors $\mathbf{v}_{\sigma,r}^X \in \mathbb{R}^I$, $\mathbf{v}_{\sigma,r}^Y \in \mathbb{R}^J$, $\mathbf{v}_{\sigma,r}^Z \in \mathbb{R}^K$.
    *   $G_c$: $R_c$ components. Each component $r$ consists of three vectors $\mathbf{v}_{c,r}^X, \mathbf{v}_{c,r}^Y, \mathbf{v}_{c,r}^Z$ and one feature vector $\mathbf{b}_r \in \mathbb{R}^P$.
*   **VM Decomposition (Vector-Matrix):**
    *   $G_\sigma$: $R_\sigma$ components. Each component $r$ consists of three vectors $\mathbf{v}_{\sigma,r}^X \in \mathbb{R}^I$, $\mathbf{v}_{\sigma,r}^Y \in \mathbb{R}^J$, $\mathbf{v}_{\sigma,r}^Z \in \mathbb{R}^K$ and three matrices $\mathbf{M}_{\sigma,r}^{YZ} \in \mathbb{R}^{J \times K}$, $\mathbf{M}_{\sigma,r}^{XZ} \in \mathbb{R}^{I \times K}$, $\mathbf{M}_{\sigma,r}^{XY} \in \mathbb{R}^{I \times J}$.
    *   $G_c$: $R_c$ components. Each component $r$ consists of three vectors $\mathbf{v}_{c,r}^X, \mathbf{v}_{c,r}^Y, \mathbf{v}_{c,r}^Z$, three matrices $\mathbf{M}_{c,r}^{YZ}, \mathbf{M}_{c,r}^{XZ}, \mathbf{M}_{c,r}^{XY}$, and three feature vectors $\mathbf{b}_{3r-2}, \mathbf{b}_{3r-1}, \mathbf{b}_{3r} \in \mathbb{R}^P$.
    *   **Global Appearance Dictionary ($\mathbf{B}$):** A matrix of size $P \times 3R_c$ formed by stacking all $\mathbf{b}$ vectors as columns [S1][S5, unofficial].

### Parameters
*   $I, J, K$: Grid resolutions along X, Y, Z axes.
*   $P$: Number of appearance feature channels. Typical value: 27 (3rd-order SH coefficients for RGB) [S1][S5, unofficial].
*   $R_\sigma, R_c$: Number of components for geometry and appearance. Typically $R_\sigma, R_c \ll I, J, K$ [S1][S5, unofficial].
*   $Q$: Number of shading points sampled per ray.
*   $N_0$: Initial low-resolution grid dimension for coarse-to-fine reconstruction. Typical value: 128 [S1][S5, unofficial].
*   Optimizer: Adam [S1][S5, unofficial].
    *   Learning rate for tensor factors: 0.02 [S1][S5, unofficial].
    *   Learning rate for MLP decoder (if used): 0.001 [S1][S5, unofficial].
*   Batch size: 4096 pixel rays [S1][S5, unofficial].
*   Hardware: Single Tesla V100 GPU (16GB) [S1][S5, unofficial].

### Algorithm Steps

1.  **Initialization:** Initialize tensor factors for $G_\sigma$ and $G_c$ with random values. Set initial grid resolution $N_0^3$ (e.g., $128^3$) [S1][S5, unofficial].
2.  **Coarse-to-Fine Upsampling:** During optimization, upsample the vector and matrix factors at specific steps (e.g., 2000, 3000, 4000, 5500, 7000) to increase grid resolution from $N_0^3$ to $N^3$. Interpolation is linear for vectors and bilinear for matrices [S1][S5, unofficial].
3.  **Ray Sampling:** For each ray $\mathbf{r}$, sample $Q$ points $\mathbf{x}_q$ along the ray segment within the scene bounding box. Compute step sizes $\Delta_q$ [S1][S5, unofficial].
4.  **Feature Evaluation (Trilinear Interpolation):** For each sample point $\mathbf{x} = (x, y, z)$, compute the interpolated geometry and appearance features using the factorized tensors.
    *   **Geometry Density:**
        $$ \sigma(\mathbf{x}) = \mathcal{G}_\sigma(\mathbf{x}) = \sum_{r=1}^{R_\sigma} \sum_{m \in \{X,Y,Z\}} \mathcal{A}_{\sigma,r}^m(\mathbf{x}) $$
        where for VM decomposition:
        $$ \mathcal{A}_{\sigma,r}^X(\mathbf{x}) = \mathbf{v}_{\sigma,r}^X(x) \cdot \mathbf{M}_{\sigma,r}^{YZ}(y, z) $$
        $$ \mathcal{A}_{\sigma,r}^Y(\mathbf{x}) = \mathbf{v}_{\sigma,r}^Y(y) \cdot \mathbf{M}_{\sigma,r}^{XZ}(x, z) $$
        $$ \mathcal{A}_{\sigma,r}^Z(\mathbf{x}) = \mathbf{v}_{\sigma,r}^Z(z) \cdot \mathbf{M}_{\sigma,r}^{XY}(x, y) $$
        Here, $\mathbf{v}(t)$ denotes linear interpolation of the vector at coordinate $t$, and $\mathbf{M}(t_1, t_2)$ denotes bilinear interpolation of the matrix at coordinates $(t_1, t_2)$ [S1][S5, unofficial].
        *Note: For CP decomposition, $\mathcal{A}_{\sigma,r}^X(\mathbf{x}) = \mathbf{v}_{\sigma,r}^X(x)\mathbf{v}_{\sigma,r}^Y(y)\mathbf{v}_{\sigma,r}^Z(z)$ is not used; instead, the CP sum is $\sum_r \mathbf{v}_{\sigma,r}^X(x)\mathbf{v}_{\sigma,r}^Y(y)\mathbf{v}_{\sigma,r}^Z(z)$ directly.*
    *   **Appearance Features:**
        $$ \mathcal{G}_c(\mathbf{x}) = \mathbf{B} \left( \bigoplus_{r=1}^{R_c} \bigoplus_{m \in \{X,Y,Z\}} \mathcal{A}_{c,r}^m(\mathbf{x}) \right) $$
        where $\mathcal{A}_{c,r}^m(\mathbf{x})$ is computed similarly to geometry but using appearance factors, and $\bigoplus$ denotes concatenation into a $3R_c$-dimensional vector [S1][S5, unofficial].
5.  **Color Decoding:** Compute view-dependent color $c(\mathbf{x}, \mathbf{d})$ using the decoded features $\mathcal{G}_c(\mathbf{x})$ and viewing direction $\mathbf{d}$:
    $$ c(\mathbf{x}, \mathbf{d}) = S(\mathcal{G}_c(\mathbf{x}), \mathbf{d}) $$
    $S$ is either a Spherical Harmonics (SH) function (using 3rd-order coefficients) or a small MLP (two fully connected layers with 128 hidden units and ReLU activation) [S1][S5, unofficial].
6.  **Volume Rendering:** Compute the pixel color $\hat{C}(\mathbf{r})$ using the standard differentiable volume rendering equation:
    $$ \hat{C}(\mathbf{r}) = \sum_{q=1}^Q \tau_q (1 - \exp(-\sigma_q \Delta_q)) c_q $$
    $$ \tau_q = \exp\left(-\sum_{p=1}^{q-1} \sigma_p \Delta_p\right) $$
    where $\sigma_q = \sigma(\mathbf{x}_q)$ and $c_q = c(\mathbf{x}_q, \mathbf{d})$ [S1][S5, unofficial].
7.  **Loss Computation and Optimization:**
    *   Compute L2 rendering loss between $\hat{C}(\mathbf{r})$ and ground truth pixel color.
    *   Add regularization terms: L1 norm loss on tensor factors (for sparsity) and/or Total Variation (TV) loss on factors (for smoothness). TV loss is preferred for datasets with few views or imperfect capture (e.g., LLFF, Tanks and Temples) [S1][S5, unofficial].
    *   Update tensor factors using Adam optimizer [S1][S5, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class TensoRF:
    def __init__(self, I, J, K, P, R_sigma, R_c, use_vm=True, use_sh=True):
        """
        Initialize TensoRF model.
        I, J, K: Grid dimensions.
        P: Appearance feature channels.
        R_sigma: Number of geometry components.
        R_c: Number of appearance components.
        use_vm: Use Vector-Matrix decomposition (True) or CP (False).
        use_sh: Use Spherical Harmonics (True) or MLP (False) for color decoding.
        """
        self.I, self.J, self.K = I, J, K
        self.P = P
        self.R_sigma = R_sigma
        self.R_c = R_c
        self.use_vm = use_vm
        self.use_sh = use_sh
        
        # Geometry Factors (G_sigma)
        if use_vm:
            # Vectors
            self.v_sigma_X = nn.Parameter(torch.randn(R_sigma, I))
            self.v_sigma_Y = nn.Parameter(torch.randn(R_sigma, J))
            self.v_sigma_Z = nn.Parameter(torch.randn(R_sigma, K))
            # Matrices
            self.M_sigma_YZ = nn.Parameter(torch.randn(R_sigma, J, K))
            self.M_sigma_XZ = nn.Parameter(torch.randn(R_sigma, I, K))
            self.M_sigma_XY = nn.Parameter(torch.randn(R_sigma, I, J))
        else:
            # CP: Only vectors
            self.v_sigma_X = nn.Parameter(torch.randn(R_sigma, I))
            self.v_sigma_Y = nn.Parameter(torch.randn(R_sigma, J))
            self.v_sigma_Z = nn.Parameter(torch.randn(R_sigma, K))

        # Appearance Factors (G_c)
        # Vectors
        self.v_c_X = nn.Parameter(torch.randn(R_c, I))
        self.v_c_Y = nn.Parameter(torch.randn(R_c, J))
        self.v_c_Z = nn.Parameter(torch.randn(R_c, K))
        
        if use_vm:
            # Matrices
            self.M_c_YZ = nn.Parameter(torch.randn(R_c, J, K))
            self.M_c_XZ = nn.Parameter(torch.randn(R_c, I, K))
            self.M_c_XY = nn.Parameter(torch.randn(R_c, I, J))
        
        # Global Appearance Dictionary B (P x 3*R_c)
        self.B = nn.Parameter(torch.randn(P, 3 * R_c))
        
        # Color Decoder
        if use_sh:
            self.decoder = SHDecoder(P) # Simplified SH implementation
        else:
            self.decoder = nn.Sequential(
                nn.Linear(P, 128),
                nn.ReLU(),
                nn.Linear(128, 128),
                nn.ReLU(),
                nn.Linear(128, 3) # RGB
            )

    def linear_interp(self, vec, coord):
        """Linear interpolation of a 1D vector at coordinate coord."""
        # coord is in [0, dim-1]
        idx0 = coord.long()
        idx1 = (idx0 + 1) % vec.size(1)
        weight = coord - idx0.float()
        val0 = vec[:, idx0]
        val1 = vec[:, idx1]
        return val0 * (1 - weight) + val1 * weight

    def bilinear_interp(self, mat, coord_y, coord_z):
        """Bilinear interpolation of a 2D matrix at coordinates (y, z)."""
        # coord_y, coord_z in [0, dim-1]
        y0 = coord_y.long()
        y1 = (y0 + 1) % mat.size(1)
        z0 = coord_z.long()
        z1 = (z0 + 1) % mat.size(2)
        
        wy = coord_y - y0.float()
        wz = coord_z - z0.float()
        
        val00 = mat[:, y0, z0]
        val01 = mat[:, y0, z1]
        val10 = mat[:, y1, z0]
        val11 = mat[:, y1, z1]
        
        val0 = val00 * (1 - wz) + val01 * wz
        val1 = val10 * (1 - wz) + val11 * wz
        return val0 * (1 - wy) + val1 * wy

    def compute_density(self, x, y, z):
        """Compute density sigma(x,y,z) using VM or CP decomposition."""
        sigma = 0.0
        if self.use_vm:
            # Term X: v_X(x) * M_YZ(y,z)
            v_x = self.linear_interp(self.v_sigma_X, x) # (R_sigma,)
            m_yz = self.bilinear_interp(self.M_sigma_YZ, y, z) # (R_sigma,)
            sigma += torch.sum(v_x * m_yz, dim=0)
            
            # Term Y: v_Y(y) * M_XZ(x,z)
            v_y = self.linear_interp(self.v_sigma_Y, y)
            m_xz = self.bilinear_interp(self.M_sigma_XZ, x, z)
            sigma += torch.sum(v_y * m_xz, dim=0)
            
            # Term Z: v_Z(z) * M_XY(x,y)
            v_z = self.linear_interp(self.v_sigma_Z, z)
            m_xy = self.bilinear_interp(self.M_sigma_XY, x, y)
            sigma += torch.sum(v_z * m_xy, dim=0)
        else:
            # CP Decomposition
            v_x = self.linear_interp(self.v_sigma_X, x)
            v_y = self.linear_interp(self.v_sigma_Y, y)
            v_z = self.linear_interp(self.v_sigma_Z, z)
            # Outer product sum
            sigma = torch.sum(v_x * v_y * v_z, dim=0)
        return sigma

    def compute_appearance_features(self, x, y, z):
        """Compute appearance feature vector G_c(x,y,z) using VM decomposition."""
        # Concatenate all A_c factors
        features_list = []
        
        # Term X: v_c_X(x) * M_c_YZ(y,z)
        v_x = self.linear_interp(self.v_c_X, x)
        m_yz = self.bilinear_interp(self.M_c_YZ, y, z)
        features_list.append(v_x * m_yz)
        
        # Term Y: v_c_Y(y) * M_c_XZ(x,z)
        v_y = self.linear_interp(self.v_c_Y, y)
        m_xz = self.bilinear_interp(self.M_c_XZ, x, z)
        features_list.append(v_y * m_xz)
        
        # Term Z: v_c_Z(z) * M_c_XY(x,y)
        v_z = self.linear_interp(self.v_c_Z, z)
        m_xy = self.bilinear_interp(self.M_c_XY, x, y)
        features_list.append(v_z * m_xy)
        
        # Concatenate along feature dimension
        # features_list contains R_c vectors of size (R_c,) -> shape (3, R_c)
        # We need to stack them to get (3*R_c,)
        all_features = torch.cat(features_list, dim=0) # (3*R_c,)
        
        # Multiply by Global Appearance Dictionary B
        # B is (P, 3*R_c)
        features = torch.matmul(self.B, all_features) # (P,)
        return features

    def render_ray(self, ray_origin, ray_direction, num_samples):
        """Render a single ray."""
        # Sample points along ray
        t_vals = torch.linspace(0, 1, num_samples)
        points = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(0).unsqueeze(-1)
        
        # Convert to grid coordinates (assuming bounding box is normalized or known)
        # This is a placeholder for coordinate transformation logic
        x, y, z = points[:, 0], points[:, 1], points[:, 2]
        
        # Compute densities
        sigmas = self.compute_density(x, y, z)
        
        # Compute transmittance
        deltas = torch.diff(torch.cat([torch.zeros_like(t_vals), t_vals], dim=0))
        alpha = 1 - torch.exp(-sigmas * deltas)
        T = torch.cumprod(torch.cat([torch.ones_like(t_vals[:1]), 1 - alpha[:-1] + 1e-10], dim=0), dim=0)
        
        # Compute colors
        # Get viewing direction (constant for the ray)
        # Compute appearance features
        app_features = self.compute_appearance_features(x, y, z)
        colors = self.decoder(app_features, ray_direction) # (num_samples, 3)
        
        # Volume rendering
        weights = alpha * T
        color = torch.sum(weights.unsqueeze(-1) * colors, dim=0)
        
        return color

    def forward(self, rays, gt_colors):
        """Forward pass for a batch of rays."""
        pred_colors = []
        for ray, gt in zip(rays, gt_colors):
            # ray: (origin, direction)
            color = self.render_ray(ray.origin, ray.direction, 128) # 128 samples
            pred_colors.append(color)
        
        pred_colors = torch.stack(pred_colors)
        loss = F.mse_loss(pred_colors, gt_colors)
        
        # Regularization
        reg_loss = 0.0
        # L1 on factors
        for param in self.parameters():
            reg_loss += torch.norm(param, p=1)
        
        # TV loss on factors (simplified)
        # ... compute TV on v_sigma_X, M_sigma_YZ, etc. ...
        
        return loss + reg_loss

class SHDecoder(nn.Module):
    def __init__(self, P):
        super().__init__()
        self.P = P
        # SH coefficients for 3rd order (9 coefficients per channel)
        # P=27 implies 9 coefficients * 3 channels
        self.coeffs = nn.Parameter(torch.randn(3, 9))
    
    def forward(self, features, direction):
        # features: (batch, P)
        # direction: (batch, 3)
        # Compute SH basis for direction
        # ...
        # Return RGB
        return torch.zeros_like(features[:, :3]) # Placeholder
```

## Subtopics

### [CasCPRF](7415-cascprf.md)
A multi-scale tensor decomposition radiance field representation that achieves superior compactness compared to TensoRF by employing a cascaded scheme to separate frequencies across multiple hierarchical levels.

### [Simple-TensoRF Augmentation](7596-simple-tensorf-augmentation.md)
A regularization method for TensoRF designed to mitigate overfitting and geometric distortions when training with sparse input views by using a secondary, lower-capacity model for depth priors.

### [Generalized Tensorial Radiance Fields (GTensoRF)](8560-generalized-tensorial-radiance-fields-gtensorf.md)
A framework that learns multiple scenes simultaneously using higher-order tensor decomposition, shifting from scene-specific to generalizable radiance fields.

## Operating characteristics
**Time Complexity:** Reconstruction time is significantly faster than NeRF. For example, TensoRF-VM-192 on Synthetic NeRF takes ~17.4 minutes for 30k steps, compared to NeRF's ~35 hours [S1][S5, unofficial].
**Space Complexity:** Reduced from $O(n^3)$ to $O(n)$ with CP decomposition or $O(n^2)$ with VM decomposition [S3]. Model sizes are compact: TensoRF-CP-384 is <4MB, TensoRF-VM-192 is ~72MB [S1][S5, unofficial].
**Performance:** On Synthetic NeRF, TensoRF-VM-192 achieves PSNR 33.14 and SSIM 0.963 [S1][S5, unofficial]. On NSVF, it achieves PSNR 36.52 and SSIM 0.982 [S1][S5, unofficial]. On Tanks and Temples, it achieves PSNR 28.56 and SSIM 0.920 [S1][S5, unofficial].
**Strengths:** Fast convergence, compact model size, no need for customized CUDA kernels (standard PyTorch implementation) [S3][S5, unofficial].
**Limitations:** May suffer from overfitting and local minima with few views, requiring regularization (TV loss) [S1]. VM decomposition is more complex to implement than CP [S1].

## Background
TensoRF was introduced by Anpei Chen, Zexiang Xu, Andreas Geiger, Jingyi Yu, and Hao Su in 2022 at the European Conference on Computer Vision (ECCV) [S1, S3]. It addresses the limitations of NeRF, which relies on MLPs for scene representation, leading to high memory usage and slow training times. TensoRF proposes modeling the radiance field as a factorized tensor, enabling efficient scene modeling and reconstruction [S1, S3].

## Sources
[S1] TensoRF: Tensorial Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_20, fetched 2026-10-26
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S3] TensoRF: Tensorial Radiance Fields - GitHub Pages, https://apchenstu.github.io/TensoRF/, fetched 2026-10-29
[S4] PDF TensoRF: Tensorial Radiance Fields - ECVA, https://www.ecva.net/papers/eccv_2022/papers_ECCV/papers/136920332.pdf, fetched 2026-10-29 (unofficial)
[S5] Tensorial Radiance Fields - TensoRF, https://www.cvlibs.net/publications/Chen2022ECCV.pdf, fetched 2026-10-29 (unofficial)
