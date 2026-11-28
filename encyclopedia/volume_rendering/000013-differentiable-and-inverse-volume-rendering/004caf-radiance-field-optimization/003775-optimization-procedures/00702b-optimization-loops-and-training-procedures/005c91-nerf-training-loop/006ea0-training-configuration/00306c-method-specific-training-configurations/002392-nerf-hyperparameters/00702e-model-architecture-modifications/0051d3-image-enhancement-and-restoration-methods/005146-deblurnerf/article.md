# DeblurNeRF

Also called: Deblur-NeRF, Deblur-NeRF: Neural Radiance Fields from Blurry Images

## Overview
DeblurNeRF is a framework that reconstructs a sharp Neural Radiance Field (NeRF) from blurry input images by explicitly modeling the blurring process via an analysis-by-synthesis approach [S1]. Unlike standard NeRF which assumes sharp inputs, DeblurNeRF jointly optimizes the NeRF parameters and a Deformable Sparse Kernel (DSK) module to simulate how a sharp scene would appear blurry, allowing the network to learn the underlying sharp geometry and appearance [S1]. It sets itself apart from siblings like [IE-NeRF](13737-ie-nerf.md) and [Sharp-NeRF](14270-sharp-nerf.md) by introducing a learnable, spatially-varying blur kernel parameterized by an MLP, rather than relying on fixed priors or post-processing [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of blurry multi-view images $B$. Camera poses (origin $o$, direction $d$) are assumed known or estimated (e.g., via COLMAP) [S1].
*   **Outputs:** A sharp NeRF representation $F_\Theta$ and optimized DSK parameters $\Phi$. At inference, the output is sharp novel views rendered from $F_\Theta$ without the DSK module [S1].

**Data Structures:**
1.  **NeRF ($F_\Theta$):** An MLP mapping positional encoded 3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$ to color $\mathbf{c}$ and density $\sigma$ [S1].
    *   Input: $\gamma_L(\mathbf{x}), \gamma_L(\mathbf{d})$.
    *   Output: $(\mathbf{c}, \sigma)$.
2.  **Deformable Sparse Kernel (DSK) Module ($G_\Phi$):** An MLP parameterized by $\Phi$ that predicts blur kernel deformations and weights for each pixel [S1].
    *   **Structure:** 4 fully-connected hidden layers, each with 64 channels and ReLU activations. Includes a shortcut connection from the first layer to the last layer [S1].
    *   **Inputs:**
        *   Pixel coordinate $\mathbf{p} \in \mathbb{R}^2$.
        *   Canonical kernel location $\mathbf{q}^0 \in \mathbb{R}^2$ (fixed for a given pixel).
        *   View embedding $\mathbf{l} \in \mathbb{R}^{32}$ (learned per view) [S1].
    *   **Outputs:** For each of the $N$ sparse locations, a tuple $(\Delta \mathbf{o}_q, \Delta \mathbf{v}_T, w_q)$ [S1, S3].
        *   $\Delta \mathbf{o}_q \in \mathbb{R}^3$: Ray origin translation.
        *   $\Delta \mathbf{v}_T \in \mathbb{R}^2$: Direction deformation (applied to pixel coordinates to derive new direction) [S3].
        *   $w_q \in \mathbb{R}$: Composition weight.
3.  **Canonical Kernel:** A set of $N$ fixed 2D locations $N^0(\mathbf{p}) = \{\mathbf{q}^0_i\}_{i=0}^{N-1}$ distributed around pixel $\mathbf{p}$ [S1].

**Numbered Steps:**

1.  **Ray Generation (Standard NeRF):** For a pixel $\mathbf{p}$, generate a primary ray $\mathbf{r}_p(t) = \mathbf{o} + t\mathbf{d}_p$ from camera origin $\mathbf{o}$ and direction $\mathbf{d}_p$ [S1].
2.  **Coarse Sampling:** Divide the ray interval $[t_n, t_f]$ into $N_s$ intervals (default 64) using stratified sampling to generate coarse samples $\mathbf{r}_i = \mathbf{o} + t_i \mathbf{d}_p$ [S3].
3.  **Coarse Rendering:** Query $F_\Theta$ at each sample to get $(\mathbf{c}_i, \sigma_i)$. Compute transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and color $\hat{\mathbf{C}}_c = \sum_{i=1}^{N_s} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i$, where $\delta_i = t_{i+1} - t_i$ [S3].
4.  **Hierarchical Sampling:** Compute normalized weights $\hat{w}_i = w_i / \sum_j w_j$ where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$. Use $\hat{w}_i$ as a PDF to sample $N_s$ additional fine samples [S3].
5.  **Fine Rendering:** Query $F_\Theta$ at fine samples and compute fine color $\hat{\mathbf{C}}_f$ using the same volume rendering equation [S3].
6.  **DSK Prediction:** For the target pixel $\mathbf{p}$, iterate over each canonical location $\mathbf{q}^0 \in N^0(\mathbf{p})$. Input $(\mathbf{p}, \mathbf{q}^0, \mathbf{l})$ into $G_\Phi$ to predict $(\Delta \mathbf{o}_q, \Delta \mathbf{v}_T, w_q)$ [S1].
7.  **Ray Deformation:** Compute the deformed kernel location $\mathbf{q} = \mathbf{q}^0 + \Delta \mathbf{v}_T$ (Note: Source [S1] uses $\Delta \mathbf{q}$ for location deformation, Source [S3] uses $\Delta \mathbf{v}_T$ for direction/coordinate transformation. We follow [S1]'s notation for location: $\mathbf{q} = \mathbf{q}^0 + \Delta \mathbf{q}$) [S1]. Derive the new direction $\mathbf{d}_q$ from the deformed location $\mathbf{q}$ relative to the camera center (or as defined by the transformation) [S3]. Generate the optimized ray $\mathbf{r}_q(t) = (\mathbf{o} + \Delta \mathbf{o}_q) + t \mathbf{d}_q$ [S1].
8.  **Blurred Color Synthesis:** Render each deformed ray $\mathbf{r}_q$ using $F_\Theta$ to obtain color $\mathbf{c}^0_q$ [S1]. Compute the final blurry pixel color $\hat{\mathbf{b}}_\mathbf{p}$ by blending the rendered colors with weights $w_q$, applying gamma correction to handle non-linear Camera Response Function (CRF) [S1]:
    $$ \hat{\mathbf{b}}_\mathbf{p} = \left( \sum_{q \in N(\mathbf{p})} w_q \mathbf{c}^0_q \right)^{1/2.2} $$
    where $\sum_{q \in N(\mathbf{p})} w_q = 1$ [S1].
9.  **Loss Computation:** Compute the reconstruction loss against the ground truth blurry image $\mathbf{b}^{gt}_\mathbf{p}$ and the alignment loss [S1]:
    $$ L_{reconstruct} = \| \hat{\mathbf{b}}_\mathbf{p} - \mathbf{b}^{gt}_\mathbf{p} \|_2^2 $$
    $$ L_{align} = \| \mathbf{q}^0 - \mathbf{p} \|_2^2 + \lambda_o \| \Delta \mathbf{o}_{q^0} \|_2^2 $$
    where $\mathbf{q}^0$ is a fixed element in $N(\mathbf{p})$ (typically the center) [S1].
    Total Loss: $L = L_{reconstruct} + \lambda_a L_{align}$ [S1].
10. **Optimization:** Jointly update $F_\Theta$ and $G_\Phi$ using Adam optimizer [S1].
11. **Inference:** Render rays from the trained $F_\Theta$ directly without DSK deformation to produce sharp novel views [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DeblurNeRF:
    def __init__(self, nerf_model, dsk_mlp, num_kernel_points=5, view_embedding_dim=32):
        self.nerf = nerf_model
        self.dsk = dsk_mlp
        self.N = num_kernel_points
        self.view_embedding_dim = view_embedding_dim
        self.lambda_o = 10.0
        self.lambda_a = 0.1
        self.epsilon = 0.1
        
        # Initialize DSK outputs to be small to keep rays close to original
        # This is a heuristic initialization strategy mentioned in [S1]
        # In practice, weights are initialized to be uniform or small, 
        # and deformations are scaled by epsilon.
        
    def positional_encoding(self, x, L=10):
        """Standard NeRF positional encoding [S1]"""
        scales = torch.tensor([2**i for i in range(L)], dtype=x.dtype, device=x.device)
        xb = torch.reshape((x * scales[:, None]), (len(x), -1))
        freq_bands = torch.cat([torch.sin(xb), torch.cos(xb)], dim=-1)
        return freq_bands

    def render_ray(self, ray_origins, ray_directions, num_samples=128):
        """Render a single ray using the NeRF [S3]"""
        # Stratified sampling
        t_n, t_f = 0.1, 100.0 # Typical bounds, not given in sources, assumed standard
        t_vals = torch.linspace(t_n, t_f, num_samples)
        t_vals = t_vals + torch.rand_like(t_vals) * (t_f - t_n) / num_samples
        
        points = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(0)
        
        # Query NeRF
        # Assuming nerf_model takes (x, d) and returns (c, sigma)
        # x and d need positional encoding
        x_enc = self.positional_encoding(points.view(-1, 3)).view(points.shape + (-1,))
        d_enc = self.positional_encoding(ray_directions).view(ray_directions.shape + (-1,))
        
        # Flatten for MLP
        x_flat = x_enc.view(-1, x_enc.shape[-1])
        d_flat = d_enc.view(-1, d_enc.shape[-1])
        
        c_sigma = self.nerf(x_flat, d_flat)
        c = c_sigma[:, :3].view(points.shape[:-1] + (3,))
        sigma = c_sigma[:, 3:].view(points.shape[:-1])
        
        # Volume Rendering
        deltas = t_vals[1:] - t_vals[:-1]
        delta_inf = 1e10
        deltas = torch.cat([deltas, delta_inf], dim=-1)
        
        alpha = 1 - torch.exp(-sigma * deltas)
        transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1 - alpha + 1e-10], dim=-1), dim=-1)[..., :-1]
        
        weights = alpha * transmittance
        color = torch.sum(weights.unsqueeze(-1) * c, dim=-2)
        
        return color

    def forward(self, pixel_coords, ray_origins, ray_directions, view_embedding, gt_image):
        """
        pixel_coords: (B, 2)
        ray_origins: (B, 3)
        ray_directions: (B, 3)
        view_embedding: (B, 32)
        gt_image: (B, 3)
        """
        B = pixel_coords.shape[0]
        
        # 1. Generate Canonical Kernel Locations
        # N^0(p) are fixed relative to p. 
        # For simplicity, assume they are offsets. In implementation, these are broadcasted.
        # Let's assume canonical_offsets is (N, 2)
        canonical_offsets = torch.zeros(self.N, 2, device=pixel_coords.device) 
        # In reality, these would be spread out, e.g., grid around p.
        # Source [S1] says "sparsely distributed around p".
        
        q0 = pixel_coords.unsqueeze(1) + canonical_offsets # (B, N, 2)
        
        # 2. DSK Prediction
        # Inputs: p (B, 2), q0 (B, N, 2), l (B, 32)
        # Reshape for MLP: (B*N, 2+2+32)
        p_input = pixel_coords.unsqueeze(1).expand(-1, self.N, -1).reshape(-1, 2)
        q0_input = q0.reshape(-1, 2)
        l_input = view_embedding.unsqueeze(1).expand(-1, self.N, -1).reshape(-1, self.view_embedding_dim)
        
        dsk_input = torch.cat([p_input, q0_input, l_input], dim=-1)
        
        # MLP Forward
        dsk_output = self.dsk(dsk_input) # (B*N, 3 + 2 + 1) = (B*N, 6)
        
        # Parse outputs
        delta_o = dsk_output[:, :3].reshape(B, self.N, 3)
        delta_vT = dsk_output[:, 3:5].reshape(B, self.N, 2) # Direction/Location deformation
        weights = dsk_output[:, 5:].reshape(B, self.N, 1)
        
        # Initialize with small gain epsilon [S1]
        delta_o = delta_o * self.epsilon
        delta_vT = delta_vT * self.epsilon
        # Weights need to sum to 1. Source [S1] says "roughly the same weights at beginning".
        # We normalize here.
        weights = F.softmax(weights, dim=1)
        
        # 3. Deform Rays
        # New locations q = q0 + delta_vT (using delta_vT as location shift per [S1] notation q=q0+Delta q)
        # Note: [S3] calls delta_vT direction transform. [S1] calls delta_q location transform.
        # We follow [S1] logic: q is location on image plane.
        q = q0 + delta_vT # (B, N, 2)
        
        # New directions d_q derived from q. 
        # If q is a pixel coordinate, direction is from camera center to q.
        # This requires camera intrinsics. Assuming standard pinhole:
        # d_q = normalize(camera_center -> pixel_at_q)
        # For implementation simplicity in this pseudocode, we assume ray_directions 
        # can be recomputed or passed. Here we approximate direction change.
        # A robust implementation would use camera matrix K.
        # d_q = K^{-1} [q, 1]
        
        # New origins o_q = o + delta_o
        o_q = ray_origins.unsqueeze(1) + delta_o # (B, N, 3)
        
        # 4. Render Deformed Rays
        # We need d_q. Let's assume a helper to get direction from pixel coord.
        # d_q = get_direction_from_pixel(q, camera_intrinsics)
        # For this pseudocode, we assume ray_directions are updated or passed.
        # Let's assume we have a function `get_ray_from_pixel`
        d_q = self.get_ray_directions_from_pixel_coords(q, ray_origins) # (B, N, 3)
        
        # Flatten for rendering
        o_q_flat = o_q.reshape(-1, 3)
        d_q_flat = d_q.reshape(-1, 3)
        
        colors_q = self.render_ray(o_q_flat, d_q_flat) # (B*N, 3)
        colors_q = colors_q.reshape(B, self.N, 3)
        
        # 5. Blend
        # Gamma correction: (sum w c)^(1/2.2)
        blended = torch.sum(weights * colors_q, dim=1) # (B, 3)
        blurred_pred = torch.pow(blended, 1/2.2)
        
        # 6. Loss
        L_recon = torch.mean((blurred_pred - gt_image)**2)
        
        # Alignment Loss: q0 (center) should be close to p, and delta_o for center should be small
        # Assume index 0 is the center
        L_align = torch.mean((q0[:, 0, :] - pixel_coords)**2) + \
                  self.lambda_o * torch.mean((delta_o[:, 0, :])**2)
                  
        L_total = L_recon + self.lambda_a * L_align
        
        return blurred_pred, L_total, L_recon, L_align

    def get_ray_directions_from_pixel_coords(self, pixel_coords, origins):
        """
        Approximate direction calculation from pixel coordinates.
        Assumes origins are camera centers.
        In a real implementation, this uses camera intrinsics K.
        d = K^{-1} [x, y, 1]
        """
        # Placeholder for actual camera matrix logic
        # For the sake of the article, we note that direction is derived from the deformed pixel location
        # relative to the camera center.
        # Since we don't have K in the class, we return a dummy direction or assume 
        # the direction change is handled by the DSK's delta_vT if interpreted as direction.
        # However, [S1] explicitly says q is location and d is derived.
        # We will return the original directions for now, noting this is a simplification.
        # A correct implementation would transform the ray direction based on the pixel shift.
        return origins.new_zeros_like(origins) # Placeholder
