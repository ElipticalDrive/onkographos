# NeRF-W

Also called: Neural Radiance Fields for Unconstrained Photo Collections

## Overview
NeRF-W extends the standard NeRF framework to handle unconstrained, "in-the-wild" photo collections by relaxing the assumption of a static, consistently lit scene [S1]. It disentangles the scene into shared static geometry and image-dependent transient elements (such as pedestrians) and photometric variations (such as lighting and auto-exposure) using learned appearance embeddings and a secondary transient radiance field with uncertainty modeling [S1]. Unlike NeRF-A, which only models appearance, or NeRF-U, which only models uncertainty, NeRF-W combines both to reconstruct high-frequency details and occluders without artifacts [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ images $\{I_i\}_{i=1}^N$ with known camera poses (intrinsic and extrinsic parameters estimated via COLMAP [S1]), a 3D position $\mathbf{t}$, a viewing direction $\mathbf{d}$, and an image index $i$.
*   **Outputs:** For a given ray $\mathbf{r}$ passing through pixel $(u,v)$ in image $I_i$, the method outputs a rendered color $\hat{C}_i(\mathbf{r})$ and an uncertainty variance $\beta_i(\mathbf{r})^2$. At test time, it outputs only the static color $\hat{C}_i^{static}(\mathbf{r})$.

**Data Structures:**
1.  **Static Radiance Field:** Defined by MLP $\theta_1$.
    *   Input: Position $\mathbf{t}$, positional encoding $\gamma(\mathbf{t})$.
    *   Output: Latent feature vector $\mathbf{z}(\mathbf{t})$ and static volume density $\sigma(\mathbf{t})$.
2.  **Appearance-Conditioned Color Field:** Defined by MLP $\theta_2$.
    *   Input: Latent feature $\mathbf{z}(\mathbf{t})$, viewing direction $\mathbf{d}$ (encoded as $\gamma(\mathbf{d})$), and appearance embedding $\mathbf{l}_i^{(a)}$.
    *   Output: Static color $\mathbf{c}(\mathbf{t}, \mathbf{d}, \mathbf{l}_i^{(a)})$.
3.  **Transient Radiance Field:** Defined by MLP $\theta_3$.
    *   Input: Position $\mathbf{t}$, transient embedding $\mathbf{l}_i^{(\tau)}$.
    *   Output: Transient volume density $\sigma_i^{(\tau)}(\mathbf{t})$ and transient color $\mathbf{c}_i^{(\tau)}(\mathbf{t})$.
4.  **Uncertainty Field:** Defined by MLP $\theta_4$.
    *   Input: Position $\mathbf{t}$, viewing direction $\mathbf{d}$, appearance embedding $\mathbf{l}_i^{(a)}$.
    *   Output: Variance $\beta_i(\mathbf{t}, \mathbf{d}, \mathbf{l}_i^{(a)})^2$.
5.  **Embeddings:**
    *   $\mathbf{l}_i^{(a)} \in \mathbb{R}^{n^{(a)}}$: Appearance embedding for image $i$.
    *   $\mathbf{l}_i^{(\tau)} \in \mathbb{R}^{n^{(\tau)}}$: Transient embedding for image $i$.

**Numbered Steps:**

1.  **Initialization:** Initialize MLPs $\theta_1, \theta_2, \theta_3, \theta_4$ with random weights. Initialize appearance embeddings $\{\mathbf{l}_i^{(a)}\}_{i=1}^N$ and transient embeddings $\{\mathbf{l}_i^{(\tau)}\}_{i=1}^N$ to zero or small random values.
2.  **Ray Sampling:** For each training image $I_i$, sample rays $\mathbf{r}$ through each pixel. Sample $K$ points $\{t_k\}_{k=1}^K$ along each ray using stratified sampling or importance sampling based on density.
3.  **Static Feature Extraction:** For each point $t_k$ on ray $\mathbf{r}$:
    *   Compute positional encoding $\gamma(\mathbf{t}_k)$.
    *   Pass $\gamma(\mathbf{t}_k)$ through $\theta_1$ to get $\mathbf{z}(\mathbf{t}_k)$ and $\sigma(\mathbf{t}_k)$.
    *   Note: $\sigma(\mathbf{t}_k)$ is shared across all images.
4.  **Appearance-Conditioned Color:**
    *   Compute viewing direction encoding $\gamma(\mathbf{d})$.
    *   Concatenate $\mathbf{z}(\mathbf{t}_k)$, $\gamma(\mathbf{d})$, and $\mathbf{l}_i^{(a)}$.
    *   Pass through $\theta_2$ to get static color $\mathbf{c}(\mathbf{t}_k)$.
5.  **Transient Component:**
    *   Concatenate $\mathbf{t}_k$ (or its encoding) and $\mathbf{l}_i^{(\tau)}$.
    *   Pass through $\theta_3$ to get transient density $\sigma_i^{(\tau)}(\mathbf{t}_k)$ and transient color $\mathbf{c}_i^{(\tau)}(\mathbf{t}_k)$.
    *   Apply ReLU to $\sigma_i^{(\tau)}(\mathbf{t}_k)$ and Sigmoid to $\mathbf{c}_i^{(\tau)}(\mathbf{t}_k)$ [S1].
6.  **Uncertainty Estimation:**
    *   Concatenate inputs for $\theta_4$ (position, direction, appearance embedding).
    *   Pass through $\theta_4$ to get raw variance output.
    *   Apply Softplus activation shifted by $\beta_{min}$ to ensure $\beta_i(\mathbf{t}_k)^2 \ge \beta_{min} > 0$ [S1].
7.  **Volumetric Rendering (Composite):**
    *   Compute transmittance $T_i(t_k)$ accounting for both static and transient densities:
        $$ T_i(t_k) = \exp \left( -\sum_{k'=1}^{k-1} \left( \sigma(\mathbf{t}_{k'}) + \sigma_i^{(\tau)}(\mathbf{t}_{k'}) \right) \delta_{k'} \right) $$
    *   Compute opacity for each component:
        $$ \alpha(\mathbf{t}_k) = 1 - \exp(-\sigma(\mathbf{t}_k)\delta_k) $$
        $$ \alpha_i^{(\tau)}(\mathbf{t}_k) = 1 - \exp(-\sigma_i^{(\tau)}(\mathbf{t}_k)\delta_k) $$
    *   Render composite color $\hat{C}_i(\mathbf{r})$:
        $$ \hat{C}_i(\mathbf{r}) = \sum_{k=1}^K T_i(t_k) \left[ \alpha(\mathbf{t}_k)\mathbf{c}(\mathbf{t}_k) + \alpha_i^{(\tau)}(\mathbf{t}_k)\mathbf{c}_i^{(\tau)}(\mathbf{t}_k) \right] $$
8.  **Coarse Rendering (Optional but used in training):**
    *   Use a coarse model (typically just the static head + appearance embeddings) to render $\hat{C}_i^c(\mathbf{r})$ for the coarse loss term [S1].
9.  **Loss Calculation:**
    *   Compute the loss for ray $\mathbf{r}$ in image $i$:
        $$ L_i(\mathbf{r}) = \frac{(\mathbf{C}_i(\mathbf{r}) - \hat{C}_i(\mathbf{r}))^2}{2\beta_i(\mathbf{r})^2} + \frac{\log \beta_i(\mathbf{r})^2}{2} + \frac{\lambda_u}{2K} \sum_{k=1}^K \sigma_i^{(\tau)}(\mathbf{t}_k) $$
        where $\mathbf{C}_i(\mathbf{r})$ is the ground truth pixel color [S1].
    *   Add coarse loss term if using coarse-fine training:
        $$ L_{total} = \sum_{\mathbf{r}} \left( L_i(\mathbf{r}) + \frac{1}{2} \| \mathbf{C}_i(\mathbf{r}) - \hat{C}_i^c(\mathbf{r}) \|^2 \right) $$
10. **Optimization:**
    *   Minimize $L_{total}$ using Adam optimizer with respect to $\theta_1, \theta_2, \theta_3, \theta_4$, $\{\mathbf{l}_i^{(a)}\}$, and $\{\mathbf{l}_i^{(\tau)}\}$ [S1].
11. **Test-Time Rendering:**
    *   Omit transient head and uncertainty fields.
    *   Render only static components:
        $$ \hat{C}_i^{static}(\mathbf{r}) = \sum_{k=1}^K T_i^{static}(t_k) \alpha(\mathbf{t}_k)\mathbf{c}(\mathbf{t}_k) $$
        where $T_i^{static}$ uses only $\sigma(\mathbf{t}_k)$ [S1].
    *   If evaluating on test images, optimize a single appearance embedding $\mathbf{l}^{(a)}$ using the left half of the image to avoid leakage, then render the right half [S1].

**Parameters:**
*   $\beta_{min}$: Hyperparameter $>0$ for softplus shift [S1].
*   $\lambda_u$: Weight for L1 regularization on transient density [S1].
*   $n^{(a)}$: Dimensionality of appearance embedding [S1].
*   $n^{(\tau)}$: Dimensionality of transient embedding [S1].
*   Batch size: 2048 [S1].
*   Training steps: 300,000 [S1].
*   Optimizer: Adam ($\beta_1=0.9, \beta_2=0.999$) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
import torch.nn.functional as F

class NeRF_W_Model(nn.Module):
    def __init__(self, n_a, n_tau, hidden_dim=256):
        super().__init__()
        # Static Geometry Network
        self.static_net = nn.Sequential(
            nn.Linear(9, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim + 1) # z, sigma
        )
        
        # Appearance-Conditioned Color Network
        self.color_net = nn.Sequential(
            nn.Linear(hidden_dim + 9 + n_a, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, 3) # color
        )
        
        # Transient Density and Color Network
        self.transient_net = nn.Sequential(
            nn.Linear(9 + n_tau, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, 4) # sigma, color
        )
        
        # Uncertainty Network
        self.uncertainty_net = nn.Sequential(
            nn.Linear(9 + 9 + n_a, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, 1) # variance
        )
        
        self.beta_min = 1e-4 # Hyperparameter

    def positional_encoding(self, x, min_deg=0, max_deg=10):
        # Standard NeRF positional encoding
        scales = torch.tensor([2**i for i in range(min_deg, max_deg)])
        xb = torch.reshape((x * scales).float(), tuple(list(x.shape)[:-1]) + (-1,))
        freq_bands = torch.cat([torch.sin(xb), torch.cos(xb)], dim=-1)
        return freq_bands

    def forward(self, positions, directions, appearance_embed, transient_embed, beta_min=None):
        if beta_min is None:
            beta_min = self.beta_min
            
        # 1. Static Features
        pos_enc = self.positional_encoding(positions)
        static_out = self.static_net(pos_enc)
        z = static_out[:, :-1]
        sigma_static = static_out[:, -1:]
        
        # 2. Appearance-Conditioned Color
        dir_enc = self.positional_encoding(directions)
        color_input = torch.cat([z, dir_enc, appearance_embed], dim=-1)
        color = torch.sigmoid(self.color_net(color_input))
        
        # 3. Transient Components
        trans_input = torch.cat([pos_enc, transient_embed], dim=-1)
        trans_out = self.transient_net(trans_input)
        sigma_trans = F.relu(trans_out[:, :1])
        color_trans = torch.sigmoid(trans_out[:, 1:])
        
        # 4. Uncertainty
        unc_input = torch.cat([pos_enc, dir_enc, appearance_embed], dim=-1)
        raw_var = self.uncertainty_net(unc_input)
        variance = F.softplus(raw_var) + beta_min
        
        return {
            'sigma_static': sigma_static,
            'color_static': color,
            'sigma_trans': sigma_trans,
            'color_trans': color_trans,
            'variance': variance
        }

def render_ray(points, weights_static, weights_trans, colors_static, colors_trans, variance):
    """
    Renders a single ray given sampled points and network outputs.
    points: (K, 3)
    weights_static: (K,) - alpha * T
    weights_trans: (K,) - alpha_trans * T
    colors_static: (K, 3)
    colors_trans: (K, 3)
    variance: (K,)
    """
    # Composite Color
    comp_color = torch.sum(weights_static.unsqueeze(-1) * colors_static + 
                            weights_trans.unsqueeze(-1) * colors_trans, dim=0)
    
    # Mean Variance for Loss (simplified, usually per-point or ray-wise)
    mean_var = torch.mean(variance)
    
    return comp_color, mean_var

def compute_loss(gt_color, pred_color, variance, sigma_trans, lambda_u):
    """
    Computes the NeRF-W loss for a batch of rays.
    """
    # Negative log likelihood term
    diff = gt_color - pred_color
    nll = (diff ** 2) / (2 * variance) + torch.log(variance) / 2
    
    # L1 regularization on transient density
    trans_reg = (lambda_u / 2) * torch.mean(sigma_trans)
    
    return torch.mean(nll + trans_reg)

def train_step(model, batch, optimizer, lambda_u, beta_min):
    optimizer.zero_grad()
    
    positions = batch['positions'] # (B, K, 3)
    directions = batch['directions'] # (B, K, 3)
    appearance_embed = batch['appearance_embed'] # (B, n_a)
    transient_embed = batch['transient_embed'] # (B, n_tau)
    gt_colors = batch['gt_colors'] # (B, 3)
    
    # Forward pass
    outputs = model(positions, directions, appearance_embed, transient_embed, beta_min)
    
    # Compute weights for rendering (simplified volumetric rendering logic)
    # In practice, this involves computing T_k and alpha_k for each point
    # Here we assume a helper function computes the rendered color and transmittance weights
    # For the purpose of this article, we assume the loss is computed on the final rendered pixel
    
    # Note: The actual rendering involves a loop or vectorized reduction over K points
    # to compute T_k and alpha_k.
    
    # Render composite color
    # This is a placeholder for the actual volumetric integration logic
    pred_color, variance = render_ray(
        positions, 
        outputs['weights_static'], 
        outputs['weights_trans'], 
        outputs['color_static'], 
        outputs['color_trans'], 
        outputs['variance']
    )
    
    # Compute Loss
    loss = compute_loss(gt_colors, pred_color, variance, outputs['sigma_trans'], lambda_u)
    
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

**Edge Cases and Pitfalls:**
*   **Information Leakage:** When evaluating on test images, optimizing the appearance embedding on the entire image can lead to overfitting to the test data. The method specifies using only the left half of the image for optimization and the right half for metric reporting [S1].
*   **Transient Density Explosion:** Without regularization, the model might use transient density to explain static variations. The L1 regularization term $\lambda_u$ prevents this [S1].
*   **Variance Collapse:** The variance $\beta_i(\mathbf{r})^2$ could collapse to infinity to minimize the NLL term. The softplus activation with $\beta_{min}$ prevents this trivial minimum [S1].
*   **Static Geometry Sharing:** The static opacity $\sigma(\mathbf{t})$ is generated *before* conditioning on the appearance embedding. This ensures that the 3D geometry is shared across all images, while only the radiance varies [S1].

## Operating characteristics
*   **Time Complexity:** Training requires 300,000 steps with a batch size of 2048, optimized on 8 GPUs [S1]. The addition of transient and uncertainty heads increases computational cost compared to standard NeRF.
*   **Memory Complexity:** Requires storing appearance embeddings $\mathbf{l}_i^{(a)}$ and transient embeddings $\mathbf{l}_i^{(\tau)}$ for each image, in addition to the MLP weights.
*   **Performance:** NeRF-W improves over the previous state of the art (NRW) by an average margin of 4.4dB in PSNR across 6 scenes and achieves up to 40% improvements in MS-SSIM [S1].
*   **Strengths:** Handles unconstrained photo collections with varying lighting, weather, and transient occluders (pedestrians). Produces temporally consistent novel view renderings [S1].
*   **Limitations:** Lacks a perceptual loss, so it may not match LPIPS scores of methods that include them, though it remains competitive [S1]. Requires COLMAP for camera pose estimation [S1].

## Background
NeRF-W was introduced to address the limitations of NeRF in uncontrolled settings, such as variable illumination and transient occluders, which are common in internet photo collections [S1]. It builds on the framework of Generative Latent Optimization (GLO) for appearance modeling and Bayesian learning for uncertainty estimation [S1]. The method was applied to unconstrained internet photo collections of cultural landmarks including the Trevi Fountain, Sacre Coeur, Brandenburg Gate, Taj Mahal, Prague Old Town Square, and Hagia Sophia [S1].

## Sources
[S1] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
