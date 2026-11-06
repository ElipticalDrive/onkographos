# TensoRF

Also called: TensoRF (Tensorial Radiance Field Representation), TensoRF Representation, Tensorial Radiance Field Representation, Tensorial Radiance Fields

## Overview
TensoRF represents a radiance field as a factorized tensorial grid to achieve compact modeling and fast reconstruction [S1][S3, unofficial]. It solves the memory and computational bottlenecks of dense voxel grids by decomposing the geometry and appearance fields into low-rank components using Canonical Polyadic (CP) or Vector-Matrix (VM) factorization [S1][S3, unofficial]. Unlike [Neural Radiance Fields](4972-neural-radiance-fields.md) which use MLPs, or [Plenoxels](5942-plenoxels.md) which use static grids, TensoRF uses continuous, differentiable tensor factors that are optimized via gradient descent [S1][S3, unofficial].

## How it works
**Inputs:** Multi-view images with known camera poses.
**Outputs:** Optimized tensor factors representing volume density $\sigma$ and view-dependent color $c$.

### Data Structures
The radiance field is modeled by two regular 3D grids, factorized into vectors and matrices:

1.  **Geometry Grid ($G_\sigma$):** A 3D tensor $\mathbb{R}^{I \times J \times K}$ storing single-channel volume density values directly [S1][S3, unofficial].
    *   Factorized into $R_\sigma$ components.
    *   For **VM decomposition**: Each component $r$ consists of three vectors $\mathbf{v}_{\sigma,r}^X, \mathbf{v}_{\sigma,r}^Y, \mathbf{v}_{\sigma,r}^Z$ (sizes $I, J, K$) and three matrices $\mathbf{M}_{\sigma,r}^{YZ}, \mathbf{M}_{\sigma,r}^{XZ}, \mathbf{M}_{\sigma,r}^{XY}$ (sizes $J \times K, I \times K, I \times J$) [S1][S3, unofficial].
    *   For **CP decomposition**: Each component $r$ consists of three vectors $\mathbf{v}_{\sigma,r}^X, \mathbf{v}_{\sigma,r}^Y, \mathbf{v}_{\sigma,r}^Z$ (sizes $I, J, K$) [S1][S3, unofficial].
2.  **Appearance Grid ($G_c$):** A 4D tensor $\mathbb{R}^{I \times J \times K \times P}$ storing $P$ appearance feature channels [S1][S3, unofficial].
    *   Factorized into $R_c$ components.
    *   For **VM decomposition**: Each component $r$ consists of three vectors $\mathbf{v}_{c,r}^X, \mathbf{v}_{c,r}^Y, \mathbf{v}_{c,r}^Z$ (sizes $I, J, K$), three matrices $\mathbf{M}_{c,r}^{YZ}, \mathbf{M}_{c,r}^{XZ}, \mathbf{M}_{c,r}^{XY}$ (sizes $J \times K, I \times K, I \times J$), and three vectors $\mathbf{b}_{3r-2}, \mathbf{b}_{3r-1}, \mathbf{b}_{3r}$ (size $P$) [S1][S3, unofficial].
    *   For **CP decomposition**: Each component $r$ consists of three vectors $\mathbf{v}_{c,r}^X, \mathbf{v}_{c,r}^Y, \mathbf{v}_{c,r}^Z$ (sizes $I, J, K$) and one vector $\mathbf{b}_r$ (size $P$) [S1][S3, unofficial].
3.  **Global Appearance Dictionary ($B$):** A matrix $\mathbb{R}^{P \times 3R_c}$ formed by stacking all $\mathbf{b}_r$ vectors as columns [S1][S3, unofficial].

**Parameters:**
*   $I, J, K$: Grid resolutions along X, Y, Z axes.
*   $P$: Number of appearance feature channels. Typical value: 27 [S1][S3, unofficial].
*   $R_\sigma, R_c$: Number of components for geometry and appearance. Typically $R_\sigma, R_c \ll I, J, K$ [S1][S3, unofficial].
*   $Q$: Number of shading points sampled per ray [S1][S3, unofficial].
*   Optimizer: Adam [S1][S3, unofficial].
    *   Learning rate for tensor factors: 0.02 [S1][S3, unofficial].
    *   Learning rate for MLP decoder (if used): 0.001 [S1][S3, unofficial].
