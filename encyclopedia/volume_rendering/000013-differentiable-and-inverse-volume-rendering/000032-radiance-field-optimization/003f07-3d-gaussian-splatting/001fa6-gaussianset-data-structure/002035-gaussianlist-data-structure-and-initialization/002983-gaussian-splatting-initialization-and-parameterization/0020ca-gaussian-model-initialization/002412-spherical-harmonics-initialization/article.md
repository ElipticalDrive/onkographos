# Spherical Harmonics Initialization


## Overview
Spherical Harmonics Initialization sets the initial view-dependent color coefficients for 3D Gaussian splats by assigning input RGB colors to the zeroth-order (DC) spherical harmonic term while zeroing all higher-order coefficients [S1, unofficial]. It differs from [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md) and [DreamGaussian](5998-dreamgaussian.md) by focusing exclusively on the color parameterization rather than spatial position, scale, or rotation [S1, unofficial]. Unlike [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md), which computes color at runtime, this method prepares the static coefficient tensor for optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ RGB colors, shape $(N, 3)$, where each row is $(R, G, B)$ in the range $[0, 1]$ or $[0, 255]$ (implementation dependent, but the formula assumes normalized or scaled values consistent with the basis function).
- Output: A tensor of SH coefficients, shape $(N, 27)$, representing 3 color channels $\times$ 9 SH coefficients (degree 0 to 2).

**Data Structures:**
- `sh_coeffs`: A float32 tensor of shape $(N, 27)$.
- The 27 coefficients are organized as 3 channels, each with 9 coefficients corresponding to the spherical harmonic basis functions up to degree 2. The layout is typically channel-major or flattened from a $(N, 3, 9)$ structure.

**Steps:**
1. Allocate a zero tensor `sh_coeffs` of shape $(N, 3, 9)$ [S1, unofficial].
2. For each Gaussian $i$ from $0$ to $N-1$:
   a. Retrieve the input RGB color vector $\mathbf{C} = [R, G, B]$ [S1, unofficial].
   b. Compute the DC coefficient $c_0$ for each color channel using the formula:
      $$ c_0 = \frac{\text{RGB}}{0.28209479177387814} $$
      where $0.28209479177387814$ is the normalization constant for the zeroth-order spherical harmonic basis function $Y_0^0$ [S1, unofficial].
   c. Assign $c_0$ to the index 0 of the SH coefficient array for that channel. Indices 1 through 8 remain 0.
3. Flatten the last two dimensions of `sh_coeffs` from $(N, 3, 9)$ to $(N, 27)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_sh(rgb_colors: np.ndarray) -> np.ndarray:
    """
    Initialize spherical harmonic coefficients for 3D Gaussian splatting.
    
    Args:
        rgb_colors: Array of shape (N, 3) containing RGB values.
        
    Returns:
        Array of shape (N, 27) containing flattened SH coefficients.
    """
    N = rgb_colors.shape[0]
    # Allocate zero tensor of shape (N, 3, 9)
    sh_coeffs = np.zeros((N, 3, 9), dtype=np.float32)
    
    # Normalization constant for zeroth-order SH basis function
    SH_DC_NORMALIZATION = 0.28209479177387814
    
    # Assign scaled RGB values to the DC coefficient index (0) for all channels
    # rgb_colors shape: (N, 3)
    # sh_coeffs[:, :, 0] shape: (N, 3)
    sh_coeffs[:, :, 0] = rgb_colors / SH_DC_NORMALIZATION
    
    # Flatten the last two dimensions to produce a tensor of shape (N, 27)
    sh_coeffs_flat = sh_coeffs.reshape(N, 27)
    
    return sh_coeffs_flat
```

**Parameters:**
- `0.28209479177387814`: The normalization constant for the zeroth-order spherical harmonic basis function [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Color Range:** The input RGB values must be consistent with the expected range for the division. If RGB is in $[0, 255]$, it should be normalized to $[0, 1]$ before division or the constant adjusted accordingly. The source implies direct division, suggesting normalized inputs or a specific scaling convention [S1, unofficial].
- **Higher-Order Coefficients:** All coefficients from index 1 to 8 for each channel are explicitly set to 0. This assumes an isotropic initial color appearance.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians. The operation involves a single division and assignment per Gaussian.
- **Memory Complexity:** $O(N)$, storing 27 floats per Gaussian.
- **Strengths:** Simple, fast, and provides a valid initial guess for view-dependent color optimization.
- **Limitations:** Does not account for view-dependent effects initially; relies entirely on optimization to learn higher-order SH coefficients.

## Background
This method is part of the initialization pipeline for 3D Gaussian Splatting, specifically handling the color parameterization. It is derived from the implementation details found in community repositories and unofficial sources related to the 3D Gaussian Splatting algorithm [S1, unofficial].

## Sources
[S1] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
