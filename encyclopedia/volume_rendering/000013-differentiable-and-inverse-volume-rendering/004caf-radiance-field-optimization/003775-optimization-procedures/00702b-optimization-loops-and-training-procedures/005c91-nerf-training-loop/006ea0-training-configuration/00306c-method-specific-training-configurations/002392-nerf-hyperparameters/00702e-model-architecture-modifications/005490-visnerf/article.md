# ViSNeRF

Also called: ViSNeRF (Vector-matrix and Tensor factorization for Neural Radiance Fields)

## Overview
ViSNeRF is a hybrid neural radiance field method designed for the efficient synthesis of visualizations from dynamic volumetric scenes with arbitrary adjustable parameters (e.g., time, isovalue, transfer functions) [S1]. It solves the scalability problem of high-dimensional radiance fields by combining explicit spatial feature grids with implicit MLP decoders, utilizing block-term tensor decomposition for the 3D spatial volume and CANDECOMP/PARAFAC (CP) decomposition for the K-dimensional parameter space [S1]. Unlike sibling methods that rely solely on fully implicit networks or fully explicit grids, ViSNeRF achieves a balance of memory efficiency and visual quality by factorizing the $(3+K)$-D representation into compact components, reducing complexity from $O(N^3 M^K)$ to $O(N^2 M)$ [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of training images (DVR or IR) with known camera poses, ray origins, directions, RGB values, and scene parameters ($p_0, p_1, \dots$) defining the specific frame [S1].
*   **Outputs:** Synthesized visualization images (DVR or IR) for novel viewpoints and interpolated parameter values [S1].

**Data Structures**
The model maintains three primary explicit feature structures and two implicit decoders:

1.  **Spatial Feature Grid ($T^3$):**
    *   **Layout:** Factorized into vector-matrix pairs.
    *   **Density Components:** $R_\sigma$ components. Each component consists of a spatial matrix $M \in \mathbb{R}^{H \times W}$ (or $X \times Y$, etc.) and a spatial vector $v \in \mathbb{R}^{L}$ (where $L$ is the remaining dimension length).
    *   **Color Components:** $R_c$ components. Same structure as density.
    *   **Total Spatial Components:** $R_s = R_\sigma + R_c$.
    *   **Resolution:** Initialized at $N \times N \times N$ (e.g., $128^3$), potentially increasing to $300^3$ for complex scenes. The grid is not necessarily cubic; dimensions match the bounding box aspect ratio [S1].

2.  **Parameter Feature Grid ($T^K$):**
    *   **Layout:** Factorized into $K$ vectors per component.
    *   **Dimensions:** $M^K$, where $M$ is the number of distinct values for each parameter in the training set.
    *   **Components:** $R_p$ one-rank components. Each component is the outer product of $K$ vectors, one for each parameter dimension [S1].

3.  **Implicit Decoders:**
    *   **Color MLP ($g_c$):** One hidden layer, 128 channels.
    *   **Density MLP ($g_\sigma$):** One hidden layer, 128 channels.
    *   Both are fully connected networks [S1].

**Numbered Steps**

1.  **Ray Casting:** Cast rays from screen pixels toward the scene from camera locations. For each pixel, define the ray origin $\mathbf{o}$ and direction $\mathbf{d}$, and associate the specific scene parameters $\mathbf{p} = (p_0, p_1, \dots)$ for the target frame [S1].

2.  **Sampling:** Estimate the integral along the ray using stratified sampling. Select $n$ sample points $t_i$ along the ray segment $[t_n, t_f]$. Calculate step size $\delta_i = t_{i+1} - t_i$ [S1].

3.  **Feature Retrieval (Spatial):** For a sample point $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$:
    *   Retrieve features from the spatial factorization $T^3$.
    *   For each component $r \in \{1, \dots, R_s\}$, perform element-wise multiplication between the sampled spatial matrix $M$ and the sampled spatial vector $v$ corresponding to the coordinates of $\mathbf{x}_i$ [S1].
    *   Concatenate the outcomes of these multiplications to form the spatial feature vector $f_{\text{spatial}}$ [S1].

4.  **Feature Retrieval (Parameter):**
    *   Retrieve features from the parameter factorization $T^K$.
    *   For each component $r \in \{1, \dots, R_p\}$, perform element-wise multiplication between the sampled parameter vectors $v^1_r, v^2_r, \dots, v^K_r$ corresponding to the parameters $\mathbf{p}$ [S1].
    *   Concatenate these products to form the parameter feature vector $f_{\text{param}}$ [S1].

5.  **Feature Combination:**
    *   Combine $f_{\text{spatial}}$ and $f_{\text{param}}$. The source describes concatenating the outcomes of spatial and parameter multiplications. Specifically, the final feature input to the decoders integrates the products from the three spatial pairs (XY, XZ, YZ) and the parameter vector product [S1].
    *   Let $f_\sigma$ be the density feature and $f_c$ be the color feature derived from this combination [S1].

6.  **Decoding:**
    *   Encode the viewing direction $\mathbf{d}$ using Spherical Harmonics (SH): $E_{SH}(\mathbf{d})$ [S1].
    *   Encode the spatial position $\mathbf{x}_i$ using Positional Encoding (PE): $E_P(\mathbf{x}_i)$ [S1].
    *   Compute density: $\sigma_i = g_\sigma(f_\sigma, E_P(\mathbf{x}_i))$ [S1].
    *   Compute color: $\mathbf{c}_i = g_c(f_c, E_{SH}(\mathbf{d}))$ [S1].

7.  **Volume Rendering:**
    *   Compute transmittance: $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1].
    *   Compute ray color:
        $$ \hat{C}(r) = \sum_{i=1}^{n} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
        [S1]