*   Batch size: 4096 pixel rays [S1][S3, unofficial].
*   Coarse-to-fine: Initial resolution $N_0 = 128$ ($N_0^3$ voxels). Upsampling steps: 2000, 3000, 4000, 5500, 7000 [S1][S3, unofficial].

### Method Steps

1.  **Initialization:** Initialize tensor factors for $G_\sigma$ and $G_c$ at low resolution $N_0^3$ [S1][S3, unofficial].
2.  **Sampling:** For each ray, sample $Q$ points $\mathbf{x}_q$ along the ray [S1][S3, unofficial].
3.  **Feature Evaluation:** Compute trilinearly interpolated features for each sample point $\mathbf{x}$:
    *   Compute component values $\mathcal{A}_{\sigma,r}^m(\mathbf{x})$ and $\mathcal{A}_{c,r}^m(\mathbf{x})$ using interpolated vector/matrix factors (see Formulas).
    *   Compute density $\sigma(\mathbf{x})$ and appearance features $\mathcal{G}_c(\mathbf{x})$ (see Formulas).
4.  **Color Decoding:** Convert appearance features to color $c(\mathbf{x})$ using function $S$:
    *   **Spherical Harmonics (SH):** 3rd-order SH coefficients with RGB channels ($P=27$) [S1][S3, unofficial].
    *   **MLP:** Two fully connected layers with 128 hidden units and ReLU activation [S1][S3, unofficial].
5.  **Volume Rendering:** Compute pixel color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model [S1][S3, unofficial]:
    $$ \hat{C}(\mathbf{r}) = \sum_{q=1}^Q \tau_q (1 - \exp(-\sigma_q \Delta_q)) c_q $$
    where $\tau_q = \exp\left(-\sum_{p=1}^{q-1} \sigma_p \Delta_p\right)$ [S1][S3, unofficial].
6.  **Loss Computation:** Compute L2 rendering loss between $\hat{C}(\mathbf{r})$ and ground truth pixel colors [S1][S3, unofficial].
7.  **Regularization:** Add L1 norm loss and/or Total Variation (TV) loss on vector and matrix factors to prevent overfitting [S1][S3, unofficial].
    *   L1 sparsity loss is adequate for most datasets [S1][S3, unofficial].
    *   TV loss is used for real datasets with few images (e.g., LLFF) or imperfect capture (e.g., Tanks and Temples) [S1][S3, unofficial].
8.  **Optimization:** Update tensor factors using Adam optimizer [S1][S3, unofficial].
9.  **Coarse-to-Fine Upsampling:** At specified steps (2000, 3000, 4000, 5500, 7000), upsample vectors and matrices linearly/bilinearly to increase grid resolution from $N_0^3$ to $N^3$, interpolating voxel counts linearly in logarithmic space [S1][S3, unofficial].

### Formulas

**Trilinear Interpolation of Components:**
For a component $\mathcal{A}_r^X = \mathbf{v}_r^X \circ \mathbf{M}_r^{YZ}$, the interpolated value at $\mathbf{x}=(x,y,z)$ is:
$$ \mathcal{A}_r^X(\mathbf{x}) = \mathbf{v}_r^X(x) \mathbf{M}_r^{YZ}(y, z) $$
where $\mathbf{v}_r^X(x)$ is linearly interpolated along X, and $\mathbf{M}_r^{YZ}(y, z)$ is bilinearly interpolated in the YZ plane [S1][S3, unofficial].
Similarly:
$$ \mathcal{A}_r^Y(\mathbf{x}) = \mathbf{v}_r^Y(y) \mathbf{M}_r^{XZ}(x, z) $$
$$ \mathcal{A}_r^Z(\mathbf{x}) = \mathbf{v}_r^Z(z) \mathbf{M}_r^{XY}(x, y) $$
For CP decomposition: $\mathcal{A}_r(\mathbf{x}) = \mathbf{v}_r^X(x) \mathbf{v}_r^Y(y) \mathbf{v}_r^Z(z)$ [S1][S3, unofficial].

