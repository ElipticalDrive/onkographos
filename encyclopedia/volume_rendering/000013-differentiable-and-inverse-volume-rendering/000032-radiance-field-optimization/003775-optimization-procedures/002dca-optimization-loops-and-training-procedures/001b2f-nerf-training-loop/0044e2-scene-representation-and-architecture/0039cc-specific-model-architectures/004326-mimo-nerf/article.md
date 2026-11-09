# MIMO-NeRF

Also called: MIMO-NeRF-distill, MIMO-NeRF-naive, MIMO-NeRF-self

## Overview
MIMO-NeRF is a neural rendering method that accelerates inference by replacing the standard Single-Input Single-Output (SISO) Multi-Layer Perceptron (MLP) with a Multi-Input Multi-Output (MIMO) MLP [S1]. It solves the problem of high computational cost in NeRF by processing groups of $N_p$ neighboring samples on a ray simultaneously, thereby reducing the number of MLP evaluations (# Run) per pixel [S1]. Unlike methods that reduce sample counts or network width to save compute, MIMO-NeRF maintains the original NeRF hyperparameters (sample counts and hidden dimensions) but groups inputs to parallelize computation [S1]. It distinguishes itself from knowledge distillation approaches by supporting self-supervised training, which avoids the need to train a separate teacher network [S1].

## How it works
**Inputs and Outputs:**
The method takes a set of camera poses and images as input. For a single ray, it accepts $N_p$ spatial positions $\mathbf{x}_i, \dots, \mathbf{x}_j$ and a single view direction $\mathbf{d}$. It outputs $N_p$ colors $\mathbf{c}_i, \dots, \mathbf{c}_j$ and $N_p$ volume densities $\sigma_i, \dots, \sigma_j$, where $j = i + N_p - 1$ [S1].

**Data Structures:**
The network architecture is identical to the baseline NeRF MLP except for the input and output dimensions [S1].
*   **Positional Encoding:** Input positions $\mathbf{x} \in \mathbb{R}^3$ and view direction $\mathbf{d} \in \mathbb{S}^2$ are encoded using positional encoding [S1].
    *   $\gamma(\mathbf{x})$ is a 63-dimensional vector.
    *   $\gamma(\mathbf{d})$ is a 27-dimensional vector [S1].
*   **MIMO MLP:**
    *   **Input Layer:** Accepts the concatenated positional encodings for $N_p$ points: $[\gamma(\mathbf{x}_i), \dots, \gamma(\mathbf{x}_j)] \in \mathbb{R}^{63 N_p}$ and $\gamma(\mathbf{d}) \in \mathbb{R}^{27}$ [S1].
    *   **Hidden Layers:** An 8-layer MLP with ReLU activation, each layer having 256 hidden units [S1].
    *   **Skip Connection:** A skip connection incorporates the encoded position $\gamma(\mathbf{x})$ (concatenated for all $N_p$ points) into the 5th layer [S1].
    *   **Density Branch:** A linear layer maps the 5th layer output to $\sigma \in \mathbb{R}^{N_p}$ [S1].
    *   **Color Branch:** The final layer output (256 units) is concatenated with $\gamma(\mathbf{d})$, passed through a 1-layer MLP (ReLU) to produce a 128-dimensional vector, and finally through a linear layer to produce $\mathbf{c} \in \mathbb{R}^{3 N_p}$ [S1].
*   **Reformulated MLPs:** To handle the increased input dimensionality and ambiguity, $M$ reformulated MLPs are used, where $M$ depends on $N_p$ [S1].
    *   Scaling factors $R_m$ are applied to the inputs of these MLPs.
    *   For $N_p=2$: $M=2$, $R_1=1, R_2=1$ [S1].
    *   For $N_p=4$: $M=2$, $R_1=1, R_2=2$ [S1].
    *   For $N_p=8$: $M=3$, $R_1=1, R_2=2, R_3=4$ [S1].

**Numbered Steps:**
1.  **Grouping:** For a ray with $N$ total samples, group $N_p$ neighboring samples starting from the sample nearest to the camera. This creates $N/N_p$ groups [S1].
2.  **Encoding:** For each group $i$ (where $i \in \{1, 1+N_p, \dots, 1+N-N_p\}$), compute positional encodings $\gamma(\mathbf{x}_k)$ for $k \in \{i, \dots, i+N_p-1\}$ and $\gamma(\mathbf{d})$ [S1].
3.  **Forward Pass:** Apply the MIMO function $f_{\text{MIMO}}$ to the group. The function maps the grouped positions and direction to outputs:
    $$f_{\text{MIMO}} : (\mathbb{R}^3)^{N_p} \times \mathbb{S}^2 \rightarrow (\mathbb{R}^3 \times \mathbb{R}^+)^{N_p}$$
    $$(\mathbf{x}_i, \dots, \mathbf{x}_j, \mathbf{d}) \mapsto (\mathbf{c}_i, \dots, \mathbf{c}_j, \sigma_i, \dots, \sigma_j)$$
    where $j = i + N_p - 1$ [S1].
4.  **Rendering:** Use standard volume rendering integration (Equation 2 in [S1]) and sampling schemes (stratified and hierarchical) with the outputs from $f_{\text{MIMO}}$ [S1].
5.  **Training (Self-Supervised):**
    *   Compute pixel-wise loss $\mathcal{L}_{\text{pixel}}^{\text{MIMO}}$ between rendered and ground truth colors [S1].
    *   Compute 3D consistency loss $\mathcal{L}_{\text{3D}}$ to resolve ambiguity in color and density [S1].
    *   Total loss: $\mathcal{L}_{\text{MIMO}} = \mathcal{L}_{\text{pixel}}^{\text{MIMO}} + \lambda \mathcal{L}_{\text{3D}}$ [S1].

**Formulas:**
*   **3D Consistency Loss:**
    $$\mathcal{L}_{\text{3D}} = \mathcal{L}_{\text{color 3D}} + \mathcal{L}_{\text{alpha 3D}}$$
    $$\mathcal{L}_{\text{color 3D}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\mathbf{c}_i^{m_1} - \text{sg}(\mathbf{c}_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\mathbf{c}_i^{m_1}) - \mathbf{c}_i^{m_2}\|_2^2 \right]$$
    $$\mathcal{L}_{\text{alpha 3D}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\alpha_i^{m_1} - \text{sg}(\alpha_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\alpha_i^{m_1}) - \alpha_i^{m_2}\|_2^2 \right]$$
    where $\text{sg}$ denotes the stop-gradient operation [S1].
*   **Asymmetric Weighting:**
    $$\mu_{m_j}^{m_i} = \frac{\sqrt{m_j} \sqrt{R_{\max}}}{\sqrt{R_{m_i}}}$$
    where $R_{\max} = \max_{m \in \{1, \dots, M\}} R_m$. This weight reduces the loss effect for higher $R_m$, assuming they have lower ambiguity [S1].
*   **Inference Cost (# Run):**
    $$\# \text{Run} = \frac{N_c}{N_p} + \frac{N_c + N_f}{N_p}$$
    where $N_c$ is the number of coarse samples, $N_f$ is the number of fine samples, and $N_p$ is the number of grouped samples [S1].

**Parameters:**
*   $N_p$: Number of grouped samples. Typical values: 2, 4, 8 [S1].
*   $N_c, N_f$: Number of coarse and fine samples.
    *   Blender dataset: $N_c=64, N_f=128$ [S1].
    *   LLFF dataset: $N_c=64, N_f=64$ [S1].
*   $\lambda$: Weight for 3D consistency loss.
    *   Blender: $\lambda=1$ [S1].
    *   LLFF: $\lambda=0.4$ [S1].
*   $R_m$: Scaling factors for reformulated MLPs (see Data Structures) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class PositionalEncoding(nn.Module):
    """Standard NeRF positional encoding."""
    def __init__(self, num_freqs):
        super().__init__()
        self.num_freqs = num_freqs

    def forward(self, x):
        # x shape: (..., 3)
        # Output shape: (..., 3 * 2 * num_freqs)
        # Note: Standard NeRF uses 10 freqs for pos (63 dim) and 4 for view (27 dim)
        # This is a placeholder for the specific NeRF PE implementation
        pass

class MIMOMLP(nn.Module):
    """
    MIMO-NeRF MLP.
    Accepts Np positions and 1 view direction.
    Outputs Np colors and Np densities.
    """
    def __init__(self, Np, hidden_dim=256, num_layers=8, skip_layer=5):
        super().__init__()
        self.Np = Np
        self.hidden_dim = hidden_dim
        
        # Input dimension: Np * 63 (pos) + 27 (view)
        self.input_dim = Np * 63 + 27
        
        # Layers
        self.layers = nn.ModuleList()
        for i in range(num_layers):
            if i == 0:
                self.layers.append(nn.Linear(self.input_dim, hidden_dim))
            elif i == skip_layer:
                # Skip connection adds input dim back
                self.layers.append(nn.Linear(hidden_dim + self.input_dim, hidden_dim))
            else:
                self.layers.append(nn.Linear(hidden_dim, hidden_dim))
        
        # Density head
        self.density_head = nn.Linear(hidden_dim, Np)
        
        # Color head
        # 1-layer MLP (ReLU) to 128, then linear to RGB
        self.color_mlp = nn.Sequential(
            nn.Linear(hidden_dim + 27, 128),
            nn.ReLU(),
            nn.Linear(128, Np * 3)
        )

    def forward(self, positions, view_dir):
        """
        positions: (batch, Np, 3)
        view_dir: (batch, 3)
        """
        batch_size = positions.shape[0]
        
        # Encode positions: (batch, Np, 63) -> (batch, Np*63)
        pos_enc = self.pos_enc(positions.reshape(-1, 3)).reshape(batch_size, -1)
        
        # Encode view dir: (batch, 27)
        view_enc = self.pos_enc_view(view_dir)
        
        # Concatenate
        x = torch.cat([pos_enc, view_enc], dim=-1)
        
        # Pass through MLP
        for i, layer in enumerate(self.layers):
            if i == skip_layer:
                x = torch.cat([x, pos_enc], dim=-1)
            x = torch.relu(layer(x))
            
        # Outputs
        sigma = self.density_head(x) # (batch, Np)
        color_raw = self.color_mlp(torch.cat([x, view_enc], dim=-1)) # (batch, Np*3)
        color = color_raw.reshape(batch_size, self.Np, 3)
        
        return color, sigma

class MIMONeRF:
    def __init__(self, Np, Nc, Nf, lambda_3d=1.0):
        self.Np = Np
        self.Nc = Nc
        self.Nf = Nf
        self.lambda_3d = lambda_3d
        
        # Create MIMO MLPs (M reformulated MLPs)
        # For simplicity, this example assumes a single MIMO MLP structure
        # In practice, multiple reformulated MLPs with scaling factors R_m are used
        self.mimo_mlp = MIMOMLP(Np)
        
    def render_ray(self, ray_origins, ray_directions, t_near, t_far, Nc, Nf):
        """
        Standard volume rendering with MIMO grouping.
        """
        # 1. Sample coarse points
        t_c = t_near + (t_far - t_near) * torch.rand(Nc) # Stratified sampling
        x_c = ray_origins + ray_directions * t_c.unsqueeze(-1)
        
        # 2. Group coarse samples
        # Reshape to (Nc/Np, Np, 3)
        x_c_groups = x_c.reshape(-1, self.Np, 3)
        
        # 3. Forward MIMO MLP
        # view_dir is constant for the ray
        c_c, sigma_c = self.mimo_mlp(x_c_groups, ray_directions)
        
        # 4. Compute weights and color (standard volume rendering)
        # ... (standard NeRF integration logic) ...
        
        # 5. Sample fine points (hierarchical sampling)
        # ...
        
        # 6. Group fine samples and forward pass
        # ...
        
        # Return rendered color
        pass

    def compute_loss(self, rendered_color, gt_color, sigma_outputs):
        """
        Compute MIMO loss including 3D consistency.
        """
        # Pixel loss
        pixel_loss = torch.mean((rendered_color - gt_color) ** 2)
        
        # 3D Consistency Loss (simplified)
        # In practice, this involves multiple reformulated MLPs (M)
        # and asymmetric weighting based on R_m
        # 3D_loss = L_color_3D + L_alpha_3D
        # ...
        
        total_loss = pixel_loss + self.lambda_3d * 0 # Placeholder for 3D loss
        return total_loss
```

## Subtopics

### MIMO-TensoRF
Application of MIMO-NeRF to the TensoRF architecture to improve inference and training speed without deteriorating image quality [S1]. It selects appropriate $N_p$ values: $N_p \le 2$ for Blender dataset and $N_p \le 4$ for LLFF dataset [S1]. It uses TensoRF tensor rank parameters $R_\sigma$ and $R_c$ (e.g., $R_\sigma=16, R_c=48$ for Blender; $R_{\sigma,1}=R_{\sigma,2}=4, R_{\sigma,3}=16, R_{c,1}=R_{c,2}=12, R_{c,3}=48$ for LLFF) [S1]. MIMO-TensoRF improves I-time and T-time of original TensoRFs with similar image quality [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Inference Cost (# Run):** $\# \text{Run} = \frac{N_c}{N_p} + \frac{N_c + N_f}{N_p}$. Increasing $N_p$ reduces # Run linearly [S1].
*   **FLOPs:** MIMO-NeRF reduces FLOPs compared to baseline NeRF. For Blender ($N_c=64, N_f=128$): NeRF has 303.82 FLOPs (normalized); MIMO-NeRF with $N_p=2$ has 160.33; $N_p=4$ has 88.59; $N_p=8$ has 52.72 [S1].
*   **Parameters:** # Params increases by $1.06\times$ for $N_p=2$, $1.17\times$ for $N_p=4$, and $1.39\times$ for $N_p=8$ compared to baseline NeRF [S1]. This is because the total dimension of encoded position $\gamma(\mathbf{x})$ increases by $N_p$ times, but network weights do not scale $N_p$ times [S1].

**Measured Performance:**
*   **Blender ($N_p=2$):** PSNR 31.26, SSIM 0.953, LPIPS 0.054, I-time 5.15s, T-time 5.36h, #Run 128, #Params 1.26M [S1].
*   **LLFF ($N_p=2$):** PSNR 27.70, SSIM 0.870, LPIPS 0.155, I-time 4.55s, T-time 3.97h, #Run 96, #Params 1.26M [S1].
*   Hardware: Inference time measured on NVIDIA GeForce RTX 3080 Ti; Training time on NVIDIA A100-SXM4-80GB [S1].

**Strengths and Limitations:**
*   **Strengths:** Superior speed-quality trade-off compared to NeRF-few and NeRF-small (adjusted for comparable FLOPs) [S1]. MIMO-NeRF-self outperforms MIMO-NeRF-naive and MIMO-NeRF-distill in PSNR, SSIM, and LPIPS [S1]. Maintains original NeRF hyperparameters ($N_c, N_f, F$) [S1].
*   **Limitations:** Trade-off between speed and quality with increasing $N_p$ [S1]. Ambiguity in color and volume density for individual points if not regularized by 3D consistency loss [S1]. Asymmetric weights are disabled for $N_p=2$ ($R_1=R_2=1$) [S1].

**When to Use:**
Use when inference speed is critical and you want to maintain the representational capacity of the original NeRF architecture without reducing sample counts or network width [S1].

## Background
MIMO-NeRF was introduced in the paper "MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields" [S1]. It addresses the high computational cost of NeRF by replacing the SISO MLP with a MIMO MLP [S1]. The method was evaluated on the Blender and LLFF datasets [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
