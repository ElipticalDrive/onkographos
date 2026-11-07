# 2D Gaussian Evaluation

Also called: 2D Gaussian PDF exponent

## Overview
Computes the exponent term of the 2D Gaussian probability density function at given 2D points, used for rasterization or shading [S1, unofficial]. It calculates the quadratic form of the difference vector against the inverse covariance matrix, scaled by -0.5, which serves as the input to the exponential function for the final density value. It differs from [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md) by explicitly computing the inverse of the covariance matrix from its components rather than using pre-computed conic matrix components directly, and by operating on the raw difference vector before the exponential is applied.

## How it works
**Inputs:**
- `points_2D`: Tensor of shape $(N, 2)$ containing the 2D query coordinates.
- `means_2D`: Tensor of shape $(N, 2)$ containing the 2D Gaussian means.
- `cov_2D`: Tensor of shape $(N, 2, 2)$ containing the 2D covariance matrices.

**Outputs:**
- `power`: Tensor of shape $(N,)$ containing the exponent values $P$.

**Data Structures:**
- Tensors are assumed to be batched. No specific bit-packing is required; standard floating-point tensors are used.

**Steps:**
1. Compute the difference vector between query points and 2D means:
   $$ \mathbf{x}_m = \text{points\_2D} - \text{means\_2D} $$
   where $\mathbf{x}_m$ has shape $(N, 2)$.

2. Reshape the difference vector for batched matrix multiplication:
   - Create column vector: $\mathbf{x}_m^{\text{col}} = \text{unsqueeze}(\mathbf{x}_m, \text{dim}=-1)$, shape $(N, 2, 1)$.
   - Create row vector: $\mathbf{x}_m^{\text{row}} = \text{unsqueeze}(\mathbf{x}_m, \text{dim}=-2)$, shape $(N, 1, 2)$.

3. Compute the inverse of the 2D covariance matrix and prepare for broadcasting:
   $$ C_{2D}^{-1} = \text{inverse}(\text{cov\_2D}) $$
   Shape of $C_{2D}^{-1}$ is $(N, 2, 2)$.
   - Unsqueeze for broadcasting: $C_{2D}^{-1}_{\text{broadcast}} = \text{unsqueeze}(C_{2D}^{-1}, \text{dim}=1)$, shape $(N, 1, 2, 2)$.

4. Compute the quadratic form and scale by -0.5:
   $$ P = -0.5 \times (\mathbf{x}_m^{\text{row}} \times C_{2D}^{-1}_{\text{broadcast}} \times \mathbf{x}_m^{\text{col}}) $$
   The tensor multiplication results in shape $(N, 1, 1)$.
   - Squeeze the last two dimensions to get the final scalar exponent per point:
     $$ P = \text{squeeze}(P, \text{dim}=(-1, -2)) $$
     Shape of $P$ is $(N,)$.

**Formula:**
$$ P = -\frac{1}{2} (\mathbf{x}_m^T C_{2D}^{-1} \mathbf{x}_m) $$
where:
- $\mathbf{x}_m$ is the $2 \times 1$ difference vector $(\text{points\_2D} - \text{means\_2D})$.
- $C_{2D}^{-1}$ is the $2 \times 2$ inverse covariance matrix.
- $P$ is the exponent value.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_2d_gaussian(points_2d, means_2d, cov_2d):
    """
    Computes the exponent term of the 2D Gaussian PDF.
    
    Args:
        points_2d: Tensor of shape (N, 2)
        means_2d: Tensor of shape (N, 2)
        cov_2d: Tensor of shape (N, 2, 2)
        
    Returns:
        power: Tensor of shape (N,)
    """
    # Step 1: Difference vector
    xm = points_2d - means_2d  # Shape: (N, 2)
    
    # Step 2: Reshape for batched matrix multiplication
    xm_col = xm.unsqueeze(-1)  # Shape: (N, 2, 1)
    xm_row = xm.unsqueeze(-2)  # Shape: (N, 1, 2)
    
    # Step 3: Inverse covariance and broadcasting
    cov_2d_inv = torch.inverse(cov_2d)  # Shape: (N, 2, 2)
    cov_2d_inv_broadcast = cov_2d_inv.unsqueeze(1)  # Shape: (N, 1, 2, 2)
    
    # Step 4: Quadratic form and scaling
    # Matrix multiplication: (N, 1, 2) @ (N, 1, 2, 2) @ (N, 2, 1) -> (N, 1, 1)
    quadratic_form = torch.matmul(torch.matmul(xm_row, cov_2d_inv_broadcast), xm_col)
    
    # Scale by -0.5
    power = -0.5 * quadratic_form
    
    # Squeeze to remove batch dimensions of size 1
    power = power.squeeze(-1).squeeze(-1)  # Shape: (N,)
    
    return power
```

**Parameters:**
- No explicit parameters other than the inputs. The scaling factor is fixed at -0.5.

**Edge Cases and Pitfalls:**
- **Singular Covariance:** If `cov_2d` is singular (non-invertible), `torch.inverse` will fail or produce NaNs. Ensure covariance matrices are positive definite. [G]
- **Precision:** Matrix inversion can be numerically unstable for ill-conditioned matrices. [G]
- **Batch Size:** The method assumes batched inputs. For single points, ensure dimensions are handled correctly (e.g., adding batch dimension).

## Subtopics

### Generalized Gaussian Particles
Particles using a kernel function of degree $n$ to control extent and density fall-off, replacing the standard quadratic exponent [S2]. The kernel response is defined as $\rho(\mathbf{x}) = \exp(-\lambda((\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x} - \boldsymbol{\mu}))^{\frac{n}{2}})$, where $\mathbf{x}$ is the position, $\boldsymbol{\mu}$ is the mean, $\boldsymbol{\Sigma}$ is the covariance, $\lambda$ is a scale factor, and $n$ is the kernel degree [S2]. The scale factor $\lambda$ is defined to get the same kernel response at a given distance $r$ as the reference Gaussian kernel, with a typical value of $r = 3$ [S2]. Degree $n=2$ corresponds to the standard Gaussian. Degree $n=4$ matches 3DGRT's generalized Gaussian of degree 2. Higher degrees produce denser particles with steeper and narrower fall-off [S2]. On MipNeRF360 (excluding flower and treehill): Degree 2 yields 28.77 PSNR at 207 FPS; Degree 4 yields 28.46 PSNR at 233 FPS; Degree 8 yields 27.63 PSNR at 243 FPS [S2].

## Operating characteristics
**Time and Memory Complexity:**
- Time: Dominated by matrix inversion ($O(1)$ for fixed 2x2) and matrix multiplication ($O(N)$ for $N$ points).
- Memory: $O(N)$ for storing inputs and outputs.

**Measured Performance:**
- Not given in the sources for the standard 2D Gaussian evaluation method itself. Performance data for generalized variants is provided in Subtopics.

**Scaling:**
- Scales linearly with the number of points $N$.

**Strengths and Limitations:**
- Strengths: Simple, efficient computation of the Gaussian exponent.
- Limitations: Requires inversion of covariance matrix, which can be numerically unstable.

**When to Use:**
- Use when rasterizing or shading with standard 2D Gaussians.

**When Not to Use:**
- Use [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md) if conic matrix components are already available and inversion is to be avoided.
- Use Generalized Gaussian Particles if different kernel shapes are needed.

## Background
Origin: The method is part of the 3D Gaussian Splatting pipeline, specifically for evaluating the 2D Gaussian probability density function during rasterization [S1, unofficial]. It is used in implementations such as the unofficial 3D-Gaussian-Splatting repository [S1, unofficial].

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
