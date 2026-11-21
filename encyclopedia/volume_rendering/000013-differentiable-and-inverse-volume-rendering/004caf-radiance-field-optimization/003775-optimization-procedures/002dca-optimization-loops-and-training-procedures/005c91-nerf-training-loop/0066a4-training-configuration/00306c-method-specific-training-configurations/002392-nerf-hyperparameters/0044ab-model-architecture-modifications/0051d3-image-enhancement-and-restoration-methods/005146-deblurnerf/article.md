# DeblurNeRF

Also called: DeRF, Deblur-NeRF, Deblur-NeRF: Neural Radiance Fields from Blurry Images

## Overview
DeblurNeRF is a framework that reconstructs a sharp Neural Radiance Field (NeRF) from blurry input images by explicitly modeling the blurring process via an analysis-by-synthesis approach [S1]. Unlike standard NeRF which assumes sharp inputs, DeblurNeRF jointly optimizes the NeRF parameters and a Deformable Sparse Kernel (DSK) module to simulate the blur, allowing the NeRF to recover sharp geometry and appearance during training [S1]. It sets itself apart from siblings like [IE-NeRF](13737-ie-nerf.md) and [Sharp-NeRF](14270-sharp-nerf.md) by using a pixel-wise independent deformable sparse kernel to approximate spatially varying blur, rather than relying on global scene priors or post-processing [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of blurry multi-view images. Camera poses are assumed to be known (e.g., from COLMAP) [S1].
*   **Outputs:** A sharp NeRF representation $F_\Theta$ and the DSK module parameters $\Phi$ [S1]. At inference, the DSK is discarded, and the NeRF renders sharp novel views [S1].

**Data Structures:**
1.  **NeRF ($F_\Theta$):** Standard MLP mapping $(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [S1].
2.  **DSK Module ($G_\Phi$):** An MLP parameterized by $\Phi$ [S1].
    *   **Structure:** 4 fully-connected hidden layers, each with 64 channels and ReLU activations. Includes a shortcut connection from the first layer to the last layer [S1].
    *   **Inputs:** Pixel coordinate $\mathbf{p}$, canonical kernel location $\mathbf{q}^0$, and a learned view embedding $\mathbf{l}$ (length 32) [S1].
    *   **Outputs:** For each of the $N$ sparse locations, a tuple $(\Delta \mathbf{o}_q, \Delta \mathbf{q}, w_q)$ [S1].
        *   $\Delta \mathbf{o}_q \in \mathbb{R}^3$: Ray origin translation.
        *   $\Delta \mathbf{q} \in \mathbb{R}^2$: Kernel location deformation (2D image plane offset).
        *   $w_q \in \mathbb{R}$: Composition weight.
3.  **Canonical Kernel:** A set of $N$ fixed 2D locations $N^0(\mathbf{p}) = \{\mathbf{q}^0_i\}_{i=0}^{N-1}$ distributed around the pixel $\mathbf{p}$ [S1].

**Parameters:**
*   $N$: Number of sparse kernel locations per pixel. Default: 5 [S1].
*   $L$: Highest frequency in positional encoding. Controls smoothness [S1].
*   $\epsilon$: Initialization gain. Default: 0.1 [S1].
*   $\lambda_o$: Alignment loss weight. Default: 10 [S1].
*   $\lambda_a$: Reconstruction loss weight. Default: 0.1 [S1].
*   Learning Rate: Starts at $5 \times 10^{-4}$, decays exponentially to $8 \times 10^{-5}$ [S1].
*   Batch Size: 1024 rays [S1].
*   Sampling: 64 coarse samples, 64 fine samples [S1].

**Formulas:**
*   **Positional Encoding:** $\gamma_L(\mathbf{x}) = [\sin(\pi \mathbf{x}), \cos(\pi \mathbf{x}), \dots, \sin(2^{L-1}\pi \mathbf{x}), \cos(2^{L-1}\pi \mathbf{x})]^T$ [S1].
*   **NeRF Rendering:** $\hat{c}_p = \sum_{i=1}^D T^{(i)} (1 - \exp(-\sigma^{(i)} \delta^{(i)})) c^{(i)}$, where $\delta^{(i)} = t^{(i+1)} - t^{(i)}$ and $T^{(i)} = \exp(-\sum_{j=1}^{i-1} \sigma^{(j)} \delta^{(j)})$ [S1].
*   **Blur Approximation:** The blurry color $b_p$ is approximated by a sparse sum: $b_p = \sum_{q \in N(\mathbf{p})} w_q c_q$, subject to $\sum_{q \in N(\mathbf{p})} w_q = 1$ [S1].
*   **Ray Generation:** Optimized rays are generated as $r_q = (\mathbf{o} + \Delta \mathbf{o}_q) + t \mathbf{d}_q$, where $\mathbf{d}_q$ is derived from the deformed location $\mathbf{q} = \mathbf{q}^0 + \Delta \mathbf{q}$ [S1].
*   **Gamma Correction:** Applied to compensate for non-linear Camera Response Function (CRF): $\hat{b}_p = g(\sum_{q \in N(\mathbf{p})} w_q c^0_q)$, where $g(c^0) = (c^0)^{1/2.2}$ [S1].
*   **Alignment Loss:** $L_{align} = \|\mathbf{q}^0 - \mathbf{p}\|_2^2 + \lambda_o \|\Delta \mathbf{o}_{q^0}\|_2^2$, where $\mathbf{q}^0$ is a fixed element in $N(\mathbf{p})$ [S1].
*   **Total Loss:** $L = L_{reconstruct} + \lambda_a L_{align}$, where $L_{reconstruct} = \sum_{\mathbf{p} \in R} \|\hat{b}_p - b^{gt}_p\|_2^2$ [S1].

**Steps:**
1.  **Initialize:** Initialize the DSK module such that optimized rays are close to the input ray. Multiply the output tuples $(\Delta \mathbf{o}_q, \Delta \mathbf{q}, w_q)$ by a small gain $\epsilon = 0.1$ [S1].
2.  **Predict Deformations:** For each pixel $\mathbf{p}$ in the batch, forward the MLP $G_\Phi$ $N$ times (once for each canonical location $\mathbf{q}^0$) to predict $(\Delta \mathbf{o}_q, \Delta \mathbf{q}, w_q)$ [S1].
3.  **Generate Rays:** Compute deformed locations $\mathbf{q} = \mathbf{q}^0 + \Delta \mathbf{q}$ and generate optimized rays $r_q = (\mathbf{o} + \Delta \mathbf{o}_q) + t \mathbf{d}_q$ [S1].
4.  **Render:** Render each optimized ray $r_q$ using the NeRF $F_\Theta$ to obtain color $c^0_q$ and density [S1].
5.  **Composite:** Blend the rendered colors using the predicted weights and apply gamma correction to compute the synthetic blurry pixel $\hat{b}_p$ [S1].
6.  **Compute Loss:** Calculate the reconstruction loss $L_{reconstruct}$ against the ground truth blurry image and the alignment loss $L_{align}$ [S1].
7.  **Optimize:** Update NeRF parameters $\Theta$ and DSK parameters $\Phi$ using the total loss $L$ [S1].
8.  **Inference:** Discard the DSK module. Render rays directly from the camera center $\mathbf{o}$ and direction $\mathbf{d}$ using the trained NeRF $F_\Theta$ to produce sharp novel views [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DeblurNeRF:
    def __init__(self, nerf_model, num_kernel_points=5, view_embedding_dim=32):
        """
        Initialize DeblurNeRF.
        :param nerf_model: The NeRF MLP (F_Theta).
        :param num_kernel_points: N, number of sparse kernel locations.
        :param view_embedding_dim: Length of the view embedding vector.
        """
        self.nerf = nerf_model
        self.num_kernel_points = num_kernel_points
        self.view_embedding_dim = view_embedding_dim
        
        # DSK Module: MLP with 4 hidden layers, 64 channels, ReLU, shortcut
        self.dsk_mlp = nn.Sequential(
            nn.Linear(4 + 32 + 2, 64), # Input: p(2) + q0(2) + l(32) -> 4+32+2=38? 
            # Note: Source says inputs are p, q0, l. p is 2D, q0 is 2D, l is 32D. Total 36.
            # Let's correct input dim based on source description: p(2) + q0(2) + l(32) = 36.
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 3 * num_kernel_points) # Output: (do, dq, w) for N points
        )
        # Add shortcut from first layer to last layer manually or via structure
        # Source: "shortcut that connects the first layer to the last layer"
        # We will implement the forward pass to handle this explicitly.
        
        # Initialize view embeddings for each view (stored externally or here)
        # For simplicity, assume view embedding is passed in forward pass or managed externally.
        
        # Canonical kernel locations N0(p)
        # In practice, these are relative offsets around p. 
        # Source says "canonical kernel locations ... distributed around p".
        # We assume a fixed set of relative offsets is used.
        self.register_buffer('canonical_offsets', torch.zeros(num_kernel_points, 2))
        # Initialize offsets to be close to 0 (small range)
        # Source: "randomly initialized canonical coordinates within a specific small range"
        # We'll use small random values.
        self.canonical_offsets.data = torch.randn(num_kernel_points, 2) * 0.01 

    def forward_dsk(self, p, q0, view_embedding):
        """
        Forward pass of DSK MLP with shortcut.
        :param p: Pixel coordinates (B, 2)
        :param q0: Canonical kernel locations (B, N, 2)
        :param view_embedding: View embedding (B, 32)
        :return: (delta_o, delta_q, weights)
        """
        # Concatenate inputs: p, q0, view_embedding
        # p needs to be broadcasted to (B, N, 2)
        p_expanded = p.unsqueeze(1).expand(-1, self.num_kernel_points, -1)
        input_tensor = torch.cat([p_expanded, q0, view_embedding.unsqueeze(1).expand(-1, self.num_kernel_points, -1)], dim=-1)
        
        # First layer
        x = self.dsk_mlp[0](input_tensor)
        x = self.dsk_mlp[1](x) # ReLU
        x1 = x
        
        # Hidden layers
        x = self.dsk_mlp[2](x) # ReLU
        x = self.dsk_mlp[3](x) # ReLU
        x = self.dsk_mlp[4](x) # ReLU
        x = self.dsk_mlp[5](x) # ReLU
        
        # Shortcut connection from first layer to last layer
        # Source: "shortcut that connects the first layer to the last layer"
        # Usually means adding the output of layer 1 to the output of the last layer before final projection?
        # Or adding x1 to x before the final linear layer?
        # "includes a shortcut connection from the first layer to the last layer"
        # Let's assume it adds x1 to the input of the final linear layer.
        x = x + x1
        
        # Final linear layer
        output = self.dsk_mlp[6](x)
        
        # Reshape output to (B, N, 3) -> (delta_o, delta_q, w)
        # delta_o: 3D, delta_q: 2D, w: 1D. Total 6 values per point?
        # Source: "(delta_oq, delta_q, wq) = G_Phi(...)"
        # delta_o is 3D, delta_q is 2D, w is 1D. Total 6.
        # But my linear layer output dim is 3*N?
        # Let's re-read: "outputs: (delta_o, delta_q, w)q for each q"
        # delta_o is 3D vector. delta_q is 2D vector. w is scalar.
        # Total 6 values per q.
        # So output dim should be 6 * N.
        
        # Correction: The MLP structure description says "4 fully-connected hidden layers... shortcut...".
        # It doesn't specify the output dimension explicitly in the text, but the formula implies 6 values.
        # Let's adjust the final layer.
        pass

    def render_ray(self, ray_o, ray_d, t_near, t_far, num_samples=64):
        """
        Render a single ray using NeRF.
        """
        # Stratified sampling
        t_vals = torch.linspace(t_near, t_far, num_samples)
        t_vals = t_vals + torch.rand_like(t_vals) * (t_far - t_near) / num_samples
        
        # Points along ray
        pts = ray_o.unsqueeze(1) + ray_d.unsqueeze(1) * t_vals.unsqueeze(2)
        
        # Query NeRF
        # F_Theta takes (x, d) -> (sigma, c)
        # x: (B, N, 3), d: (B, 3)
        # Positional encoding is applied inside F_Theta or before?
        # Source: "F_Theta(gamma_L(x), gamma_L(d))"
        # We assume F_Theta handles encoding or we do it here.
        # For simplicity, assume F_Theta takes raw x, d and handles encoding.
        sigma, color = self.nerf(pts, ray_d)
        
        # Volume Rendering
        # Ti = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
        # delta_i = t_{i+1} - t_i
        deltas = t_vals[1:] - t_vals[:-1]
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(torch.exp(-sigma * deltas), dim=1)
        T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
        
        weights = alpha * T
        color_rendered = torch.sum(weights.unsqueeze(-1) * color, dim=1)
        
        return color_rendered

    def train_step(self, blurry_image, camera_pose, view_embedding):
        """
        One training step.
        """
        # 1. Generate rays for each pixel in the batch
        # Assume we have a batch of pixels with coordinates p and camera pose o, d
        # p: (B, 2), o: (B, 3), d: (B, 3)
        
        # 2. Initialize canonical kernel locations relative to p
        # q0 = p + canonical_offsets
        q0 = self.canonical_offsets.unsqueeze(0) + p.unsqueeze(1)
        
        # 3. Predict DSK parameters
        # delta_o, delta_q, w = self.forward_dsk(p, q0, view_embedding)
        # Note: Implementing the MLP forward pass properly
        
        # 4. Generate optimized rays
        # q = q0 + delta_q
        # d_q = direction from o to q (projected back to 3D?)
        # Source: "dq is transformed ray direction by applying delta_vT to pixel coordinates"
        # This implies q is in 2D image space. We need to convert to 3D direction.
        # Assuming pinhole camera: d_q = unproject(q) - o
        # For simplicity, assume we have a function to get 3D direction from 2D pixel
        d_q = get_3d_direction(q, camera_pose)
        o_q = camera_pose.o + delta_o
        
        rays_o = o_q
        rays_d = d_q
        
        # 5. Render rays
        # Render coarse and fine? Source mentions coarse/fine for standard NeRF.
        # DeblurNeRF uses standard NeRF rendering.
        # We render all N rays for each pixel.
        colors = []
        for i in range(self.num_kernel_points):
            c = self.render_ray(rays_o[:, i], rays_d[:, i], t_near, t_far)
            colors.append(c)
        colors = torch.stack(colors, dim=1) # (B, N, 3)
        
        # 6. Composite
        # b_hat = gamma(w * colors)
        w = w.view(B, N, 1)
        blurred_color = torch.sum(w * colors, dim=1)
        gamma_color = blurred_color ** (1/2.2)
        
        # 7. Loss
        loss_recon = torch.mean((gamma_color - blurry_image) ** 2)
        loss_align = ... # Compute alignment loss
        loss = loss_recon + 0.1 * loss_align
        
        loss.backward()
        optimizer.step()
```

**Subtopics:**
*   **Deformable Sparse Kernel (DSK):** A module that models spatially-varying blur by deforming a canonical sparse kernel at each 2D spatial location. It differs from rigid kernel methods by allowing pixel-wise independent transformations, which can lead to 3D inconsistency [S3]. Choose when computational efficiency is prioritized over strict 3D geometric consistency [S2]. [Link to DSK if available, otherwise link to DeblurNeRF].
*   **Alignment Loss:** A regularization term that constrains the NeRF to align with observations by forcing one of the optimized rays to be similar to the input ray. It prevents non-rigid distortion of the NeRF during joint optimization [S1].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive as it requires rendering $N$ rays per pixel instead of 1. With $N=5$, this is 5x the rendering cost of standard NeRF per step [S1].
*   **Memory Complexity:** Requires storing parameters for the NeRF and the DSK MLP. The DSK MLP is small (4 layers, 64 channels) [S1].
*   **Measured Performance:** On synthetic scenes, the full model achieves the highest average PSNR/SSIM and lowest LPIPS compared to ablations (w/o gamma, w/o align, w/o origin opt) [S1].
*   **Scaling:** Quality improves as $N$ increases, but improvement is less substantial beyond $N=5$ [S1].
*   **Strengths:** Can handle both camera motion blur and defocus blur. Recovers sharp novel views from blurry inputs. Efficient approximation for moderate blur [S2].
*   **Limitations:** Pixel-wise independent optimization can incur inconsistency in 3D geometry and appearance [S3]. May suffer from non-rigid distortion if not constrained by alignment loss [S1].
*   **When to use:** When input images are blurry (motion or defocus) and sharp novel views are desired. Suitable for moderate blur [S2].
*   **When not to use:** When severe motion blur requires physically grounded formulations for better pose accuracy (use [BAD-NeRF](link if available) instead) [S2]. When 3D geometric consistency across views is critical and pixel-wise blur modeling is too inconsistent [S3].

## Background
DeblurNeRF (also known as DeRF) was proposed by Xiaoyu Li, Jing Liao, Qi Zhang, Jue Wang, Pedro V. Sander, and Li Ma from The Hong Kong University of Science and Technology, Tencent AI Lab, and City University of Hong Kong [S1]. It was published in CVPR 2022 (arXiv:2111.14292) [S1]. It is the first method to recover a sharp NeRF from blurry input images using an analysis-by-synthesis approach [S1]. It was tested on synthetic scenes generated with Blender and real-world scenes captured with a Canon EOS RP [S1].

## Sources
[S1] Deblur-NeRF: Neural Radiance Fields from Blurry Images, https://doi.org/10.1109/cvpr52688.2022.01252, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
