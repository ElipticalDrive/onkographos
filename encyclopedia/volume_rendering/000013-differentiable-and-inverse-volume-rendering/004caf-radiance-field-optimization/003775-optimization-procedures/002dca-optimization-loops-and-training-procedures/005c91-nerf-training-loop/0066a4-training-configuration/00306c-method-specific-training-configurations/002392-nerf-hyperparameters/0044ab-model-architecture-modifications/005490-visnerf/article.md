# ViSNeRF

Also called: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes

## Overview
ViSNeRF is a hybrid neural radiance field method designed for the efficient visualization synthesis of dynamic volumetric scenes with arbitrary adjustable parameters (e.g., time, isovalue, transfer functions) [S1]. It combines explicit feature grids with implicit feature decoders to achieve high-quality, 3D-consistent renderings while minimizing model size and GPU memory cost [S1]. Unlike standard NeRFs that treat scenes as static 3D volumes, ViSNeRF factorizes the representation into a 3D spatial tensor and a K-dimensional parameter tensor, allowing it to interpolate across parameter spaces without retraining [S1]. It distinguishes itself from fully explicit methods (like PlenOctrees) and fully implicit methods (like CoordNet) by using vector-matrix decomposition for spatial features and small MLPs for decoding, offering a balance between rendering speed, storage efficiency, and visual fidelity [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of training images (DVR or IR) with known camera poses and associated scene parameters (e.g., timestep, isovalue) [S1].
*   Ray origins and directions cast from screen pixels toward the scene [S1].
*   Sampling positions $(x, y, z)$ along the rays [S1].
*   Scene parameters $(p_0, p_1, \dots)$ defining the specific frame or state to visualize [S1].

**Outputs:**
*   Predicted color $\hat{C}(r)$ and density $\sigma$ for each ray $r$ [S1].
*   Synthesized visualization images for novel viewpoints and interpolated parameter values [S1].

### Data Structures and Layout

**1. Spatial Feature Grid ($T^3$):**
*   **Layout:** A 3D tensor of size $N \times N \times N$ (or non-cubic $X \times Y \times Z$ based on bounding box) [S1].
*   **Factorization:** Decomposed into vector-matrix pairs using block-term tensor decomposition [S1].
    *   Density features: Depth $R_\sigma = 16$.
    *   Appearance (color) features: Depth $R_c = 48$.
    *   Total low-rank components $R_s = 64$ [S1].
*   **Components:**
    *   Three spatial matrices: $M^X_{XY,r}$, $M^Y_{XZ,r}$, $M^Z_{YZ,r}$ representing XY, XZ, and YZ planes [S1].
    *   Three spatial vectors: $v^Z_r$, $v^Y_r$, $v^X_r$ along the X, Y, Z axes [S1].
*   **Initialization:** Starts at $N=128$. For complex scenes (e.g., Tangaroa), resolution increases to $300^3$ at iterations 2,000, 4,000, 6,000, and 8,000 [S1].

**2. Parameter Feature Grid ($T^K$):**
*   **Layout:** A K-dimensional tensor $M^K$, where $K$ is the number of adjustable parameters [S1].
*   **Dimensions:** $M^k$ corresponds to the number of distinct values for parameter $k$ in the training set [S1].
    *   Example: 5 jets (timestep) $K=1, M^1=11$; Tangaroa (isovalue) $K=1, M^1=13$; Vortex (TF-1) $K=2, M^2=36 (6 \times 6)$; Nyx (simulation params) $K=3, M^3=45 (3 \times 3 \times 5)$ [S1].
*   **Factorization:** Decomposed using CANDECOMP/PARAFAC (CP) decomposition into $R_p$ one-rank components [S1].
    *   Components: $v^1_r, v^2_r, \dots, v^K_r$ [S1].
    *   Number of components $R_p = 4$ [S1].

**3. Feature Decoders (MLPs):**
*   Two small MLPs: $g_\sigma$ for density and $g_c$ for color [S1].
*   **Architecture:** One hidden layer, 128 channels per layer [S1].
*   **Inputs:**
    *   $g_\sigma$: Density features $f_\sigma$ and positional encoding $E_P(x)$ [S1].
    *   $g_c$: Color features $f_c$ and spherical harmonic encoding $E_{SH}(d)$ of viewing direction $d$ [S1].

**4. Ray Data Structure:**
*   Each ray stores: Origin (3 floats), Direction (3 floats), RGB values (3 floats), and Parameters (1 float per parameter) [S1].
*   Total: ~10 floats per pixel [S1].

### Numbered Steps

1.  **Ray Casting:** Cast rays from screen pixels toward the scene from camera locations [S1].
2.  **Stratified Sampling:** Estimate the volume rendering integral using $n$ samples along each ray [S1].
    *   Step size $\delta_i = t_{i+1} - t_i$ [S1].
    *   Transmittance $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1].
    *   Predicted color $\hat{C}(r) = \sum_{i=1}^{n} T_i (1 - \exp(-\sigma_i \delta_i)) c_i$ [S1].