8.  **Loss Computation:**
    *   Compute Reconstruction Loss ($L_{REC}$): Mean Squared Error (MSE) between $\hat{C}(r)$ and ground truth color [S1].
    *   Compute L1 Norm Loss ($L_{L1}$):
        $$ L_{L1} = \begin{cases} \sum ||1 - w|| & \text{if } W \in v_p \text{ (parameter vectors)} \\ \sum \frac{||w||_1}{|W|} & \text{otherwise} \end{cases} $$
        where $W$ are weights in vectors/matrices. Parameter vector weights are initialized/regularized to 1 [S1].
    *   Compute Total Variation Loss ($L_{TV}$):
        $$ L_{TV} = L_{TV1} + L_{TV2} $$
        $$ L_{TV1} = \frac{1}{|V||Q|l} \sum_{v,q,i} ||v_{i,q} - v_{i-1,q}||_2 $$
        $$ L_{TV2} = \frac{1}{|M||Q|hw} \sum_{M,q,i,j} \left( ||M_{q,i,j} - M_{q,i-1,j}||_2^2 + ||M_{q,i,j} - M_{q,i,j-1}||_2^2 \right) $$
        where $v$ are vectors, $M$ are matrices, $Q$ is channels, $l$ is vector length, $h,w$ are matrix dimensions [S1].
    *   Total Loss: $L = L_{REC} + \lambda_1 L_{L1} + \lambda_2 L_{TV}$ [S1].

