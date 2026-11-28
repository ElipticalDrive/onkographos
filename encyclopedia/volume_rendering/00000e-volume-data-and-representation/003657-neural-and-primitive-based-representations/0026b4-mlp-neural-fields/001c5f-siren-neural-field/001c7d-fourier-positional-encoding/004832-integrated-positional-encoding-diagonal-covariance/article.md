# Integrated Positional Encoding (Diagonal Covariance)

Also called: Expected Sine for Positional Encoding, integrated positional encoding component

## Overview
Integrated Positional Encoding (Diagonal Covariance) computes the expected value of sinusoidal positional encodings when input coordinates are modeled as Gaussian random variables with diagonal covariance [S1, S2, unofficial]. It solves the aliasing problem in neural fields by integrating over the spatial uncertainty (ray thickness) represented by the variance [S2, unofficial]. Unlike standard Fourier Positional Encoding, which treats inputs as point samples, this method accounts for the query's spatial extent and naturally suppresses high-frequency components when variance is large [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `x_coord` containing mean coordinates $x$ and covariance information. If `diagonal=True`, covariance is a vector of variances $x_{var}$; if `diagonal=False`, it is a full covariance matrix $x_{cov}$ [S1, S2, unofficial].
- Parameters: `min_deg` (int, minimum exponent), `max_deg` (int, maximum exponent, exclusive), `diagonal` (bool) [S1, unofficial].
- Output: Encoded vector containing the expected sine/cosine values and their variance-dependent scaling factors [S1, S2, unofficial].

**Data Structures:**
- `x`: Tensor of shape $(N, D)$, where $N$ is batch size and $D$ is dimensionality (typically 3) [S1, unofficial].
- `x_var`: Tensor of shape $(N, D)$, diagonal elements of covariance [S1, unofficial].
- `scales`: Tensor of shape $(K,)$, where $K = max\_deg - min\_deg$. Values are $2^i$ for $i \in [min\_deg, max\_deg)$ [S1, S2, unofficial].

**Steps:**
1. Extract mean $x$ and covariance ($x_{var}$ or $x_{cov}$) from `x_coord` [S1, S2, unofficial].
2. Compute frequency scales: $s_i = 2^i$ for $i$ from `min_deg` to `max_deg - 1` [S1, S2, unofficial].
3. Compute transformed mean $y$ and transformed variance $y_{var}$:
   - If diagonal: $y = x \cdot B_{diag}$ and $y_{var} = x_{var} \cdot B_{diag}^2$, implemented via broadcasting multiplication of $x$ and $x_{var}$ with `scales` and `scales^2` [S1, S2, unofficial].
   - If full: Construct basis matrix $B$ by concatenating $s_i \cdot I_D$ for each scale $i$. Compute $y = x \cdot B$ and $y_{var} = \sum_{j} (x_{cov} \cdot B)_{ij} \cdot B_{kj}$ via matrix multiplication and contraction [S1, S2, unofficial].
4. Concatenate results for sine and cosine components: `[y, y + 0.5 * pi]` and `[y_var, y_var]` [S1, S2, unofficial].
5. Apply the expected sine formula to compute the final encoded values [S1, unofficial].

**Formula:**
The core integration result for a Gaussian variable $z \sim \mathcal{N}(\mu, \sigma^2)$ is:
$$ E[\sin(z)] = e^{-\frac{1}{2}\sigma^2} \sin(\mu) $$
$$ E[\cos(z)] = e^{-\frac{1}{2}\sigma^2} \cos(\mu) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def integrated_pos_enc_diag(x_coord, min_deg, max_deg, diagonal=True):
    """
    Computes integrated positional encoding for diagonal covariance.
    
    Args:
        x_coord: Dict containing 'mean' (x) and 'var' (x_var) or 'cov' (x_cov).
                 x shape: (N, D)
                 x_var shape: (N, D)
                 x_cov shape: (N, D, D)
        min_deg: int, minimum exponent for scales.
        max_deg: int, maximum exponent for scales (exclusive).
        diagonal: bool, if True uses diagonal covariance logic.
        
    Returns:
        encoded: Tensor of shape (N, 2 * D * (max_deg - min_deg))
    """
    # 1. Extract mean and covariance from x_coord
    x = x_coord['mean']  # (N, D)
    if diagonal:
        x_var = x_coord['var']  # (N, D)
    else:
        x_cov = x_coord['cov']  # (N, D, D)

    # 2. Compute scales as 2^i for i in [min_deg, max_deg)
    exponents = np.arange(min_deg, max_deg)
    scales = np.power(2.0, exponents)  # (K,)

    # 3. Compute transformed mean y and variance y_var
    if diagonal:
        # y = x @ basis_diag, where basis_diag is diag(scales)
        # Equivalent to broadcasting multiplication
        y = x[:, :, np.newaxis] * scales[np.newaxis, np.newaxis, :]  # (N, D, K)
        y = y.reshape(x.shape[0], -1)  # (N, D*K)
        # y_var = x_var @ basis_diag^2
        scales_sq = np.square(scales)
        y_var = x_var[:, :, np.newaxis] * scales_sq[np.newaxis, np.newaxis, :]  # (N, D, K)
        y_var = y_var.reshape(x.shape[0], -1)  # (N, D*K)
    else:
        # Construct basis matrix by concatenating 2^i * Identity(D)
        basis = np.concatenate([scales[i] * np.eye(x.shape[1]) for i in range(len(scales))], axis=1)  # (D, D*K)
        y = x @ basis  # (N, D*K)
        # y_var = sum((x_cov @ basis) * basis, dim=-2)
        y_var = np.sum((x_cov @ basis) * basis, axis=-2)  # (N, D*K)

    # 4. Concatenate for sine and cosine components
    # Concatenate y and y + 0.5*pi, and y_var twice
    y_concat = np.concatenate([y, y + 0.5 * np.pi], axis=-1)
    y_var_concat = np.concatenate([y_var, y_var], axis=-1)

    # 5. Compute expected sine: E[sin(z)] = exp(-0.5 * var) * sin(mean)
    decay = np.exp(-0.5 * y_var_concat)
    encoded = decay * np.sin(y_concat)

    return encoded
```

**Pitfall:**
When the variance is wide, the sine function shrinks towards zero, which is handled by the exponential decay term $\exp(-0.5 \cdot x_{var})$ [S1, unofficial].

## Operating characteristics
- Time complexity: Not given in the sources [S1, S2, unofficial].
- Memory complexity: Not given in the sources [S1, S2, unofficial].
- Performance: Not given in the sources [S1, S2, unofficial].
- Strengths: Handles ray thickness and voxel size naturally by integrating over covariance uncertainty [S2, unofficial].
- Limitations: Not given in the sources [S1, S2, unofficial].
- When to use: Not given in the sources [S1, S2, unofficial].
- When not to use: Not given in the sources [S1, S2, unofficial].

## Background
Integrated Positional Encoding was introduced in the context of Mip-NeRF to address high-frequency aliasing caused by sampling points along rays with significant thickness [S1, S2, unofficial]. It builds on the idea that positional encodings should account for the spatial extent of the query, not just the mean position [S1, S2, unofficial]. The diagonal covariance variant assumes independence between coordinate dimensions to compute the expected value efficiently [S1, S2, unofficial].

## Sources
[S1] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S2] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
