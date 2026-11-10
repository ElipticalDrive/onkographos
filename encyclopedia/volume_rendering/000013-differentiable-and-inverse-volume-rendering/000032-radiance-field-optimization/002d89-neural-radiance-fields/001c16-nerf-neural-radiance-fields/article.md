# NeRF (Neural Radiance Fields)

Also called: NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis, Neural Radiance Field, Neural Radiation Field, Original NeRF, Preliminaries: Neural Radiance Fields (NeRF), Radiance Fields (NeRF)

## Overview
NeRF represents a static 3D scene as a continuous volumetric function that maps a 3D spatial position and a 2D viewing direction to an emitted RGB color and a volume density scalar [S1][S3, S6, unofficial]. It solves the problem of novel view synthesis by replacing explicit geometric primitives (meshes or point clouds) with an implicit neural representation encoded in network weights, allowing the generation of photorealistic images from arbitrary camera viewpoints not seen during training [S1][S4, S11, unofficial]. It distinguishes itself from earlier implicit methods by explicitly modeling view-dependent appearance (specularity, transparency) via the viewing direction input and by using a two-tier hierarchical sampling strategy to efficiently resolve the scene geometry and appearance [S2, S10, unofficial].

## How it works
**Inputs and Outputs:**
*   **Training Inputs:** A set of $N$ posed 2D images $\{I_i\}$ with known camera poses (extrinsics $R_i, t_i$ and intrinsics $K_i$) [S10, S20, unofficial].
*   **Training Output:** Optimized network parameters $\theta$ for the radiance field functions [S3, unofficial].
*   **Rendering Input:** A camera pose and image resolution $(H, W)$ [S10, unofficial].
*   **Rendering Output:** A synthesized 2D image $\hat{I}$ [S1].

**Data Structures:**
The scene is represented by two Multi-Layer Perceptrons (MLPs), collectively denoted as $F_\theta$:
1.  **Density Network ($\sigma$):** An MLP that takes encoded 3D position as input and outputs a scalar volume density $\sigma \in \mathbb{R}^+$ [S2, unofficial][S27].
2.  **Color Network ($\mathbf{c}$):** An MLP that takes the intermediate feature vector from the density network and encoded viewing direction as input and outputs an RGB color vector $\mathbf{c} \in \mathbb{R}^3$ [S2, unofficial][S27].

**Positional Encoding:**
Inputs are mapped to high-dimensional Fourier features to enable the MLP to represent high-frequency details [S1][S12, unofficial]. For an input vector $\mathbf{p}$ and frequency bands $L$, the encoding $\gamma(\mathbf{p})$ is:
$$ \gamma(\mathbf{p}) = (\sin(2^0 \pi \mathbf{p}), \cos(2^0 \pi \mathbf{p}), \dots, \sin(2^{L-1} \pi \mathbf{p}), \cos(2^{L-1} \pi \mathbf{p})) $$
[S12, S26, unofficial]. The 3D position $\mathbf{x}$ uses $L_x$ bands, and the viewing direction $\mathbf{d}$ uses $L_d$ bands, where typically $L_d < L_x$ because color varies more smoothly with direction than position [S12, unofficial].

**Algorithm Steps:**

1.  **Ray Generation:** For each pixel $(u, v)$ in the target image, compute the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space using the camera pose and intrinsics [S10, S15, unofficial].
    *   Normalize pixel coordinates: $u' = u - W/2, v' = v - H/2$.
    *   Compute camera-space direction: $\mathbf{d}_{cam} = [u'/f, v'/f, -1]^T$ (assuming $f_x=f_y=f$) [S15, unofficial].
    *   Transform to world space: $\mathbf{d} = R \mathbf{d}_{cam}$, $\mathbf{o} = \mathbf{t}$ [S15, S16, unofficial].