9.  **Optimization:**
    *   Update matrices, vectors, and MLPs using Adam optimizer ($\beta_1=0.9, \beta_2=0.99$) [S1].
    *   Apply dynamic bounding box shrinking and voxel skipping starting at iteration 2,000 [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ViSNeRF:
    def __init__(self, spatial_res, param_dims, R_sigma, R_color, R_param, hidden_dim=128):
        """
        spatial_res: tuple (X, Y, Z) for spatial grid dimensions
        param_dims: tuple of lengths for each parameter dimension (M_1, M_2, ...)
        R_sigma: number of low-rank components for density
        R_color: number of low-rank components for color
        R_param: number of low-rank components for parameter space
        """
        super().__init__()
        self.R_sigma = R_sigma
        self.R_color = R_color
        self.R_param = R_param
        self.spatial_res = spatial_res
        
        # Spatial Feature Grid Factorization (Block-term)
        # Density: R_sigma components of (Matrix x Vector)
        # Matrices: M_XY, M_XZ, M_YZ. Vectors: v_X, v_Y, v_Z
        # Dimensions depend on spatial_res. Simplified for one component r:
        # M_XY: (X, Y), v_Z: (Z,) -> Outer product shape (X, Y, Z)
        # M_XZ: (X, Z), v_Y: (Y,)
        # M_YZ: (Y, Z), v_X: (X,)
        
        # Initialize Spatial Matrices and Vectors for Density
        self.density_M_XY = nn.Parameter(torch.randn(R_sigma, spatial_res[0], spatial_res[1]))
        self.density_M_XZ = nn.Parameter(torch.randn(R_sigma, spatial_res[0], spatial_res[2]))
        self.density_M_YZ = nn.Parameter(torch.randn(R_sigma, spatial_res[1], spatial_res[2]))
        self.density_v_X = nn.Parameter(torch.randn(R_sigma, spatial_res[0]))
        self.density_v_Y = nn.Parameter(torch.randn(R_sigma, spatial_res[1]))
        self.density_v_Z = nn.Parameter(torch.randn(R_sigma, spatial_res[2]))
        
        # Initialize Spatial Matrices and Vectors for Color
        self.color_M_XY = nn.Parameter(torch.randn(R_color, spatial_res[0], spatial_res[1]))
        self.color_M_XZ = nn.Parameter(torch.randn(R_color, spatial_res[0], spatial_res[2]))
        self.color_M_YZ = nn.Parameter(torch.randn(R_color, spatial_res[1], spatial_res[2]))
        self.color_v_X = nn.Parameter(torch.randn(R_color, spatial_res[0]))
        self.color_v_Y = nn.Parameter(torch.randn(R_color, spatial_res[1]))
        self.color_v_Z = nn.Parameter(torch.randn(R_color, spatial_res[2]))
        
        # Parameter Feature Grid Factorization (CP)
        # T^K = sum_r (v^1_r o v^2_r o ... o v^K_r)
        # param_dims is a tuple (M_1, M_2, ...)
        self.param_vectors = nn.ParameterList([
            nn.Parameter(torch.randn(R_param, M)) for M in param_dims
        ])
        
        # Implicit Decoders
        # Density MLP: Input is combined spatial+param features + PE(x)
        # Color MLP: Input is combined spatial+param features + SH(d)
        # Note: The paper states inputs are f_sigma/f_c (from grid) and EP(x)/ESH(d).
        # The combination of grid features is done before the MLP.
        
        # Density MLP
        self.g_sigma = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 1) # Output sigma
        )
        
        # Color MLP
        self.g_c = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # Output RGB
        )
        
        # SH Encoding for view direction (simplified to fixed order for implementation)
        # Standard SH encoding function would be called here.
        
    def get_spatial_features(self, x_coords, component_type='density'):
        """
        x_coords: tensor of shape (N, 3) containing (x, y, z) indices normalized to grid size
        Returns feature vector of shape (N, R)
        """
        if component_type == 'density':
            M_XY = self.density_M_XY
            M_XZ = self.density_M_XZ
            M_YZ = self.density_M_YZ
            v_X = self.density_v_X
            v_Y = self.density_v_Y
            v_Z = self.density_v_Z
            R = self.R_sigma
        else:
            M_XY = self.color_M_XY
            M_XZ = self.color_M_XZ
            M_YZ = self.color_M_YZ
            v_X = self.color_v_X
            v_Y = self.color_v_Y
            v_Z = self.color_v_Z
            R = self.R_color
            
        # Extract indices
        x_idx = x_coords[:, 0].long()
        y_idx = x_coords[:, 1].long()
        z_idx = x_coords[:, 2].long()
        
        # Gather vectors
        # v_X shape: (R, X) -> gather -> (N, R)
        v_X_gathered = v_X[:, x_idx].transpose(0, 1) # (N, R)
        v_Y_gathered = v_Y[:, y_idx].transpose(0, 1) # (N, R)
        v_Z_gathered = v_Z[:, z_idx].transpose(0, 1) # (N, R)
        
        # Gather matrices
        # M_XY shape: (R, X, Y) -> gather x -> (R, N, Y) -> gather y -> (R, N, N) -> transpose -> (N, N, R)
        # This is complex. Element-wise multiplication implies we sum over R.
        # Feature for point i is sum_r ( M_XY[r, x_i, y_i] * v_Z[r, z_i] + ... )
        # This results in a scalar per component type? 
        # The paper says "concatenate the outcomes... to construct the final feature".
        # If R is the depth, we need to preserve R.
        # Let's assume the "feature" is the vector of length R formed by the contributions of each rank.
        
        # Contribution from XY plane + Z vector
        # M_XY[r, x, y] * v_Z[r, z]
        M_XY_vals = M_XY[:, x_idx, y_idx] # (R, N)
        v_Z_vals = v_Z_gathered # (N, R)
        term_XY = M_XY_vals * v_Z_vals # (N, R)
        
        # Contribution from XZ plane + Y vector
        M_XZ_vals = M_XZ[:, x_idx, z_idx] # (R, N)
        v_Y_vals = v_Y_gathered # (N, R)
        term_XZ = M_XZ_vals * v_Y_vals # (N, R)
        
        # Contribution from YZ plane + X vector
        M_YZ_vals = M_YZ[:, y_idx, z_idx] # (R, N)
        v_X_vals = v_X_gathered # (N, R)
        term_YZ = M_YZ_vals * v_X_vals # (N, R)
        
        # Combine spatial features (sum over the three planes for each rank)
        spatial_feat = term_XY + term_XZ + term_YZ # (N, R)
        return spatial_feat

    def get_param_features(self, param_indices):
        """
        param_indices: list of tensors, one for each parameter dimension.
        Each tensor shape (N,) containing indices into the parameter grid.
        Returns feature vector of shape (N, R_param)
        """
        # T^K = sum_r (v^1_r o ... o v^K_r)
        # For a point with parameter indices p_1, p_2, ..., p_K
        # Feature[r] = v^1_r[p_1] * v^2_r[p_2] * ... * v^K_r[p_K]
        
        # Initialize with ones
        param_feat = torch.ones(param_indices[0].shape[0], self.R_param, device=param_indices[0].device)
        
        for k, idx in enumerate(param_indices):
            # v^k shape: (R, M_k)
            # Gather: (R, N) -> transpose to (N, R)
            v_k = self.param_vectors[k][:, idx].transpose(0, 1)
            param_feat = param_feat * v_k
            
        return param_feat

    def forward(self, x_coords, param_indices, view_dirs):
        """
        x_coords: (N, 3) spatial coordinates (indices)
        param_indices: list of (N,) tensors for each parameter
        view_dirs: (N, 3) viewing directions
        """
        # 1. Get Spatial Features
        f_sigma_spatial = self.get_spatial_features(x_coords, 'density')
        f_color_spatial = self.get_spatial_features(x_coords, 'color')
        
        # 2. Get Parameter Features
        f_param = self.get_param_features(param_indices)
        
        # 3. Combine Features
        # The paper implies concatenating spatial and param products.
        # However, the decoders take f_sigma and f_c.
        # Let's assume f_sigma = f_sigma_spatial + f_param (or concat)
        # Given the dimensions, if R_sigma != R_param, we might need projection.
        # The paper says "concatenate the outcomes... to construct the final feature".
        # Let's assume the final feature vector is the concatenation of spatial and param contributions.
        # But the decoders have fixed input size (hidden_dim).
        # We must project or assume R matches hidden_dim.
        # In the paper, hidden_dim is 128. R_sigma=16, R_color=48, R_param=4.
        # This suggests the "feature" passed to MLP is not just the raw grid output.
        # Re-reading: "decode the retrieved features...".
        # "concatenate the outcomes... to construct the final feature".
        # It is likely that the grid outputs are summed or projected.
        # For implementation, we will assume the grid outputs are summed to form a vector of size R_max,
        # then projected to hidden_dim. Or, more likely, the "feature" is the result of the element-wise ops
        # which results in a scalar per rank, and these scalars are the input.
        # Let's assume the input to the MLP is the concatenated vector of all rank contributions.
        
        # Simplified: Input to MLP is the spatial feature vector (size R) concatenated with param feature vector (size R_param).
        # We need to align dimensions. Let's assume a linear projection to hidden_dim.
        
        # Density Input
        # f_sigma_spatial (N, R_sigma) and f_param (N, R_param)
        # Concatenate
        f_sigma_input = torch.cat([f_sigma_spatial, f_param], dim=1)
        # Project to hidden_dim
        # Note: The paper doesn't explicitly state a projection layer, but MLP input dim must match.
        # We'll assume a linear layer is part of the decoder or implicit.
        # For this pseudocode, we pass the concatenated vector to the MLP.
        # If dimensions don't match, the MLP weights must be initialized accordingly.
        
        # Color Input
        f_color_input = torch.cat([f_color_spatial, f_param], dim=1)
        
        # 4. Encode Inputs
        # PE(x) and SH(d) are added to the features or concatenated.
        # Paper: sigma(x) = g_sigma(f_sigma(x), EP(x))
        # This implies concatenation or addition.
        # We will concatenate.
        
        # Placeholder for PE and SH encoding functions
        pe_x = positional_encoding(x_coords) # (N, D_pe)
        sh_d = spherical_harmonics_encoding(view_dirs) # (N, D_sh)
        
        # 5. Decode
        # Density
        # Input to g_sigma: f_sigma_input concatenated with PE(x)
        # We need to ensure f_sigma_input is projected to match MLP input size if necessary.
        # Assuming g_sigma handles the dimensionality.
        sigma = self.g_sigma(torch.cat([f_sigma_input, pe_x], dim=1))
        
        # Color
        # Input to g_c: f_color_input concatenated with SH(d)
        color = self.g_c(torch.cat([f_color_input, sh_d], dim=1))
        
        return sigma.squeeze(-1), color

    def compute_loss(self, sigma, color, gt_color, rays):
        # L_REC
        l_rec = F.mse_loss(color, gt_color)
        
        # L_L1
        # Apply to all parameters in the model (matrices, vectors)
        l1_loss = 0.0
        for name, param in self.named_parameters():
            if 'param_vectors' in name:
                # Parameter vectors: sum ||1 - w||
                l1_loss += torch.sum(torch.abs(1 - param))
            else:
                # Matrices/Vectors: sum ||w||_1 / |W|
                l1_loss += torch.sum(torch.abs(param)) / param.numel()
                
        # L_TV
        # Simplified TV loss calculation on spatial matrices
        # L_TV1 on vectors
        tv1 = 0.0
        for v in [self.density_v_X, self.density_v_Y, self.density_v_Z, 
                  self.color_v_X, self.color_v_Y, self.color_v_Z]:
            # Diff along first dim
            diff = v[:, 1:] - v[:, :-1]
            tv1 += torch.sum(torch.norm(diff, dim=0)) / (v.shape[0] * v.shape[1])
            
        # L_TV2 on matrices
        tv2 = 0.0
        for M in [self.density_M_XY, self.density_M_XZ, self.density_M_YZ,
                  self.color_M_XY, self.color_M_XZ, self.color_M_YZ]:
            # Diff along rows and cols
            diff_h = M[:, 1:, :] - M[:, :-1, :]
            diff_w = M[:, :, 1:] - M[:, :, :-1]
            tv2 += torch.sum(diff_h**2) / (M.shape[0] * M.shape[1] * M.shape[2])
            tv2 += torch.sum(diff_w**2) / (M.shape[0] * M.shape[1] * M.shape[2])
            
        l_tv = tv1 + tv2
        
        # Total Loss
        lambda1 = 1e-4 # Initial
        lambda2 = 1.0
        return l_rec + lambda1 * l1_loss + lambda2 * l_tv