**Density Evaluation (VM):**
$$ \mathcal{G}_\sigma(\mathbf{x}) = \sum_{r=1}^{R_\sigma} \sum_{m \in XYZ} \mathcal{A}_{\sigma,r}^m(\mathbf{x}) \quad (13) $$
For voxel $ijk$: $\mathcal{G}_{\sigma,ijk} = \sum_{r=1}^{R_\sigma} \sum_{m \in XYZ} \mathcal{A}_{\sigma,r,ijk}^m$ [S1][S3, unofficial].

**Appearance Evaluation (VM):**
$$ \mathcal{G}_c(\mathbf{x}) = \mathbf{B}(\oplus[\mathcal{A}_{c,r}^m(\mathbf{x})]_{m,r}) \quad (14) $$
where $\oplus$ concatenates scalars $\mathcal{A}_{c,r,ijk}^m$ for $m=X,Y,Z$ and $r=1..R_c$ into a $3R_c$-dimensional vector, and $\mathbf{B} \in \mathbb{R}^{P \times 3R_c}$ is the global appearance dictionary [S1][S3, unofficial].
For voxel $ijk$: $\mathcal{G}_{c,ijk} = \sum_{r=1}^{R_c} \mathcal{A}_{c,r,ijk}^X \mathbf{b}_{3r-2} + \mathcal{A}_{c,r,ijk}^Y \mathbf{b}_{3r-1} + \mathcal{A}_{c,r,ijk}^Z \mathbf{b}_{3r}$ [S1][S3, unofficial].