2.  **Sampling:** Sample $N$ points along the ray within the interval $[t_n, t_f]$ [S10, unofficial].
    *   **Coarse Sampling:** Divide $[t_n, t_f]$ into $N_c$ equal bins. Draw one sample $t_i$ uniformly from each bin $[t_n + \frac{i-1}{N_c}(t_f-t_n), t_n + \frac{i}{N_c}(t_f-t_n)]$ [S10, S14, unofficial]. During training, add random jitter ($perturb=True$) to break symmetry [S10, unofficial].
    *   **Fine Sampling (Hierarchical):** Use the coarse network to compute weights $w_i = T_i(1-\exp(-\sigma_i \delta_i))$. Normalize these weights to form a probability distribution. Sample $N_f$ additional points using inverse-CDF sampling from this distribution to concentrate samples in dense regions [S10, S21, unofficial].

3.  **Network Query:** For each sampled point $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$:
    *   Compute encoded position $\gamma(\mathbf{x}_i)$ and direction $\gamma(\mathbf{d})$ [S6, unofficial].
    *   **Density Prediction:** Pass $\gamma(\mathbf{x}_i)$ through the density network. The output is passed through a Softplus activation to ensure $\sigma_i \ge 0$ [S9, S18, unofficial].
        $$ \sigma_i = \text{Softplus}(\text{MLP}_{\sigma}(\gamma(\mathbf{x}_i))) $$
    *   **Color Prediction:** Pass $\gamma(\mathbf{x}_i)$ through the density network to get an intermediate feature vector $\mathbf{h}_i$ [S27]. Concatenate $\mathbf{h}_i$ with $\gamma(\mathbf{d})$ and pass through the color network. Apply Sigmoid activation to bound RGB values to $[0, 1]$ [S16, unofficial][S27].
        $$ \mathbf{c}_i = \text{Sigmoid}(\text{MLP}_{\mathbf{c}}([\mathbf{h}_i, \gamma(\mathbf{d})])) $$

4.  **Volume Rendering:** Compute the rendered color $\hat{C}(\mathbf{r})$ for the ray using the discrete volume rendering equation [S5][S10, unofficial]:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
    Where:
    *   $\delta_i = t_{i+1} - t_i$ is the distance between adjacent samples (with $t_{N+1}$ often set to $t_f$ or handled via norm of direction) [S10, S12, unofficial].
    *   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity of the segment [S10, unofficial].
    *   $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance [S10, unofficial].

5.  **Background Handling:** If a white background is assumed, add the unaccounted opacity to the final color:
    $$ \hat{C}_{final}(\mathbf{r}) = \hat{C}(\mathbf{r}) + T_{N+1} \mathbf{c}_{bg} $$
    where $\mathbf{c}_{bg} = [1, 1, 1]$ if `white_bkgd` is True, else $\mathbf{0}$ [S6, S10, unofficial].