```

**Parameters**
*   **Spatial Resolution ($N$):** Initial $128^3$. Can increase to $300^3$ for complex scenes [S1].
*   **Low-rank Components ($R$):** $R_\sigma = 16$ (density), $R_c = 48$ (color), $R_p = 4$ (parameter) [S1].
*   **MLP Hidden Dim:** 128 channels [S1].
*   **Batch Size:** 4096 [S1].
*   **Samples per Ray:** 192 (initial), 512 (if grid is $300^3$) [S1].
*   **Optimizer:** Adam ($\beta_1=0.9, \beta_2=0.99$) [S1].
*   **Learning Rate:** 0.02 for matrices/vectors, 0.001 for MLPs [S1].
*   **L1 Weight ($\lambda_1$):** $10^{-4}$ initially, $10^{-5}$ after 2,000 iterations [S1].
*   **TV Weight ($\lambda_2$):** 1.0 [S1].
*   **Training Iterations:** 30,000 for static, 90,000 for dynamic [S1].

**Edge Cases and Pitfalls**
*   **Noise:** Without L1 and TV regularization, excessive noise and film grains appear in spatial matrices and images [S1].
*   **Memory:** High-resolution datasets (e.g., 1024x1024) require significant RAM (~29 GB for ray storage alone). ViSNeRF filters rays not intersecting the bounding box to reduce runtime RAM [S1].
*   **Hardware:** Minimum 32 GB RAM and 10 GB VRAM recommended; 64 GB RAM and 16 GB VRAM ideal [S1].
*   **Bounding Box:** Empty voxels must be handled. A mask volume is created at iteration 2,000 to shrink the bounding box and enable voxel skipping [S1].

## Operating characteristics
*   **Time Complexity:** Factorization reduces spatial complexity from $O(N^3)$ to $O(N^2)$ and parameter complexity from $O(M^K)$ to $O(M)$, resulting in overall $O(N^2 M)$ [S1].
*   **Memory Complexity:** Reduced from $O(N^3 M^K)$ to $O(N^2 M)$ [S1].
*   **Performance:** ViSNeRF is the fastest among compared methods (InSituNet, CoordNet, StyleGAN2, EG3D) for training time [S1]. It is the slowest for inference time, with frame rates dropping substantially for 1024x1024 images (at least 1.86 FPS for 256x256) [S1].
*   **Strengths:** High visualization quality (PSNR/SSIM) with significantly reduced training data (95% less than InSituNet) and smaller model size than most baselines [S1]. Supports super-resolution synthesis [S1].
*   **Limitations:** High VRAM/RAM requirements compared to traditional DVR/IR. Slow inference speed for high resolutions [S1].

## Background
ViSNeRF was introduced in the paper "ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes" [S1]. It addresses the need for efficient visualization of dynamic volumetric scenes (e.g., scientific simulations) by extending the NeRF framework with tensor factorization techniques (TensoRF, K-Planes) to handle arbitrary parameters [S1].

## Sources
[S1] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
