# SDF Volume Rendering

Also called: Volume SDF Rendering, Signed Distance Field Volume Rendering, Volume Rendering from SDF, SDF → density volume rendering

## Overview
SDF Volume Rendering is a technique that integrates color and density along rays sampled from an implicit Signed Distance Function (SDF) [S1, unofficial]. It converts SDF values to density using a Laplace distribution CDF and aggregates color using standard volume rendering weights [S1, unofficial]. Unlike standard volume rendering which uses raw density fields, this method leverages the geometric priors of the SDF to better recover underlying geometry [S6].

## How it works
**Inputs:**
- Ray bundle containing origins $\mathbf{o}$ and directions $\boldsymbol{\omega}$ [S7, unofficial].
- Implicit SDF function $f(\mathbf{x})$ returning signed distance and optionally color/features [S1, unofficial].
- Parameters $\alpha$ (scaling factor) and $\beta$ (scale parameter for Laplace distribution) [S1, unofficial].
- Small constant $\epsilon = 10^{-10}$ for numerical stability [S1, unofficial].

**Outputs:**
- Rendered color tensor of shape $(H, W, 3)$ [S7, unofficial].

**Data Structures:**
- `RayBundle`: Contains `origins` (shape $N, 3$), `directions` (shape $N, 3$), and `sample_points` (shape $N, D, 3$) [S7, unofficial].
- `deltas`: Tensor of shape $(N, D)$ representing segment lengths [S1, unofficial].

**Steps:**

1. **Ray Sampling:** Sample points along rays using a sampler (e.g., StratifiedRaysampler) to get `sample_points` and depth values [S1, unofficial].
2. **SDF Evaluation:** Query the implicit function for distance and color at sample points:
   $$ \text{distance}, \text{color} = \text{implicit\_fn}(\text{sample\_points}) $$
   [S1, unofficial]
3. **Delta Computation:** Compute segment lengths $\Delta t$ from depth values. For $D$ points, concatenate differences between consecutive depth values with a large constant ($10^{10}$) at the end to handle the last segment [S1, unofficial].
   $$ \delta_i = t_{i+1} - t_i \quad \text{for } i < D-1 $$
   $$ \delta_{D-1} = 10^{10} $$
   [S1, unofficial]
   This large value ensures that if a ray does not hit a surface within the sampled range, the transmittance decays sufficiently, preventing artifacts from the infinite tail of the ray [S1, unofficial].
4. **SDF to Density Conversion:** Convert signed distance $d$ to density $\rho$ using the Laplace distribution CDF [S1, unofficial]:
   $$ \rho = \alpha \cdot \text{CDF}_{\text{Laplace}(0, \beta)}(-d) $$
   [S1, unofficial]
   Explicitly:
   $$ \rho = \begin{cases} 0.5 \alpha e^{-d/\beta} & d > 0 \\ \alpha (1 - 0.5 e^{d/\beta}) & d \le 0 \end{cases} $$
   [S4, unofficial]
5. **Transmittance and Weight Calculation:** Initialize $T_0 = 1$. For $i$ from $1$ to $D$:
   $$ T_i = T_{i-1} \cdot \exp(-\rho_{i-1} \cdot \delta_{i-1} + \epsilon) $$
   [S1, unofficial]
   Weights are:
   $$ w_i = T_i \cdot (1 - \exp(-\rho_i \cdot \delta_i + \epsilon)) $$
   [S1, unofficial]