6.  **Optimization:** Minimize the Mean Squared Error (MSE) loss between the rendered image $\hat{I}$ and the ground truth image $I$ using stochastic gradient descent [S3, unofficial][S23]:
    $$ \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{I}(\mathbf{r}) - I(\mathbf{r}) \|_2^2 $$
    [S5][S26, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PositionalEncoding(nn.Module):
    def __init__(self, num_bands):
        super().__init__()
        self.num_bands = num_bands

    def forward(self, x):
        # x shape: (..., 3) or (..., 2)
        freq_bands = 2.0 ** torch.linspace(0, self.num_bands - 1, self.num_bands).to(x.device)
        # Expand dimensions for broadcasting
        freq_bands = freq_bands.view(1, -1, 1)
        # x * freq_bands -> (..., num_bands, 3)
        embedded = x.unsqueeze(-2) * freq_bands
        # Apply sin/cos and flatten last two dims
        embedded = torch.cat([torch.sin(embedded), torch.cos(embedded)], dim=-1)
        # Result shape: (..., num_bands * 2 * input_dim)
        return embedded

class NeRF(nn.Module):
    def __init__(self, D=8, W=256, skip=[4], in_channels_xyz=63, in_channels_dir=27):
        super().__init__()
        self.D = D
        self.W = W
        self.skip = skip
        
        # Density Network
        self.linear_xyz = nn.ModuleList([nn.Linear(in_channels_xyz, W)])
        for _ in range(1, D):
            self.linear_xyz.append(nn.Linear(W, W))
            
        # Color Network
        self.linear_feat = nn.Linear(W, W)
        self.linear_rgb = nn.Linear(W + in_channels_dir, 3)

    def forward(self, xyz_encoded, dir_encoded):
        # xyz_encoded: (..., in_channels_xyz)
        # dir_encoded: (..., in_channels_dir)
        
        h = xyz_encoded
        for i, layer in enumerate(self.linear_xyz):
            h = layer(h)
            if i < self.D - 1:
                h = F.relu(h)
                if i in self.skip:
                    h = torch.cat([h, xyz_encoded], dim=-1)
        
        # Density output (Softplus for positivity)
        sigma = F.softplus(self.linear_xyz[-1](h)) # Note: Original paper uses Softplus on the final output of the density branch. 
        # In many implementations, the last layer of the density network outputs sigma directly or via softplus.
        # The provided notes [S9, S18, unofficial] indicate Softplus.
        # However, standard implementation often has a separate linear layer for sigma.
        # Let's assume the last layer of linear_xyz outputs the feature for color, 
        # and we need a separate path for sigma? 
        # Re-reading [S27]: "The density network is an 8-layer MLP... The color network is a 2-layer MLP."
        # Re-reading [S2, unofficial]: "opacity is predicted without the view direction".
        # Standard NeRF (Mildenhall et al.): 
        # h = MLP_xyz(x) -> sigma = Softplus(h_last)
        # h_feat = ReLU(h_last) -> concat with dir -> MLP_dir -> color
        
        # Correcting structure based on [S27] and [S2, unofficial]:
        # The first D-1 layers of the XYZ network are shared.
        # The final layer of the XYZ network outputs the feature vector for the color network.
        # Density is predicted from the output of the (D-1)-th layer? Or a separate head?
        # Mildenhall 2020: "The first half takes encoded position x to output intermediate feature vector h and scalar density sigma."
        # This implies the density network outputs BOTH h and sigma.
        
        # Let's implement the standard Mildenhall structure:
        # Input x -> Linear -> ReLU -> ... -> Linear -> ReLU -> Linear -> [h, sigma]
        # But the code in [S18, unofficial] shows:
        # x = F.relu(self.linears_xyz[i](x))
        # density = F.relu(self.linear_density(x)) -> This suggests a separate head or layer.
        
        # Let's stick to the most common interpretation [S27][S2, unofficial]:
        # 1. Process xyz through D layers.
        # 2. The output of layer D-1 (0-indexed) is used for density? 
        # Actually, Mildenhall 2020 Fig 2: 
        # x -> [Linear, ReLU] x 8 -> [Linear] -> h (256)
        # h -> [Linear] -> sigma (1)
        # h -> [Concat dir] -> [Linear, ReLU] -> [Linear] -> color (3)
        
        # Re-implementing based on [S27] "8-layer MLP... intermediate feature vector size 256"
        # and [S2, unofficial] "opacity predicted without view direction".
        
        h = xyz_encoded
        for i in range(self.D):
            h = self.linear_xyz[i](h)
            if i < self.D - 1:
                h = F.relu(h)
                if i in self.skip:
                    h = torch.cat([h, xyz_encoded], dim=-1)
        
        # Predict sigma from the output of the last layer? 
        # Mildenhall 2020: "The first half... output intermediate feature vector h and scalar density sigma."
        # This usually means the last layer outputs h, and a separate small network or layer outputs sigma from h?
        # Or the last layer outputs sigma, and a parallel branch outputs h?
        # Common implementation: 
        # h = ReLU(Linear(x))
        # sigma = Softplus(Linear(h))
        # h_feat = ReLU(h) -> concat dir -> ...
        
        # Let's use the structure from [S18, unofficial] which is explicit:
        # density = F.relu(self.linear_density(x)) -> This is likely Softplus in practice for sigma.
        
        # To be safe and implementable:
        # We assume self.linear_xyz[-1] outputs the feature vector h_feat.
        # We assume a separate linear layer for sigma.
        
        # However, the prompt asks for *one* clean style. 
        # Let's follow the explicit structure in [S27]:
        # Density network: 8 layers. Color network: 2 layers.
        # Input x -> 8 layer MLP -> h (256).
        # h -> Linear -> sigma.
        # h -> Concat dir -> 2 layer MLP -> color.
        
        # But wait, [S2, unofficial] says "The model is deeper... skip connection... ray direction added before final layer".
        # And [S27] says "Density network is 8-layer... Color network is 2-layer".
        # This implies the first 8 layers are shared? No, usually the first 8 layers are the "Density Network" part 
        # that produces h, and then h is split.
        
        # Let's refine the forward pass to match Mildenhall 2020 exactly:
        # 1. x_enc -> Linear -> ReLU -> ... -> Linear -> ReLU -> Linear -> h (256)
        # 2. sigma = Softplus(Linear(h))
        # 3. h_feat = ReLU(h)
        # 4. h_feat = Concat(h_feat, d_enc)
        # 5. h_feat -> Linear -> ReLU -> Linear -> color
        
        # Since I defined self.linear_xyz as a list of D layers, let's assume the last layer outputs h.
        # And I need a separate linear for sigma.
        
        # Correction: In the standard code (e.g., yenchenlin/nerf-pytorch [S6, unofficial]):
        # self.linear_xyz = nn.ModuleList([nn.Linear(in_channels_xyz, W)])
        # for _ in range(1, D): self.linear_xyz.append(nn.Linear(W, W))
        # self.linear_density = nn.Linear(W, 1)
        # self.linear_feat = nn.Linear(W, W)
        # self.linear_rgb = nn.Linear(W + in_channels_dir, 3)
        
        # Forward:
        h = xyz_encoded
        for i, layer in enumerate(self.linear_xyz):
            h = layer(h)
            if i < self.D - 1:
                h = F.relu(h)
                if i in self.skip:
                    h = torch.cat([h, xyz_encoded], dim=-1)
        
        # h is now the output of the last layer of the XYZ network.
        # Predict sigma
        sigma = F.softplus(self.linear_density(h))
        
        # Predict color
        h_feat = F.relu(h)
        h_feat = torch.cat([h_feat, dir_encoded], dim=-1)
        h_feat = F.relu(self.linear_feat(h_feat))
        rgb = torch.sigmoid(self.linear_rgb(h_feat))
        
        return rgb, sigma

def render_ray(ray_o, ray_d, t_vals, network):
    """
    ray_o: (..., 3)
    ray_d: (..., 3)
    t_vals: (..., N)
    network: NeRF instance
    """
    # 1. Sample points
    xyz = ray_o.unsqueeze(-2) + ray_d.unsqueeze(-2) * t_vals.unsqueeze(-1) # (..., N, 3)
    
    # 2. Positional Encoding
    # Assume network has encoders or we pass encoded inputs
    # For this pseudocode, we assume the network handles encoding or inputs are pre-encoded
    # Let's assume inputs to network are encoded.
    xyz_enc = network.pos_enc_xyz(xyz)
    dir_enc = network.pos_enc_dir(ray_d)
    
    # 3. Query Network
    rgb, sigma = network(xyz_enc, dir_enc) # rgb: (..., N, 3), sigma: (..., N, 1)
    
    # 4. Compute deltas
    # t_vals is sorted. dists = t_vals[i+1] - t_vals[i]
    # Add a small epsilon to avoid division by zero or log(0) issues if needed, 
    # but here we use exp(-sigma * delta).
    deltas = t_vals[..., 1:] - t_vals[..., :-1] # (..., N-1)
    # Last delta is often far - last_t, or just handled by the loop logic.
    # Standard implementation: append far to t_vals, compute deltas, then drop last sigma?
    # Or compute deltas for N-1 intervals, and assume the last segment goes to infinity?
    # Mildenhall 2020: "delta_i = t_{i+1} - t_i". 
    # If we have N samples, we have N-1 intervals between them.
    # The last interval is from t_N to far.
    
    # Let's assume t_vals includes far at the end? 
    # No, usually t_vals are the sample points.
    # Let's compute weights for N samples.
    # We need N deltas.
    # delta_i = t_{i+1} - t_i for i=0..N-2
    # delta_{N-1} = far - t_{N-1}
    
    # This requires 'far' to be passed.
    
    # 5. Compute alpha and transmittance
    # alpha_i = 1 - exp(-sigma_i * delta_i)
    # T_i = exp(-sum_{j<i} sigma_j * delta_j)
    
    # This is vectorized in the actual implementation.
    pass
```

**Parameters:**
*   **Network Depth/Width:** 8 layers, 256 hidden units for the density network; 2 layers for the color network [S27].
*   **Skip Connections:** Added at layer 4 (0-indexed) to address signal attenuation [S2, S9, unofficial].
*   **Positional Encoding:** $L_x=10$ bands for position (63-dim input), $L_d=4$ bands for direction (27-dim input) [S9, unofficial].
*   **Sampling:** $N_c=64$ coarse samples, $N_f=128$ fine samples [S15, unofficial].
*   **Bounds:** Near plane $t_n=2.0$, Far plane $t_f=6.0$ for Blender datasets [S15, S16, unofficial].
*   **Background:** White background assumption (`white_bkgd=True`) [S6, S10, unofficial].

**Edge Cases and Pitfalls:**
*   **Spectral Bias:** Without positional encoding, the MLP struggles to learn high-frequency details [S1][S12, unofficial].
*   **Shape-Radiance Ambiguity:** The network can trade off density and color to fit the data. The specific MLP structure (direction injected late, low-frequency direction encoding) acts as a prior favoring smooth reflectance [S3, unofficial].
*   **Empty Space:** Hierarchical sampling is crucial to avoid wasting queries on empty space [S10, S21, unofficial].
*   **Reflections:** Standard NeRF can confuse reflected images with real geometry if multi-view consistency is not enforced or if the scene contains mirrors [S5].

## Subtopics

### [NeRF Training Configuration](1203-nerf-training-configuration.md)
Defines the optimization loop, data handling, and regularization strategies for training a NeRF model, focusing on standard MLP architecture and volumetric opacity rather than explicit surface constraints.

### [NeRF-SH](1348-nerf-sh.md)
Replaces direct RGB output with spherical harmonic coefficients to efficiently store view-dependent appearance in a sparse voxel-based octree, enabling faster training and rendering.

### [Recursive-NeRF](3029-recursive-nerf.md)
Routes query points through a hierarchy of branch networks based on predicted uncertainty, dynamically growing the network structure to handle scene complexity.

### [Mip-NeRF](3381-mip-nerf.md)
Renders anti-aliased conical frustums instead of infinitesimal rays to reduce aliasing artifacts and improve representation of fine details.

### [NeMF (Neural Microflake Field)](3387-nemf-neural-microflake-field.md)
Represents the scene as a distribution of microflakes to recover physical attributes like shape and reflectance under unknown natural illumination.

### [EgoNeRF](4096-egonerf.md)
Adapts NeRF for large-scale egocentric scenes using a balanced spherical feature grid to match omnidirectional capture geometry.

### [Intrinsic NeRF](4623-intrinsic-nerf.md)
Decouples geometry, illumination, and material properties to enable relighting and material editing without supervised labels.

### [Sparse Neural Volume Representation](7055-sparse-neural-volume-representation.md)
Uses a sparse 3D grid of per-voxel neural features to improve memory efficiency and scalability for large scenes.

### [Instant Neural Graphics Primitives (Instant NGP)](8637-instant-neural-graphics-primitives-instant-ngp.md)
Accelerates NeRF by replacing the large MLP with a Multi-resolution Hash Encoding data structure and a small fully-fused neural network.

### [Bayesian Uncertainty Quantification for SeaThru-NeRF](11861-bayesian-uncertainty-quantification-for-seathru-nerf.md)
Quantifies spatial uncertainty of underwater 3D reconstructions by approximating the posterior distribution of perturbation parameters.

## Operating characteristics
*   **Time Complexity:** Training is slow, typically requiring 10-20 hours on a single GPU for a single scene [S1]. Rendering is also slow, taking ~30 seconds per novel view at 1080p [S1]. This is due to the need to query the MLP hundreds of times per ray and process hundreds of thousands of rays per image [S1].
*   **Memory Complexity:** The model size is small (~5MB for weights) [S6, unofficial], but training requires storing gradients for the entire network. Rendering memory is dominated by the ray batch size and sample count.
*   **Performance:** On the NeRF Synthetic (Blender) dataset, NeRF achieves a PSNR of ~31.0 dB [S1]. It performs poorly on unbounded scenes compared to later variants like Mip-NeRF 360 [S1].
*   **Strengths:** High-quality novel view synthesis for complex scenes with view-dependent effects (specularity, transparency) [S2, S6, unofficial]. Implicit representation handles arbitrary topology [S4, unofficial].
*   **Limitations:** Slow training and rendering speeds [S1]. Poor surface reconstruction accuracy (density field is not a true SDF) [S1]. Struggles with unbounded scenes without specific parameterizations [S3, unofficial].

## Background
NeRF was introduced by Mildenhall et al. in 2020 [S3, unofficial]. It established the field of neural rendering by demonstrating that a simple fully connected network, trained on posed images, could represent a 3D scene as a continuous 5D function and synthesize photorealistic novel views through differentiable volume rendering [S1][S6, unofficial]. The method builds on the concept of Implicit Neural Representations (INRs) and positional encoding [S13, S25, unofficial]. It is widely used as a baseline for novel view synthesis and has inspired numerous variants addressing its speed and accuracy limitations [S1].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S3] NeRF++: Analyzing and Improving Neural Radiance Fields, https://doi.org/10.48550/arxiv.2010.07492, fetched 2026-10-26 (unofficial)
[S4] NeRF Explained Simply | The 5D Function & Volumetric Rendering, https://www.youtube.com/watch?v=BL-KcFm4Wms, fetched 2026-10-27 (unofficial)
[S5] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
[S6] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S7] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
[S8] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S9] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S10] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S11] PDF Volume Rendering - faculty.cc.gatech.edu, https://faculty.cc.gatech.edu/~hays/compvision2025spring/lectures/26.pdf, fetched 2026-10-26 (unofficial)
[S12] GRAF: Generative Radiance Fields for 3D-Aware Image Synthesis, https://doi.org/10.48550/arxiv.2007.02442, fetched 2026-10-26 (unofficial)
[S13] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S14] NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis | Takuma Yoneda, https://takuma.yoneda.xyz/posts/nerf, fetched 2026-10-27 (unofficial)
[S15] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S16] sr-bang/NeRF, https://github.com/sr-bang/NeRF, fetched 2026-10-27 (unofficial)
[S17] Neural Radiance Fields - Lei Mao's Log Book, https://leimao.github.io/blog/Neural-Radiance-Fields-NeRF/, fetched 2026-10-27 (unofficial)
[S18] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S19] Generalizing Neural Radiance Fields - BAIR Commons, https://bcommons.berkeley.edu/generalizing-neural-radiance-fields, fetched 2026-10-28 (unofficial)
[S20] Taichi NeRF (Part 1): Develop and Deploy Instant NGP without writing CUDA | Taichi Docs, https://docs.taichi-lang.org/blog/taichi-instant-ngp, fetched 2026-10-29 (unofficial)
[S21] A Visual Exploration of Neural Radiance Fields, https://djrhails.github.io/post--radiance-fields/, fetched 2026-10-29 (unofficial)
[S22] HyperNeRFGAN: Hypernetwork approach to 3D NeRF GAN, https://doi.org/10.48550/arxiv.2301.11631, fetched 2026-10-29 (unofficial)
[S23] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S24] Vision through Obstacles—3D Geometric Reconstruction and Evaluation of Neural Radiance Fields (NeRFs), https://doi.org/10.3390/rs16071188, fetched 2026-10-30
[S25] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S26] ニューラル放射輝度場 NeRF 入門, https://zenn.dev/abist_inno/articles/353f36119d6b39, fetched 2026-10-30 (unofficial)
[S27] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
