# 2D Gaussian Evaluation

Also called: 2D Gaussian PDF exponent

## Overview
Computes the exponent term of the 2D Gaussian probability density function at given 2D points, used for rasterization or shading [S1, unofficial]. It differs from its parent [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md) by returning only the unnormalized exponent value $P$ rather than the full normalized density, allowing the caller to apply normalization factors or opacity scaling separately.

## How it works
**Inputs:**
- `points_2D`: Tensor of shape $(N, 2)$ containing the 2D query coordinates $\mathbf{x}$.
- `means_2D`: Tensor of shape $(N, 2)$ containing the 2D Gaussian means $\boldsymbol{\mu}$.
- `cov_2D`: Tensor of shape $(N, 2, 2)$ containing the 2D covariance matrices $C_{2D}$.

**Outputs:**
- `power`: Tensor of shape $(N,)$ containing the exponent values $P$.

**Data Structures:**
- Standard floating-point tensors (typically `float32`). No specific bit-packing is required for the inputs; the covariance matrix is stored as a standard $2 \times 2$ matrix.

**Steps:**
1. Compute the difference vector between query points and 2D means:
   $$ \mathbf{x}_m = \mathbf{x} - \boldsymbol{\mu} $$
   In code, this is `xm = points_2D - means_2D`.

2. Reshape the difference vector for batched matrix multiplication. The vector $\mathbf{x}_m$ is treated as a column vector in the quadratic form.
   - Create a column vector representation: $\mathbf{x}_m^{col}$ of shape $(N, 2, 1)$.
   - Create a row vector representation: $\mathbf{x}_m^{row}$ of shape $(N, 1, 2)$.
   In code, this is `xm_col = xm.unsqueeze(-1)` and `xm_row = xm.unsqueeze(-2)`.

3. Compute the inverse of the 2D covariance matrix for each Gaussian.
   $$ C_{2D}^{-1} = C_{2D}^{-1} $$
   In code, this is `cov_2D_inv = cov_2D.inverse()`. The result is unsqueezed to shape $(N, 1, 2, 2)$ for broadcasting against the batch dimension.

4. Compute the quadratic form and scale by $-0.5$:
   $$ P = -\frac{1}{2} (\mathbf{x}_m^{row} \cdot C_{2D}^{-1} \cdot \mathbf{x}_m^{col}) $$
   The matrix multiplication results in a tensor of shape $(N, 1, 1)$. Squeeze the last two dimensions to obtain the final scalar exponent per point.
   In code: `power = -0.5 * (xm_row @ cov_2D_inv @ xm_col).squeeze(-1).squeeze(-1)`.

**Formula:**
$$ P = -\frac{1}{2} (\mathbf{x}_m^T C_{2D}^{-1} \mathbf{x}_m) $$
where $\mathbf{x}_m$ is the $2 \times 1$ difference vector, $C_{2D}^{-1}$ is the $2 \times 2$ inverse covariance matrix, and $P$ is the exponent value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def evaluate_2d_gaussian_exponent(points_2d: torch.Tensor, means_2d: torch.Tensor, cov_2d: torch.Tensor) -> torch.Tensor:
    """
    Computes the exponent term of the 2D Gaussian PDF.
    
    Args:
        points_2d: (N, 2) tensor of query points.
        means_2d: (N, 2) tensor of Gaussian means.
        cov_2d: (N, 2, 2) tensor of covariance matrices.
        
    Returns:
        power: (N,) tensor of exponent values.
    """
    # Step 1: Difference vector
    xm = points_2d - means_2d  # (N, 2)
    
    # Step 2: Reshape for batched matrix multiplication
    # xm_col: (N, 2, 1), xm_row: (N, 1, 2)
    xm_col = xm.unsqueeze(-1)
    xm_row = xm.unsqueeze(-2)
    
    # Step 3: Inverse covariance
    # cov_2d_inv: (N, 2, 2)
    cov_2d_inv = cov_2d.inverse()
    # Unsqueezed for broadcasting: (N, 1, 2, 2)
    cov_2d_inv = cov_2d_inv.unsqueeze(1)
    
    # Step 4: Quadratic form and scaling
    # Result of matmul: (N, 1, 1)
    quadratic = xm_row @ cov_2d_inv @ xm_col
    power = -0.5 * quadratic.squeeze(-1).squeeze(-1)
    
    return power
```

**Edge Cases and Pitfalls:**
- **Singular Covariance:** If $C_{2D}$ is singular (determinant near zero), `inverse()` will produce NaN or Inf values. In practice, a small regularization term (e.g., $\epsilon I$) is often added to the covariance before inversion to ensure numerical stability [G].
- **Precision:** Using `float32` is standard, but for very small or large variances, `float64` may be required to avoid underflow/overflow in the exponent calculation [G].

## Subtopics

### Generalized Gaussian Particles
Particles using a kernel function of degree $n$ to control extent and density fall-off, replacing the standard quadratic exponent [S2]. The formula is $\rho(\mathbf{x}) = \exp(-\lambda((\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x} - \boldsymbol{\mu}))^{\frac{n}{2}})$, where $\lambda$ is a scale factor defined to get the same kernel response at a given distance $r$ as the reference Gaussian kernel (typical value $r=3$) [S2]. Degree $n=2$ corresponds to the standard Gaussian. Degree $n=4$ matches 3DGRT's generalized Gaussian of degree 2. Higher degrees produce denser particles with steeper and narrower fall-off [S2]. On MipNeRF360 (excluding flower and treehill): Degree 2 yields 28.77 PSNR at 207 FPS; Degree 4 yields 28.46 PSNR at 233 FPS; Degree 8 yields 27.63 PSNR at 243 FPS [S2].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ for $N$ points, dominated by the matrix inverse ($O(1)$ per Gaussian for $2 \times 2$) and matrix multiplications.
- Memory: $O(N)$ for storing inputs and outputs.

**Measured Performance:**
- Performance depends heavily on the hardware and the specific kernel degree if generalized Gaussians are used. For standard Gaussians, the operation is lightweight and often batched efficiently on GPUs [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Computationally efficient; separates the exponent calculation from normalization, allowing flexible opacity handling.
- Limitations: Requires non-singular covariance matrices; sensitive to numerical precision in the inverse operation.

**When to Use:**
- Use when rasterizing 3D Gaussians projected to 2D and only the exponent term is needed for further processing (e.g., combining with opacity or normalization factors).

## Background
Originated in the context of 3D Gaussian Splatting rasterization, where 3D Gaussians are projected to 2D and evaluated at pixel centers [S1, unofficial]. The method is a fundamental component of the forward pass in Gaussian Splatting renderers.

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