**Final Radiance Field:**
$$ \sigma, c = \sum_r \sum_m \mathcal{A}_{\sigma,r}^m(\mathbf{x}), S(\mathbf{B}(\oplus[\mathcal{A}_{c,r}^m(\mathbf{x})]_{m,r}), d) \quad (15) $$
where $S$ is the decoding function (MLP or SH) [S1][S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class TensoRF:
    def __init__(self, I, J, K, P, R_sigma, R_c, use_vm=True, decoder_type='mlp'):
        """
        Initialize TensoRF model.
        I, J, K: Grid dimensions.
        P: Appearance feature channels (default 27).
        R_sigma, R_c: Number of components.
        use_vm: Use Vector-Matrix decomposition (True) or CP (False).
        decoder_type: 'mlp' or 'sh'.
        """
        self.I, self.J, self.K = I, J, K
        self.P = P
        self.R_sigma = R_sigma
        self.R_c = R_c
        self.use_vm = use_vm
        self.decoder_type = decoder_type
        
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
        if use_vm:
            # Vectors
            self.v_c_X = nn.Parameter(torch.randn(R_c, I))
            self.v_c_Y = nn.Parameter(torch.randn(R_c, J))
            self.v_c_Z = nn.Parameter(torch.randn(R_c, K))
            # Matrices
            self.M_c_YZ = nn.Parameter(torch.randn(R_c, J, K))
            self.M_c_XZ = nn.Parameter(torch.randn(R_c, I, K))
            self.M_c_XY = nn.Parameter(torch.randn(R_c, I, J))
            # Feature mode vectors (stacked into B)
            self.b_vectors = nn.Parameter(torch.randn(3 * R_c, P))
        else:
            # CP: Vectors and single b vector per component
            self.v_c_X = nn.Parameter(torch.randn(R_c, I))
            self.v_c_Y = nn.Parameter(torch.randn(R_c, J))
            self.v_c_Z = nn.Parameter(torch.randn(R_c, K))
            self.b_vectors = nn.Parameter(torch.randn(R_c, P))

        # Decoder
        if decoder_type == 'mlp':
            self.decoder = nn.Sequential(
                nn.Linear(P, 128),
                nn.ReLU(),
                nn.Linear(128, 128),
                nn.ReLU(),
                nn.Linear(128, 3) # RGB
            )
        else:
            # SH Decoder: P=27 corresponds to 3rd order SH for RGB
            # Pre-compute SH basis for viewing directions if needed, 
            # but here we assume S takes features and direction.
            # Simplified: SH coefficients are stored in b_vectors or separate grid?
            # Sources say G_c contains SH coefficients. 
            # In VM, b_vectors are global. For SH, we might need per-voxel SH coeffs.
            # However, TensoRF paper states G_c contains SH coefficients and uses global B.
            # Implementation detail: S is applied to G_c(x).
            # For SH, S(features, d) = sum(features * SH_basis(d)).
            # We will implement a placeholder SH evaluation.
            self.decoder = None # Handled in forward

    def linear_interp(self, vec, coord, dim_size):
        """Linear interpolation of a 1D vector at coordinate coord."""
        # coord in [0, dim_size-1]
        idx0 = torch.floor(coord).long().clamp(0, dim_size - 2)
        idx1 = idx0 + 1
        weight = coord - idx0.float()
        val0 = vec[idx0]
        val1 = vec[idx1]
        return val0 * (1 - weight) + val1 * weight

    def bilinear_interp(self, mat, coord_y, coord_z, dim_y, dim_z):
        """Bilinear interpolation of a 2D matrix at (y, z)."""
        idx0_y = torch.floor(coord_y).long().clamp(0, dim_y - 2)
        idx1_y = idx0_y + 1
        w_y = coord_y - idx0_y.float()
        
        idx0_z = torch.floor(coord_z).long().clamp(0, dim_z - 2)
        idx1_z = idx0_z + 1
        w_z = coord_z - idx0_z.float()
        
        # mat shape: (..., dim_y, dim_z)
        # We need to interpolate along last two dims
        # val00 = mat[..., idx0_y, idx0_z]
        # val01 = mat[..., idx0_y, idx1_z]
        # val10 = mat[..., idx1_y, idx0_z]
        # val11 = mat[..., idx1_y, idx1_z]
        
        # Efficient interpolation
        v0 = torch.lerp(mat[..., idx0_y, :], mat[..., idx1_y, :], w_y.unsqueeze(-1))
        return torch.lerp(v0[..., idx0_z], v0[..., idx1_z], w_z)

    def get_sigma(self, x):
        """Compute density sigma at points x (N, 3)."""
        x_c, y_c, z_c = x[:, 0], x[:, 1], x[:, 2]
        sigma = 0.0
        
        if self.use_vm:
            # X mode: v_X(x) * M_YZ(y, z)
            v_X = self.linear_interp(self.v_sigma_X, x_c, self.I) # (R, N)
            M_YZ = self.bilinear_interp(self.M_sigma_YZ, y_c, z_c, self.J, self.K) # (R, N)
            sigma += torch.sum(v_X * M_YZ, dim=0)
            
            # Y mode: v_Y(y) * M_XZ(x, z)
            v_Y = self.linear_interp(self.v_sigma_Y, y_c, self.J)
            M_XZ = self.bilinear_interp(self.M_sigma_XZ, x_c, z_c, self.I, self.K)
            sigma += torch.sum(v_Y * M_XZ, dim=0)
            
            # Z mode: v_Z(z) * M_XY(x, y)
            v_Z = self.linear_interp(self.v_sigma_Z, z_c, self.K)
            M_XY = self.bilinear_interp(self.M_sigma_XY, x_c, y_c, self.I, self.J)
            sigma += torch.sum(v_Z * M_XY, dim=0)
        else:
            # CP
            v_X = self.linear_interp(self.v_sigma_X, x_c, self.I)
            v_Y = self.linear_interp(self.v_sigma_Y, y_c, self.J)
            v_Z = self.linear_interp(self.v_sigma_Z, z_c, self.K)
            # Outer product sum
            # sigma = sum_r v_X[r] * v_Y[r] * v_Z[r]
            sigma = torch.sum(v_X * v_Y * v_Z, dim=0)
            
        return sigma

    def get_color_features(self, x):
        """Compute appearance feature vector G_c(x) at points x (N, 3)."""
        x_c, y_c, z_c = x[:, 0], x[:, 1], x[:, 2]
        features_concat = [] # List of (N,) tensors
        
        if self.use_vm:
            for r in range(self.R_c):
                # X mode
                v_X = self.linear_interp(self.v_c_X[r], x_c, self.I)
                M_YZ = self.bilinear_interp(self.M_c_YZ[r], y_c, z_c, self.J, self.K)
                features_concat.append(v_X * M_YZ)
                
                # Y mode
                v_Y = self.linear_interp(self.v_c_Y[r], y_c, self.J)
                M_XZ = self.bilinear_interp(self.M_c_XZ[r], x_c, z_c, self.I, self.K)
                features_concat.append(v_Y * M_XZ)
                
                # Z mode
                v_Z = self.linear_interp(self.v_c_Z[r], z_c, self.K)
                M_XY = self.bilinear_interp(self.M_c_XY[r], x_c, y_c, self.I, self.J)
                features_concat.append(v_Z * M_XY)
        else:
            for r in range(self.R_c):
                v_X = self.linear_interp(self.v_c_X[r], x_c, self.I)
                v_Y = self.linear_interp(self.v_c_Y[r], y_c, self.J)
                v_Z = self.linear_interp(self.v_c_Z[r], z_c, self.K)
                features_concat.append(v_X * v_Y * v_Z)
                
        # Concatenate all scalars into (N, 3*R_c)
        A_vec = torch.stack(features_concat, dim=1) # (N, 3*R_c)
        
        # Multiply by B (3*R_c, P) -> (N, P)
        # B is stored as b_vectors (3*R_c, P)
        G_c = torch.matmul(A_vec, self.b_vectors) # (N, P)
        
        return G_c

    def forward(self, x, d):
        """
        x: (N, 3) coordinates
        d: (N, 3) viewing directions
        Returns: sigma (N,), color (N, 3)
        """
        sigma = self.get_sigma(x)
        G_c = self.get_color_features(x)
        
        if self.decoder_type == 'mlp':
            color = self.decoder(G_c)
        else:
            # SH Decoder
            # P=27. SH basis for direction d.
            # SH basis calculation for 3rd order (9 coefficients per channel)
            # Total 27 coefficients for RGB.
            # We assume G_c contains the coefficients directly.
            # Color = sum(G_c * SH_basis(d))
            # Implementing SH basis evaluation
            x_d, y_d, z_d = d[:, 0], d[:, 1], d[:, 2]
            # 3rd order SH basis (9 terms)
            # c0 = 0.282095
            # c1 = 0.488603 * z
            # c2 = 1.092548 * x * y
            # c3 = 0.315392 * (3*z^2 - 1)
            # c4 = 1.092548 * y * z
            # c5 = 0.546274 * (x^2 - y^2)
            # c6 = 1.092548 * x * z
            # c7 = 0.737676 * x
            # c8 = 0.737676 * y
            # ... standard SH basis ...
            # For brevity, assuming a helper function or precomputed basis
            # Since we must be self-contained, we compute it.
            
            # Normalize d
            d_norm = torch.norm(d, dim=1, keepdim=True)
            nx, ny, nz = d[:, 0]/d_norm, d[:, 1]/d_norm, d[:, 2]/d_norm
            
            # SH Basis (9 components)
            sh0 = torch.full_like(nx, 0.282095)
            sh1 = 0.488603 * nz
            sh2 = 1.092548 * nx * ny
            sh3 = 0.315392 * (3 * nz**2 - 1)
            sh4 = 1.092548 * ny * nz
            sh5 = 0.546274 * (nx**2 - ny**2)
            sh6 = 1.092548 * nx * nz
            sh7 = 0.737676 * nx
            sh8 = 0.737676 * ny
            
            sh_basis = torch.stack([sh0, sh1, sh2, sh3, sh4, sh5, sh6, sh7, sh8], dim=1) # (N, 9)
            
            # G_c is (N, 27). Reshape to (N, 3, 9) for R, G, B
            G_c_reshaped = G_c.view(-1, 3, 9)
            
            # Color = sum(G_c * sh_basis) over SH coeffs
            color = torch.sum(G_c_reshaped * sh_basis.unsqueeze(1), dim=2) # (N, 3)
            
        return sigma, color

    def render_ray(self, ray_origins, ray_dirs, t_near, t_far, Q=128):
        """Differentiable volume rendering for a batch of rays."""
        # Sample points
        t_vals = torch.linspace(t_near, t_far, Q).unsqueeze(0) # (1, Q)
        t_vals = t_vals.expand(ray_origins.shape[0], Q) # (N_rays, Q)
        
        # Add noise for anti-aliasing (optional, standard in NeRF)
        # t_samples = t_vals + noise
        
        pts = ray_origins.unsqueeze(1) + ray_dirs.unsqueeze(1) * t_samples.unsqueeze(2) # (N_rays, Q, 3)
        
        sigma, color = self.forward(pts.reshape(-1, 3), ray_dirs.unsqueeze(1).expand(-1, Q, -1).reshape(-1, 3))
        sigma = sigma.reshape(-1, Q)
        color = color.reshape(-1, Q, 3)
        
        # Step sizes
        dists = t_samples[:, 1:] - t_samples[:, :-1]
        dists = torch.cat([dists, torch.tensor([1e10]).to(dists.device)], dim=1) # Last sample infinite
        
        # Density to opacity
        alpha = 1 - torch.exp(-sigma * dists)
        
        # Transmittance
        T = torch.cumprod(torch.exp(-sigma * dists), dim=1)
        T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
        
        # Weighted color
        weights = alpha * T
        color_out = torch.sum(weights.unsqueeze(2) * color, dim=1)
        
        return color_out
```

## Subtopics

### TensoRF-CP
Uses Canonical Polyadic decomposition. Super compact modeling (e.g., <4MB for 500^3 voxels). Lower rendering quality than VM [S1][S3, unofficial].

### TensoRF-VM
Uses Vector-Matrix decomposition. Significantly better rendering quality than CP because it compresses more parameters in each component. Recommended for most use cases [S1][S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Reconstruction is significantly faster than NeRF. VM models with 300^3 voxels and 192 components take ~17.4 minutes on Synthetic NeRF (30k steps) [S1][S3, unofficial].
*   **Memory Complexity:** Model size is compact. VM-192 (300^3 voxels) is ~71.8 MB. CP-384 (500^3 voxels) is <4 MB [S1][S3, unofficial].
*   **Performance:** On Synthetic NeRF, TensoRF-VM-192 achieves PSNR 33.14 and SSIM 0.963 [S1][S3, unofficial]. On NSVF, PSNR 36.52 [S1][S3, unofficial]. On Tanks & Temples, PSNR 28.56 [S1][S3, unofficial].
*   **Strengths:** Fast reconstruction, compact model size, high quality.
*   **Limitations:** Requires coarse-to-fine upsampling for high resolution. TV loss needed for challenging real-world datasets [S1][S3, unofficial].

## Background
TensoRF was introduced by Chen et al. in 2022 [S1][S3, unofficial]. It builds on the idea of tensor factorization for radiance fields, aiming to reduce the memory footprint and training time of NeRF-like methods [S1][S3, unofficial].

## Sources
[S1] TensoRF: Tensorial Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_20, fetched 2026-10-26
[S2] PDF TensoRF: Tensorial Radiance Fields - ECVA, https://www.ecva.net/papers/eccv_2022/papers_ECCV/papers/136920332.pdf, fetched 2026-10-29 (unofficial)
[S3] Tensorial Radiance Fields - TensoRF, https://www.cvlibs.net/publications/Chen2022ECCV.pdf, fetched 2026-10-29 (unofficial)