6. **Color Aggregation:** Aggregate color by summing the product of weights and per-sample colors:
   $$ C = \sum_{i=0}^{D-1} w_i \cdot \text{color}_i $$
   [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sdf_volume_rendering(ray_bundle, implicit_fn, alpha, beta, eps=1e-10):
    """
    ray_bundle: object with origins (N, 3), directions (N, 3), and sample_points (N, D, 3)
    implicit_fn: callable returning (distance, color) for points
    alpha: scaling factor
    beta: scale parameter for Laplace distribution
    eps: small constant for stability
    """
    origins = ray_bundle.origins  # (N, 3)
    directions = ray_bundle.directions  # (N, 3)
    sample_points = ray_bundle.sample_points  # (N, D, 3)
    
    # Compute depths along the ray
    depths = torch.sum((sample_points - origins[:, None, :]) * directions[:, None, :], dim=-1)  # (N, D)
    
    # Compute segment lengths (deltas)
    # Concatenate differences with a large constant for the last segment
    deltas = torch.cat((depths[:, 1:] - depths[:, :-1], 1e10 * torch.ones_like(depths[:, :1])), dim=-1)  # (N, D)
    
    # Evaluate SDF and color
    distance, color = implicit_fn(sample_points)  # distance: (N, D), color: (N, D, 3)
    
    # Convert SDF to density using Laplace CDF
    # CDF_Laplace(0, beta)(-d) = 0.5*exp(-d/beta) if d>0, else 1 - 0.5*exp(d/beta)
    density = torch.where(
        distance > 0,
        0.5 * torch.exp(-distance / beta),
        1.0 - 0.5 * torch.exp(distance / beta),
    ) * alpha  # (N, D)
    
    # Compute transmittance T and weights w
    # T[0] = 1. T[i] = T[i-1] * exp(-density[i-1] * delta[i-1] + eps)
    # w[i] = T[i] * (1 - exp(-density[i] * delta[i] + eps))
    
    # Vectorized computation
    # exponent for transmittance decay
    exponent = -density * deltas + eps  # (N, D)
    
    # Cumulative sum of exponents for T
    # T[i] = exp(sum_{j=0}^{i-1} exponent[j])
    # cumsum_exponent[i] = sum_{j=0}^{i} exponent[j]
    # So T[i] = exp(cumsum_exponent[i-1]) for i>0, T[0] = 1
    cumsum_exponent = torch.cumsum(exponent, dim=1)  # (N, D)
    log_T = torch.cat([torch.zeros_like(cumsum_exponent[:, :1]), cumsum_exponent[:, :-1]], dim=1)  # (N, D)
    T = torch.exp(log_T)  # (N, D)
    
    # Weights
    weights = T * (1.0 - torch.exp(exponent))  # (N, D)
    
    # Aggregate color
    color = torch.sum(weights[:, :, None] * color, dim=1)  # (N, 3)
    
    return color
```

## Subtopics

### [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md)
VolSDF Volume Rendering performs volume integration using Signed Distance Functions (SDFs) as the underlying geometric representation instead of raw density fields. It addresses optimization instability by converting SDF values into volumetric densities using learnable scaling and shrinking coefficients before applying standard alpha compositing. Unlike [Volume SDF Rendering](1344-volume-sdf-rendering.md), VolSDF explicitly learns these coefficients to ensure alpha-invariance, meaning the magnitude of the volume density function is globally tied to its sharpness.

### NeUDF
NeUDF is a neural rendering framework that reconstructs surfaces with arbitrary topologies, including open surfaces, from multi-view 2D image supervision by leveraging the Unsigned Distance Function (UDF) as the implicit surface representation. Unlike its sibling [Volume SDF Rendering](1344-volume-sdf-rendering.md), which is limited to closed (watertight) surfaces, NeUDF resolves the ambiguity of surface orientation in open structures by introducing a specialized "Udensity" probability density function and a normal regularization strategy. It differs from naive extensions of SDF-based rendering by avoiding spurious surfaces in void regions through an unbiased, occlusion-aware weighting paradigm tailored for UDFs.

### [SRDF-based Volume Rendering](15314-srdf-based-volume-rendering.md)
SRDF-based Volume Rendering is a neural scene reconstruction technique that models volume density using a Ray-Specific Signed Distance Function (SRDF) rather than a global Signed Distance Function (SDF). It solves the problem of false local maxima in volume density caused by surfaces not intersecting the camera ray, which leads to inaccurate depth and noisy surface reconstruction in multi-object indoor scenes. Unlike standard SDF-based methods where density is derived from a global field, SRDF computes the shortest distance to the surface *along the sampled ray direction*, ensuring that density peaks only around the actual surface intersection. This method is distinct from [NeUDF](3821-neudf.md) by explicitly decoupling the ray-specific density modeling from the global surface representation.

### [Learnable Alpha/Beta SDF Density Modulation](18988-learnable-alpha-beta-sdf-density-modulation.md)
This method modulates volume density in a Signed Distance Function (SDF) representation using two separate learnable parameters, $\alpha$ and $\beta$, rather than tying them equal as in prior work. It solves the convergence instability issues found in methods that use a single tied parameter by allowing independent optimization of the density scale and shape. It differs from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) by decoupling the $\alpha$ and $\beta$ parameters.

### Chunked VolSDF Rendering
Chunked VolSDF Rendering processes rays in chunks to manage memory usage during training [S7, unofficial]. It gets the total number of rays $N$ from `ray_bundle.origins.shape[0]`, initializes an empty list `chunk_colors`, and loops over chunks [S7, unofficial]. For each chunk, it slices the `ray_bundle`, calls the parent forward method to get `cur_color`, appends it to the list, and finally concatenates all chunk colors along dimension 0 [S7, unofficial]. The parameter `chunk_size` defines the maximum number of rays processed in a single forward pass [S7, unofficial].

### NeuS Volume Rendering Transformation
NeuS Volume Rendering Transformation is a method that transforms Signed Distance Function (SDF) values into volumetric densities for alpha compositing, ensuring the CDF of volume rendering matches a scaled, flipped sigmoid shape [S9]. The transmittance $T(t)$ is related to the SDF $S(t)$ by $1 - T(t) = \text{sigmoid}(s \cdot -SDF(t))$, where $s$ is a global scaling coefficient [S9]. The weighting function $w(t)$ (derivative of CDF) has local maxima at SDF value 0 [S9]. NeuS ties the magnitude of volume density globally to its sharpness via the sigmoid parameter $s$, which is an alpha-invariant formulation by default but restrictive [S9].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays [G].
- **Memory Complexity:** Linear with respect to the number of samples and rays, as all distances, colors, and transmittances must be stored for backpropagation [G]. Chunking can reduce peak memory usage [S7, unofficial].
- **Strengths:** Better recovery of underlying geometry compared to direct density encoding [S6].
- **Limitations:** Requires careful tuning of $\alpha$ and $\beta$ or learning them [G].
- **When to use:** When geometric accuracy is prioritized over appearance flexibility, or when training from multi-view images where surface reconstruction is the goal [S7, unofficial].
- **When not to use:** For open surfaces (use NeUDF) or when ray-specific density modeling is needed to avoid false maxima (use SRDF) [G].

## Background
Originating from the need to improve geometric reconstruction in neural rendering, SDF Volume Rendering was introduced as an alternative to NeRF's direct density encoding [S5, S6]. VolSDF [S2, unofficial] and NeuS [S9] demonstrated that modeling density as a transformed function of an implicit SDF enables better recovery of underlying geometry [S6]. The method maps the SDF to a volume density function, assigning a value of 1 to points on the zero-level set and 0 to points far from this set [S5]. It is widely used in neural surface reconstruction from multi-view images [S7, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
[S6] Sur$$^{2}$$f: A Hybrid Representation for High-Quality and Efficient Surface Reconstruction from Multi-view Images, https://doi.org/10.1007/978-3-031-73027-6_1, fetched 2026-10-26
[S7] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S8] dehezhang2/RISE-SDF, https://github.com/dehezhang2/RISE-SDF, fetched 2026-10-27 (unofficial)
[S9] Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields, https://www.youtube.com/watch?v=YQhBEXC464w, fetched 2026-10-31