3.  **Feature Retrieval:**
    *   For each sample point $(x,y,z)$ and parameters $(p_0, \dots)$, retrieve features from the spatial and parameter grids [S1].
    *   Spatial features are combined via element-wise multiplication of vector-matrix pairs:
        $$ V = \sum_{r=1}^{R_s} \left( M^X_{XY,r} \circ v^Z_r + M^Y_{XZ,r} \circ v^Y_r + M^Z_{YZ,r} \circ v^X_r \right) $$
        [S1]
    *   Parameter features are combined via element-wise multiplication of parameter vectors:
        $$ P = \sum_{r=1}^{R_p} v^1_r \circ v^2_r \circ \dots \circ v^K_r $$
        [S1]
    *   Concatenate the outcomes of these multiplications to form the final feature vector [S1].
4.  **Decoding:**
    *   Compute density: $\sigma(x) = g_\sigma(f_\sigma(x), E_P(x))$ [S1].
    *   Compute color: $c(x) = g_c(f_c(x), E_{SH}(d))$ [S1].
5.  **Rendering:** Compose the visualization by collecting sampled points and applying the volume rendering equation [S1].
6.  **Optimization:** Minimize the loss function $L = L_{REC} + \lambda_1 L_{L1} + \lambda_2 L_{TV}$ [S1].
    *   $L_{REC}$: MSE between predicted and ground truth color [S1].
    *   $L_{L1}$: L1 norm regularization on grid weights (initialized to 1 for parameter vectors) [S1].
    *   $L_{TV}$: Total Variation loss on vectors and matrices to reduce noise [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ViSNeRF:
    def __init__(self, spatial_res, param_dims, R_sigma=16, R_color=48, R_param=4):
        """
        Initialize ViSNeRF model.
        spatial_res: int, N for N x N x N grid (or tuple for non-cubic)
        param_dims: list of ints, M^k for each parameter dimension
        R_sigma: int, low-rank components for density
        R_color: int, low-rank components for color
        R_param: int, low-rank components for parameter space
        """
        super().__init__()
        
        # Spatial Feature Grids (Block-Term Decomposition)
        # Density: R_sigma components
        self.M_XY_sigma = nn.Parameter(torch.randn(R_sigma, spatial_res[0], spatial_res[1]))
        self.M_XZ_sigma = nn.Parameter(torch.randn(R_sigma, spatial_res[0], spatial_res[2]))
        self.M_YZ_sigma = nn.Parameter(torch.randn(R_sigma, spatial_res[1], spatial_res[2]))
        self.v_Z_sigma = nn.Parameter(torch.randn(R_sigma, spatial_res[2]))
        self.v_Y_sigma = nn.Parameter(torch.randn(R_sigma, spatial_res[1]))
        self.v_X_sigma = nn.Parameter(torch.randn(R_sigma, spatial_res[0]))
        
        # Color: R_color components
        self.M_XY_color = nn.Parameter(torch.randn(R_color, spatial_res[0], spatial_res[1]))
        self.M_XZ_color = nn.Parameter(torch.randn(R_color, spatial_res[0], spatial_res[2]))
        self.M_YZ_color = nn.Parameter(torch.randn(R_color, spatial_res[1], spatial_res[2]))
        self.v_Z_color = nn.Parameter(torch.randn(R_color, spatial_res[2]))
        self.v_Y_color = nn.Parameter(torch.randn(R_color, spatial_res[1]))
        self.v_X_color = nn.Parameter(torch.randn(R_color, spatial_res[0]))
        
        # Parameter Feature Grid (CP Decomposition)
        # K dimensions, each with M^k entries
        self.param_vectors = []
        for k_dim in param_dims:
            # R_param vectors of length k_dim
            vec = nn.Parameter(torch.randn(R_param, k_dim))
            self.param_vectors.append(vec)
            
        # MLP Decoders
        # Density MLP: input dim = R_sigma, output dim = 1 (sigma)
        self.g_sigma = nn.Sequential(
            nn.Linear(R_sigma, 128),
            nn.ReLU(),
            nn.Linear(128, 1)
        )
        
        # Color MLP: input dim = R_color, output dim = 3 (RGB)
        # Note: SH encoding is applied to direction d before input
        self.g_color = nn.Sequential(
            nn.Linear(R_color, 128),
            nn.ReLU(),
            nn.Linear(128, 3)
        )
        
        # Hyperparameters
        self.R_sigma = R_sigma
        self.R_color = R_color
        self.R_param = R_param
        self.param_dims = param_dims
        self.spatial_res = spatial_res

    def get_spatial_features(self, x, feature_type='sigma'):
        """
        Retrieve spatial features for position x using block-term decomposition.
        x: Tensor of shape (..., 3) containing (x, y, z)
        """
        if feature_type == 'sigma':
            M_XY = self.M_XY_sigma
            M_XZ = self.M_XZ_sigma
            M_YZ = self.M_YZ_sigma
            v_X = self.v_X_sigma
            v_Y = self.v_Y_sigma
            v_Z = self.v_Z_sigma
            R = self.R_sigma
        else:
            M_XY = self.M_XY_color
            M_XZ = self.M_XZ_color
            M_YZ = self.M_YZ_color
            v_X = self.v_X_color
            v_Y = self.v_Y_color
            v_Z = self.v_Z_color
            R = self.R_color
            
        # Extract coordinates
        x_coord = x[..., 0]
        y_coord = x[..., 1]
        z_coord = x[..., 2]
        
        # Interpolate features from matrices and vectors
        # Note: In implementation, interpolation functions (e.g., grid_sample or trilinear)
        # would be used. Here we assume a helper 'interpolate' that handles the indexing.
        
        # M_XY: shape (R, X, Y) -> sample at (x, y)
        feat_XY = interpolate(M_XY, x_coord, y_coord) # Shape (..., R)
        
        # M_XZ: shape (R, X, Z) -> sample at (x, z)
        feat_XZ = interpolate(M_XZ, x_coord, z_coord) # Shape (..., R)
        
        # M_YZ: shape (R, Y, Z) -> sample at (y, z)
        feat_YZ = interpolate(M_YZ, y_coord, z_coord) # Shape (..., R)
        
        # v_X: shape (R, X) -> sample at x
        feat_X = interpolate(v_X, x_coord) # Shape (..., R)
        
        # v_Y: shape (R, Y) -> sample at y
        feat_Y = interpolate(v_Y, y_coord) # Shape (..., R)
        
        # v_Z: shape (R, Z) -> sample at z
        feat_Z = interpolate(v_Z, z_coord) # Shape (..., R)
        
        # Element-wise multiplication and summation
        # V = sum_r ( M_XY_r * v_Z_r + M_XZ_r * v_Y_r + M_YZ_r * v_X_r )
        # feat_XY is (..., R), feat_Z is (..., R) -> element-wise mult
        term1 = feat_XY * feat_Z
        term2 = feat_XZ * feat_Y
        term3 = feat_YZ * feat_X
        
        features = term1 + term2 + term3
        return features

    def get_param_features(self, params):
        """
        Retrieve parameter features using CP decomposition.
        params: Tensor of shape (..., K) containing parameter values
        """
        K = len(self.param_dims)
        # Initialize feature accumulation
        # We need to sum over R_p components
        # Each component is the outer product of K vectors
        
        # Reshape params to (..., K, 1) for broadcasting
        params_expanded = params.unsqueeze(-1) # (..., K, 1)
        
        # For each parameter dimension k, interpolate the corresponding vector
        param_feats = []
        for k in range(K):
            # self.param_vectors[k] shape (R_param, M^k)
            # params_expanded[..., k, :] shape (..., 1)
            # Interpolate along the M^k dimension
            feat_k = interpolate(self.param_vectors[k], params_expanded[..., k, :]) # Shape (..., R_param)
            param_feats.append(feat_k)
            
        # Outer product of K vectors for each rank r
        # feat_0 * feat_1 * ... * feat_{K-1}
        # Since all are (..., R_param), we can multiply them element-wise
        combined = param_feats[0]
        for k in range(1, K):
            combined = combined * param_feats[k]
            
        # Sum over R_p components
        features = torch.sum(combined, dim=-1) # Shape (..., )
        return features

    def forward(self, rays, params):
        """
        Forward pass for a batch of rays.
        rays: Tensor of shape (B, 3) -> (origin_x, origin_y, origin_z) for sampled points
        params: Tensor of shape (B, K) -> scene parameters for the frame
        """
        # 1. Get Spatial Features
        f_sigma = self.get_spatial_features(rays, 'sigma')
        f_color = self.get_spatial_features(rays, 'color')
        
        # 2. Get Parameter Features
        f_param = self.get_param_features(params)
        
        # 3. Combine Features
        # Concatenate spatial and parameter features? 
        # The source says: "concatenate the outcomes of these element-wise multiplications 
        # to construct the final feature, integrating products from three spatial pairs 
        # and the parameter vector product."
        # However, the decoders take f_sigma and f_color separately.
        # The parameter features likely modulate or are concatenated.
        # Based on "decode density and color features separately", and "combine features... 
        # to construct the final feature", it implies the parameter features are combined 
        # with spatial features before decoding or the spatial features are already 
        # conditioned on parameters via the grid lookup.
        # Given the structure: f_sigma is from spatial grid. f_param is from param grid.
        # The source states: "ViSNeRF then retrieves features from spatial matrices and 
        # vectors as well as parameter vectors, decoding them into color and density values."
        # And: "concatenate the outcomes... to construct the final feature".
        # Let's assume the final feature for decoding is the concatenation of spatial 
        # and parameter features, or that the spatial features are already the result 
        # of the interaction. 
        # Re-reading: "The combination of features sampled from spatial matrices and 
        # vectors is achieved through element-wise multiplications... This element-wise 
        # multiplication is also applied for combining features sampled from the 
        # parameter vectors. We concatenate the outcomes... to construct the final feature."
        # This suggests: Final_Feat = Concat( Spatial_Feat, Param_Feat )?
        # But decoders are separate. 
        # Let's look at Eq 7: sigma(x) = g_sigma(f_sigma(x), E_P(x)). 
        # It does not explicitly show param features in the decoder input in Eq 7.
        # However, the factorization T^{3+K} = T^3 o T^K implies the features are 
        # derived from the combined tensor.
        # In practice, for hybrid models, param features often modulate spatial features.
        # Given the ambiguity in the exact fusion mechanism in the text, 
        # we follow the explicit decoder inputs: f_sigma and f_color are spatial.
        # The parameter features must be integrated. 
        # A common approach in such factorized NeRFs is to use param features to 
        # select or weight spatial features. 
        # However, the text says "concatenate... to construct the final feature".
        # Let's assume the final feature passed to the MLP is the concatenation 
        # of the spatial feature vector and the parameter feature vector.
        
        # Expand f_sigma and f_color to match batch size if needed
        # f_sigma shape: (B, R_sigma)
        # f_param shape: (B, R_param) ? No, CP decomposition output is scalar per rank?
        # Actually, CP decomposition of T^K results in a vector of size R_p? 
        # No, T^K is a tensor. The decomposition is sum of outer products.
        # The result of the CP decomposition for a specific parameter point is a 
        # scalar value for each rank? Or a vector?
        # "T^K = sum v1 o v2 ... o vK". This results in a scalar field over the K-dim space.
        # So f_param is a scalar (or vector of size R_p if we keep ranks separate).
        # Let's assume f_param has shape (B, R_p).
        
        # Concatenate
        # Note: R_sigma and R_p might differ.
        # If they differ, we might need to align them or just concatenate.
        # The text doesn't specify alignment. We concatenate.
        
        # However, g_sigma takes f_sigma. g_color takes f_color.
        # If we concatenate, we change input dim.
        # Let's assume the "final feature" refers to the feature vector used for 
        # the specific prediction. 
        # Given "decode density and color features separately", and Eq 7 uses f_sigma(x),
        # it's possible the parameter features are used to index or modulate the spatial 
        # features *before* they are passed to the decoder, or the decoder takes both.
        # Let's assume the decoder takes the concatenated feature for robustness, 
        # or that f_sigma already incorporates parameter info via the grid structure.
        # Actually, the grid is 3D. The parameters are separate.
        # The most logical interpretation of "concatenate... to construct the final feature" 
        # in the context of separate decoders is that the feature vector passed to 
        # g_sigma is [f_sigma, f_param] and to g_color is [f_color, f_param].
        
        final_feat_sigma = torch.cat([f_sigma, f_param], dim=-1)
        final_feat_color = torch.cat([f_color, f_param], dim=-1)
        
        # 4. Encode Inputs
        # Positional Encoding for x
        x_pe = positional_encoding(rays) # E_P(x)
        
        # Spherical Harmonics for direction d
        # Note: rays tensor here is just position. We need direction.
        # Assuming 'rays' input in forward includes direction or we pass it separately.
        # For this pseudocode, we assume 'directions' is passed or derived.
        # Let's assume 'rays' contains (origin, direction) or we have a 'directions' arg.
        # For simplicity, let's assume 'directions' is available.
        d_sh = spherical_harmonic_encoding(directions) # E_SH(d)
        
        # 5. Decode
        sigma = self.g_sigma(torch.cat([final_feat_sigma, x_pe], dim=-1))
        color = self.g_color(torch.cat([final_feat_color, d_sh], dim=-1))
        
        return sigma, color

    def compute_loss(self, sigma_pred, color_pred, sigma_gt, color_gt, params):
        """
        Compute total loss.
        """
        # Reconstruction Loss (MSE)
        L_REC = F.mse_loss(color_pred, color_gt)
        
        # L1 Loss on Grid Weights
        L_L1 = 0.0
        # Apply to spatial matrices and vectors
        for param in list(self.parameters()):
            if param in [self.M_XY_sigma, self.M_XZ_sigma, self.M_YZ_sigma, 
                         self.v_X_sigma, self.v_Y_sigma, self.v_Z_sigma,
                         self.M_XY_color, self.M_XZ_color, self.M_YZ_color,
                         self.v_X_color, self.v_Y_color, self.v_Z_color]:
                L_L1 += torch.mean(torch.abs(param))
            elif param in self.param_vectors:
                # For parameter vectors, regularize to 1
                # L1 = sum ||1 - w||
                L_L1 += torch.mean(torch.abs(1.0 - param))
                
        # TV Loss (Simplified)
        # L_TV1 on vectors, L_TV2 on matrices
        L_TV = 0.0
        # Implementation of TV loss requires gradient computation or finite differences
        # This is a placeholder for the complex TV calculation described in [S1]
        L_TV += compute_tv_loss(self.v_X_sigma, self.v_Y_sigma, self.v_Z_sigma)
        L_TV += compute_tv_loss(self.M_XY_sigma, self.M_XZ_sigma, self.M_YZ_sigma)
        # ... repeat for color features
        
        # Weights
        lambda_1 = 1e-4 # Initially
        lambda_2 = 1.0
        
        L = L_REC + lambda_1 * L_L1 + lambda_2 * L_TV
        return L
```

### Parameters and Typical Values
*   **Spatial Grid Resolution ($N$):** 128 (initial), up to $300^3$ for complex scenes [S1].
*   **Low-Rank Components ($R_\sigma, R_c, R_p$):** $R_\sigma=16, R_c=48, R_p=4$ [S1].
*   **MLP Hidden Layers:** 1 layer, 128 channels [S1].
*   **Batch Size:** 4096 [S1].
*   **Samples per Ray:** 192 (initial), 512 (if grid is $300^3$) [S1].
*   **Optimizer:** Adam ($\beta_1=0.9, \beta_2=0.99$) [S1].
*   **Learning Rates:** 0.02 for grids, 0.001 for MLPs [S1].
*   **Loss Weights:** $\lambda_1 = 10^{-4}$ (switches to $10^{-5}$ after 2,000 iters), $\lambda_2 = 1.0$ [S1].
*   **Training Iterations:** 30,000 for static, 90,000 for dynamic [S1].

### Edge Cases and Pitfalls
*   **Noise:** Without L1 and TV regularization, excessive noise and film grains appear in spatial matrices and images [S1].
*   **Memory:** High-resolution datasets (e.g., 1024x1024) require significant RAM (~29 GB for raw ray data) and VRAM. Filtering rays outside the bounding box is necessary [S1].
*   **Hardware:** Minimum 32 GB RAM / 10 GB VRAM; 64 GB RAM / 16 GB VRAM recommended [S1].
*   **Convergence:** Static scenes converge at 30,000 iterations; dynamic scenes require triple this [S1].

## Operating characteristics

**Time and Memory Complexity:**
*   Spatial complexity reduced from $O(N^3)$ to $O(N^2)$ [S1].
*   Parameter space complexity reduced from $O(M^K)$ to $O(M)$ [S1].
*   Overall complexity reduced from $O(N^3 M^K)$ to $O(N^2 M)$ [S1].

**Measured Performance:**
*   **Inference Speed:** At least 1.86 FPS for 256x256 images; drops substantially for 1024x256 [S1]. Slower than InSituNet/StyleGAN2 but faster than CoordNet/EG3D for low-res [S1].
*   **Training Time:** Fastest among compared methods (e.g., 1.43 hours for five jets) [S1].
*   **Model Size:** Compact (e.g., 15.43 MB for five jets, 69.91 MB for Tangaroa) [S1].

**Strengths and Limitations:**
*   **Strengths:** High quality (PSNR 27.51 on five jets vs 22.74 for EG3D), supports parameter interpolation, reduces training data by 95% compared to InSituNet, super-resolution capability [S1].
*   **Limitations:** Slow inference for high resolutions, high initial memory requirement for loading ray data, requires GPU training [S1].

## Background

**Origin:**
ViSNeRF was introduced in the paper "ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes" [S1].

**History and Development:**
It builds on the NeRF framework [S1] and TensoRF [S1], adapting tensor factorization techniques (block-term and CP decomposition) to handle dynamic scenes with arbitrary parameters. It addresses the limitations of fully explicit methods (limited expressiveness) and fully implicit methods (slow training/inference) by using a hybrid approach [S1].

**Usage:**
Used for visualization synthesis of dynamic volumetric scenes, including super-resolution synthesis and parameter-space exploration (time, isovalue, transfer functions) [S1].

## Sources

[S1] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