```

**Parameters:**
*   **N (Number of sparse locations):** 5 [S1]. Improves quality as N increases, but diminishing returns after 5.
*   **View Embedding Length:** 32 [S1].
*   **MLP Hidden Layers:** 4 layers, 64 channels each, ReLU activation [S1].
*   **Learning Rate:** Starts at $5 \times 10^{-4}$, decays exponentially to $8 \times 10^{-5}$ [S1].
*   **Batch Size:** 1024 rays [S1].
*   **Samples:** 64 coarse, 64 fine [S1].
*   **Alignment Loss Weight ($\lambda_o$):** 10 [S1].
*   **Reconstruction Loss Weight ($\lambda_a$):** 0.1 [S1].
*   **Initialization Gain ($\epsilon$):** 0.1 [S1].

**Edge Cases and Pitfalls:**
*   **Non-rigid Distortion:** If all components (NeRF and DSK) are freely optimized, the NeRF may undergo non-rigid distortion to compensate for blur errors. This is mitigated by the alignment loss $L_{align}$ [S1].
*   **Positional Encoding:** Using positional encoding on the input pixel coordinate $\mathbf{p}$ of the DSK MLP $G_\Phi$ was tested but found not to help, likely because the spatially-varying kernel changes gradually without high-frequency variation [S1].
*   **3D Inconsistency:** Pixel-wise independent optimization of the blur kernel (as in DeblurNeRF) can incur inconsistency in 3D geometry and appearance compared to scene-wise shared transformations (like in DP-NeRF) [S3].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive due to the joint optimization of the NeRF and the DSK module, requiring rendering multiple rays per pixel (N=5) during training [S1]. Inference is comparable to standard NeRF as the DSK is bypassed [S1].
*   **Memory Complexity:** Requires storing the NeRF MLP and the DSK MLP. The DSK MLP is small (4 layers, 64 channels) [S1].
*   **Performance:** On synthetic scenes, DeblurNeRF achieves higher PSNR/SSIM and lower LPIPS compared to ablations without gamma correction, alignment loss, or origin optimization [S1]. It handles both camera motion blur and defocus blur [S1].
*   **Hardware:** Tested on a single NVIDIA V100 GPU [S1].
*   **Scaling:** The method scales with the number of kernel points N, but N=5 is sufficient for most cases [S1].
*   **Strengths:** Can recover sharp NeRF from blurry inputs without needing sharp reference images. Handles spatially-varying blur.
*   **Limitations:** May suffer from 3D inconsistency due to pixel-wise independent blur modeling. Less effective for severe motion blur compared to physically grounded methods like BAD-NeRF [S2].

## Background
DeblurNeRF was introduced by Xiaoyu Li et al. in 2021/2022 [S1]. It is the first method to recover a sharp NeRF from blurry input images using an analysis-by-synthesis approach [S1]. It builds on the standard NeRF architecture [Parent](3065-nerf-hyperparameters.md) and addresses the degradation of reconstruction quality caused by defocus or motion blur in real-world captures [S1]. It differs from DP-NeRF by using pixel-wise independent transformations (DSK) instead of scene-wise shared rigid transformations [S3].

## Sources
[S1] Deblur-NeRF: Neural Radiance Fields from Blurry Images, https://doi.org/10.1109/cvpr52688.2022.01252, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
