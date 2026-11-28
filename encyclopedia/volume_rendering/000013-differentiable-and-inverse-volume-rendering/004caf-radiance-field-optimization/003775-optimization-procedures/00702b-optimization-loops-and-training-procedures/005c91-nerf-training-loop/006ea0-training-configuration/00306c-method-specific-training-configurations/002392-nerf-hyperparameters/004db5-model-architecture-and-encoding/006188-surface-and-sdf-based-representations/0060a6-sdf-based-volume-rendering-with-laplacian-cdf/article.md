# SDF-based Volume Rendering with Laplacian CDF

Also called: Laplacian opacity transformation, VolSDF variant

## Overview
This method renders geometry by predicting a Signed Distance Function (SDF) value $s$ and a variance parameter $\beta$ from a neural network, then transforming $s$ into volume opacity $\sigma$ using the Cumulative Distribution Function (CDF) of a Laplacian distribution [S1]. It builds on [Surface and SDF-Based Representations](18411-surface-and-sdf-based-representations.md) by providing a differentiable mechanism to extract tight surfaces while enabling efficient inverse CDF sampling [S1]. Unlike [I2-SDF](18170-i2-sdf.md), which uses a piecewise exponential density definition, this approach explicitly models the opacity via the Laplacian CDF, allowing $\beta$ to vary spatially to control surface tightness locally [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{v}$ [S1].
*   **Output:** Opacity $\sigma \in [0, 1]$, radiance color $\mathbf{c}$, and geometry features $\mathbf{f}_{geo}$ [S1].

**Data Structures**
*   **Triplane Features ($\mathbf{T}_x$):** Three aligned 2D feature grids (e.g., resolution $512 \times 512$, feature size 32) [S1]. For a point $\mathbf{x}$, features are gathered by projecting $\mathbf{x}$ onto the three axis-aligned planes and taking the Hadamard product of the resulting vectors [S1].
*   **MLP Architecture:**
    *   `MLPSDF`: Takes positional encoding of $\mathbf{x}$ and triplane features $\mathbf{T}_x$. Outputs scalar SDF $s$, scalar variance $\beta$, and geometry feature vector $\mathbf{f}_{geo}$ [S1].
    *   `MLP_c`: Takes positional encoding of viewing direction $\mathbf{v}$ and geometry features $\mathbf{f}_{geo}$. Outputs radiance $\mathbf{c}$ [S1].

**Parameters**
*   $\beta$: The variance of the Laplacian distribution, governing the "tightness" of the surface representation [S1]. It is a learnable output of `MLPSDF` and varies spatially across the volume [S1].
*   $N$: Number of samples per ray [S1].

**Step-by-Step Method**

1.  **Positional Encoding:** Compute the positional encoding $\text{PosEnc}(\mathbf{x})$ for the input point $\mathbf{x}$ [S1].
2.  **Feature Gathering:** Retrieve triplane features $\mathbf{T}_x$ by projecting $\mathbf{x}$ to the three axis-aligned planes and computing the Hadamard product of the three resultant vectors [S1].
3.  **SDF Prediction:** Pass $(\text{PosEnc}(\mathbf{x}), \mathbf{T}_x)$ to `MLPSDF` to obtain the SDF value $s$, variance $\beta$, and geometry features $\mathbf{f}_{geo}$ [S1]:
    $$
    (s, \beta, \mathbf{f}_{geo}) = \text{MLPSDF}(\text{PosEnc}(\mathbf{x}), \mathbf{T}_x)
    $$
    [G] Initialize the ReLU activations in `MLPSDF` as in SAL to ensure an approximately spherical SDF in early training.
4.  **Opacity Transformation:** Transform $s$ and $\beta$ into opacity $\sigma$ using the Laplacian CDF [S1]:
    $$
    \sigma = \begin{cases} 
    \frac{1}{2} \exp\left(\frac{s}{\beta}\right) & \text{if } s \le 0 \\
    1 - \frac{1}{2} \exp\left(-\frac{s}{\beta}\right) & \text{if } s > 0 
    \end{cases}
    $$
    where $s$ is the SDF value and $\beta$ is the variance [S1].
5.  **Radiance Prediction:** Compute radiance $\mathbf{c}$ using the geometry features and viewing direction [S1]:
    $$
    \mathbf{c} = \text{MLP}_c(\text{PosEnc}(\mathbf{v}), \mathbf{f}_{geo})
    $$
6.  **Inverse CDF Sampling:** To sample points near the surface, generate random numbers $u_i \in [0, 1]$ and transform them to sample distances $t_i$ using the inverse CDF $\Phi^{-1}$ [S1]:
    $$
    t_i = \Phi^{-1}(u_i)
    $$
    where $\Phi$ is the CDF derived from the Laplacian distribution defined by $s$ and $\beta$, and $u_i$ is sorted in increasing order [S1].
7.  **Rendering:** Approximate the volumetric rendering integral for a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ by summing the contributions of samples [S1]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i
    $$
    where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance, $\alpha_i$ is the opacity $\sigma$ at sample $i$, and $\mathbf{c}_i$ is the radiance [S1].

**Stratification**
During training, stratify the random numbers $u_i$ by dividing the unit interval into bins and taking a sample from each bin to lower rendering variance, especially at low sample counts $N$ [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def positional_encoding(x, min_deg=0, max_deg=10, power=2):
    """Standard NeRF positional encoding [S1]."""
    scales = torch.tensor([2**i for i in range(min_deg, max_deg + 1)], dtype=x.dtype, device=x.device)
    scaled_x = (math.pi * x.unsqueeze(-1)) * scales
    return torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)

def gather_triplane_features(triplanes, x):
    """Project x to three axis-aligned planes and take Hadamard product [S1]."""
    features = []
    for plane in triplanes:
        # Extract 2D coordinates for each axis-aligned plane
        coords = torch.stack([x[:2], x[1:], x[2:]], dim=1)
        # Bilinear interpolation at projected coordinates
        feat = torch.nn.functional.grid_sample(plane.unsqueeze(0), coords.unsqueeze(0), align_corners=False).squeeze()
        features.append(feat)
    return torch.prod(torch.stack(features), dim=0)

def laplacian_cdf(s, beta):
    """Transform SDF s and variance beta into opacity sigma [S1]."""
    beta = torch.clamp(beta, min=1e-6)
    return torch.where(s <= 0, 0.5 * torch.exp(s / beta), 1.0 - 0.5 * torch.exp(-s / beta))

def inverse_laplacian_cdf(u, beta):
    """Compute sample distance t from random u using inverse Laplacian CDF [S1]."""
    # Inverse of the CDF: if u <= 0.5, t = beta * ln(2u); else t = -beta * ln(2(1-u))
    return torch.where(u <= 0.5, beta * torch.log(2 * u), -beta * torch.log(2 * (1 - u)))

def render_ray(ray_o, ray_d, triplanes, mlpsdf, mlpc, num_samples):
    """Render a single ray using SDF-based volume rendering with Laplacian CDF."""
    # 1. Stratified sampling: divide unit interval into bins and sample from each [S1]
    indices = torch.arange(num_samples)
    u = (indices + torch.rand(num_samples)) / num_samples
    u = torch.sort(u).values  # Ensure increasing order [S1]

    # 2. Initialize sample distances and points
    t_samples = torch.zeros(num_samples)
    points = ray_o.unsqueeze(0) + t_samples.unsqueeze(1) * ray_d.unsqueeze(0)

    # 3. Query network at sample points
    pos_enc_x = positional_encoding(points)
    tri_feat = gather_triplane_features(triplanes, points)
    s, beta, f_geo = mlpsdf(pos_enc_x, tri_feat)

    # 4. Transform to opacity and predict radiance [S1]
    sigma = laplacian_cdf(s, beta)
    pos_enc_v = positional_encoding(ray_d)
    c = mlpc(pos_enc_v, f_geo)

    # 5. Inverse CDF sampling to refine distances near surface [S1]
    t_samples = inverse_laplacian_cdf(u, beta)

    # 6. Volume rendering integral [S1]
    alpha = sigma
    transmittance = torch.ones_like(alpha)
    transmittance[1:] = torch.cumprod(1.0 - alpha[:-1], dim=0)
    color = torch.sum(transmittance.unsqueeze(-1) * alpha.unsqueeze(-1) * c, dim=0)
    return color, sigma, s, beta
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N$ and the cost of the MLP evaluations [S1]. The use of inverse CDF sampling concentrates samples near the surface, potentially reducing the number of samples needed for high-fidelity geometry compared to uniform sampling [S1].
*   **Memory Complexity:** Requires storing the triplane features (e.g., $3 \times 512 \times 512 \times 32$ floats) and the MLP weights [S1].
*   **Strengths:** Tight surface extraction due to the SDF representation; efficient sampling near the surface via inverse CDF; lower rendering variance with stratification [S1].
*   **Limitations:** Requires careful initialization (e.g., SAL initialization) to avoid degenerate SDFs early in training [S1]. The spatially varying $\beta$ requires regularization to prevent instability [S1].
*   **When to use:** When high-fidelity geometry extraction is prioritized over texture, and when training a 3D GAN or differentiable renderer [S1].
*   **When not to use:** When computational resources are extremely limited and simple uniform sampling suffices, or when the scene does not have well-defined surfaces [S1].

## Background
This method is described in "What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs" [S1]. It builds upon VolSDF and StyleSDF, improving upon NeRF and EG3D by incorporating stratified sampling and a triplane-based SDF representation [S1]. The use of the Laplacian CDF for opacity transformation was previously explored in VolSDF, but this method extends it by allowing $\beta$ to vary spatially and integrating it into a GAN framework with triplane conditioning [S1].

## Sources
[S1] What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs, https://doi.org/10.1109/cvpr52733.2024.02148, fetched 2026-10-26
