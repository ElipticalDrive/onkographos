# HALO
Also called: HALO (Harnessing Low-frequency Neural Fields), Hi-NeRF, Input-space frequency regularization, LoNeRF, Low-frequency regularized NeRF

## Overview
HALO is a few-shot view synthesis method that prevents overfitting in neural radiance fields by regularizing a high-frequency NeRF (Hi-NeRF) using a low-frequency NeRF (Lo-NeRF) [S1, unofficial]. It leverages the distinct interpolation and extrapolation properties of low- and high-frequency representations to enforce geometric consistency without requiring extra semantic supervision [S1, unofficial]. Unlike sibling methods such as [DietNeRF](18409-training-configuration-and-optimization.md) which rely on CLIP-based semantic priors, HALO maintains better lighting consistency and operates purely within the input space [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A sparse set of $N$ training images with known camera poses (intrinsics and extrinsics).
*   **Outputs:** A trained Hi-NeRF MLP capable of rendering novel views.
*   **Implicit Outputs:** A Lo-NeRF MLP and a Ray-based field MLP (optional for dynamic scenes).

**Data Structures**
1.  **Lo-NeRF:** An MLP taking encoded position $\gamma_{Lo}(\mathbf{x})$ and direction $\gamma_{Lo}(\mathbf{d})$.
2.  **Ray-based Field:** An MLP taking ray origin $\mathbf{o}$ and direction $\mathbf{d}$. Input is 6D ($\mathbf{o} \in \mathbb{R}^3, \mathbf{d} \in \mathbb{R}^3$), but has 4 degrees of freedom as $\mathbf{o}$ lies on a predefined sphere and $\mathbf{d}$ is a unit vector [S1, unofficial].
3.  **Hi-NeRF:** An MLP taking encoded position $\gamma_{Hi}(\mathbf{x})$ and direction $\gamma_{Hi}(\mathbf{d})$.

**Positional Encoding**
The input coordinates are mapped to higher dimensions using hierarchical positional encoding $\gamma(\cdot)$ [S1, 13550-model-architecture-and-encoding.md, unofficial]. For a scalar input $p$, the encoding is:
$$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
where $L$ is the number of frequency bands [S1, 13550-model-architecture-and-encoding.md, unofficial].
*   **Lo-P.E.:** Uses a reduced frequency $L_{Lo}$ determined by the tuning criterion.
*   **Hi-P.E.:** Uses the default NeRF frequency $L_{Hi} = 10$ [S1, unofficial].

**Numbered Steps**

1.  **Tune Lo-NeRF Frequency:**
    *   Render $N$ paired images with a small baseline.
    *   Compute the averaged frequency domain difference $\sigma$:
        $$ \sigma = \frac{1}{N} \sum_{i=1}^N \| \mathcal{F}(I_{ia}) - \mathcal{F}(I_{ib}) \| $$
        where $\mathcal{F}()$ is the Fourier transformation [S1, unofficial].
    *   Reduce the frequency parameter $L$ until $\sigma$ reaches a predefined threshold (25) [S1, unofficial]. This $L$ defines Lo-NeRF.

2.  **Train Lo-NeRF:**
    *   Train the Lo-NeRF using standard volume rendering loss on the training views.
    *   Formula for rendered color $\hat{C}(\mathbf{r})$ along ray $\mathbf{r}$:
        $$ \hat{C}(\mathbf{r}) = \sum_{k=1}^K e^{-\sum_{k_0=1}^{k-1} \sigma_{k_0} \delta_{k_0}} (1 - e^{-\sigma_k \delta_k}) c_k $$
        where $\sigma_k = \sigma(\mathbf{r}(t_k))$, $c_k = c(\mathbf{r}(t_k))$, and $\delta_k = t_{k+1} - t_k$ [S1, unofficial].
    *   Loss:
        $$ L_{rec} = \sum_{\mathbf{r}} \| \hat{C}(\mathbf{r}) - C_{gt}(\mathbf{r}) \|_2^2 $$
        where $C_{gt}$ is ground truth RGB [S1, unofficial].

3.  **Train Ray-based Field (Static Scenes):**
    *   Freeze Lo-NeRF. Randomly initialize the Ray-based field $F_{ray}$.
    *   Compute depth $D(\mathbf{r}; F_{point})$ by sampling points and accumulating density from the frozen Lo-NeRF [S1, unofficial].
    *   Optimize $F_{ray}$ with loss:
        $$ L_{ray} = \sum_{\mathbf{r}=(\mathbf{o},\mathbf{d})} \| F_{ray}(\mathbf{o}, \mathbf{d}) - D(\mathbf{r}; F_{point}) \|_2^2 $$
        where $\mathbf{o}$ is the ray origin on a predefined sphere and $\mathbf{d}$ is the unit view direction [S1, unofficial].

4.  **Train Hi-NeRF:**
    *   Freeze Lo-NeRF and $F_{ray}$. Randomly initialize Hi-NeRF with $L=10$.
    *   Sample points along rays according to the depth predicted by $F_{ray}$ (following the DONeRF strategy) [S1, unofficial].
    *   Optimize Hi-NeRF with total loss:
        $$ L = L_{rec} + \lambda L_{empty} $$
    *   **Empty Space Loss:** Regularizes empty space where depth cannot be inferred from Lo-NeRF. Define accumulated occupancy for a ray $\mathbf{r}$ as:
        $$ acc(\mathbf{r}) = \prod_{k=1}^K e^{-\sum_{k_0=1}^{k-1} \sigma_{k_0} \delta_{k_0}} (1 - e^{-\sigma_k \delta_k}) $$
        The empty space loss is:
        $$ L_{empty} = \sum_{\mathbf{r}: acc_{Lo}(\mathbf{r}) < \tau} acc_{Hi}(\mathbf{r}) $$
        where $\tau = 0.01$ is the threshold for determining if space is empty [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

def positional_encoding(p, L):
    """
    Standard hierarchical positional encoding.
    p: scalar or tensor of shape (..., 1)
    L: number of frequency bands
    Returns: tensor of shape (..., 2*L)
    """
    # p is expected to be in range [-1, 1] or similar, scaled by pi
    # Formula: (sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^(L-1) pi p), cos(2^(L-1) pi p))
    freq_bands = 2.0 ** torch.arange(L, device=p.device).float()
    freq_bands = freq_bands * np.pi
    # Expand dimensions to broadcast
    freq_bands = freq_bands.view(1, -1)
    encoded = p * freq_bands
    sin_enc = torch.sin(encoded)
    cos_enc = torch.cos(encoded)
    return torch.cat([sin_enc, cos_enc], dim=-1)

def volume_rendering(sigma, color, t_starts, t_ends):
    """
    Standard volume rendering integral.
    sigma: density values (..., K)
    color: RGB values (..., K, 3)
    t_starts: start times of bins (..., K)
    t_ends: end times of bins (..., K)
    Returns: rendered color (..., 3)
    """
    dists = t_ends - t_starts
    alpha = 1.0 - torch.exp(-sigma * dists)
    transmittance = torch.cumprod(torch.exp(-sigma * dists), dim=-1)
    # Prepend 1.0 for the first segment
    transmittance = torch.cat([torch.ones_like(transmittance[..., :1]), transmittance[..., :-1]], dim=-1)
    weights = alpha * transmittance
    rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
    return rendered_color

class HALO_Trainer:
    def __init__(self, train_images, train_poses, num_views=8):
        self.train_images = train_images
        self.train_poses = train_poses
        self.num_views = num_views
        
        # Hyperparameters
        self.freq_threshold = 25
        self.empty_threshold = 0.01
        self.L_hi = 10
        self.lambda_empty = 1.0 # Balancing parameter for L_empty
        
        # Networks
        self.lo_nerf = None
        self.ray_field = None
        self.hi_nerf = None
        
        # Determine Lo-NeRF frequency
        self.L_lo = self.tune_frequency()

    def tune_frequency(self):
        """
        Stage 1: Determine optimal low frequency L.
        Renders paired images with small baseline and checks frequency domain difference.
        """
        # Implementation detail: Iterate L from low to high.
        # For each L, train a temporary Lo-NeRF.
        # Render N pairs with small baseline.
        # Compute sigma = mean(||F(I_ia) - F(I_ib)||).
        # Return L where sigma <= 25.
        # [G] Placeholder for the iterative search logic as exact search algorithm 
        # is not fully detailed in source, but the criterion is.
        L = 1
        while True:
            # Train temporary Lo-NeRF with L
            # Render pairs, compute freq diff
            # if diff <= 25: return L
            # L += 1
            pass 

    def train_lo_nerf(self):
        """
        Stage 2: Train Lo-NeRF with Lo-P.E.
        """
        # Initialize Lo-NeRF with L_lo
        # Optimize using L_rec on training views
        pass

    def train_ray_field(self):
        """
        Stage 3: Train Ray-based Field.
        Freezes Lo-NeRF.
        """
        # Initialize Ray Field
        # For each ray (o, d):
        #   Compute depth D from Lo-NeRF
        #   Optimize Ray Field to predict D
        pass

    def train_hi_nerf(self):
        """
        Stage 4: Train Hi-NeRF.
        Freezes Lo-NeRF and Ray Field.
        """
        # Initialize Hi-NeRF with L_hi=10
        # For each ray:
        #   Sample points using depth from Ray Field
        #   Compute L_rec and L_empty
        #   Optimize Hi-NeRF
        pass

    def render(self, ray_o, ray_d):
        """
        Render using Hi-NeRF.
        """
        # Sample points along ray
        # Encode with Hi-P.E.
        # Query Hi-NeRF for sigma and color
        # Volume render
        pass
```

**Edge Cases and Pitfalls**
*   **High-Frequency Encoding:** Using high-frequency positional encoding leads to poor interpolation results [S1, unofficial].
*   **Ray-based Blurriness:** Ray-based predictions may suffer from blurry edges, but this is acceptable for rough geometry regularization as Hi-NeRF fine-tunes details [S1, unofficial].
*   **Dynamic Scenes:** Using high-frequency encoding for the time axis introduces temporal flickering motion when dynamic sequence observations are sparse [S1, unofficial].
*   **Empty Space:** The empty space loss relies on the threshold $\tau=0.01$. If Lo-NeRF predicts non-zero occupancy in empty regions, Hi-NeRF may incorrectly fill them [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training involves three sequential stages. The ray-based field training adds overhead compared to vanilla NeRF. On dynamic scenes (TiNeuVox baseline), HALO takes 0.67 GPU/h compared to 0.17 GPU/h for the baseline, but yields significantly higher PSNR (25.687 vs 21.392) [S1, unofficial].
*   **Memory Complexity:** Requires storing parameters for Lo-NeRF, Ray-based Field, and Hi-NeRF. The Ray-based field is lightweight as it only predicts a scalar depth.
*   **Strengths:** Does not require extra semantic supervision (unlike DietNeRF); maintains lighting consistency; effective for few-shot settings (4-8 views); applicable to dynamic scenes via time dimension extension.
*   **Limitations:** Requires three-stage training; sensitive to the frequency tuning threshold; ray-based field may introduce blurriness in regularization.
*   **When to Use:** Few-shot view synthesis (static or dynamic) where semantic priors are unavailable or undesirable.
*   **When Not to Use:** Large-scale dense reconstruction (vanilla NeRF suffices); scenarios requiring real-time training with minimal overhead.

## Background
HALO (Harnessing Low-Frequency Neural Fields) was proposed in 2023 [S1, unofficial]. It addresses the overfitting problem in NeRFs under few-shot settings by exploiting the different interpolation and extrapolation properties of low- and high-frequency neural fields. It is related to [DietNeRF](18409-training-configuration-and-optimization.md) but avoids semantic supervision. It builds on [Position Encoding and Embedding](18459-position-encoding-and-embedding.md) principles.

## Sources
[S1] Harnessing Low-Frequency Neural Fields for Few-Shot View Synthesis, https://doi.org/10.48550/arxiv.2303.08370, fetched 2026-10-29 (unofficial)
