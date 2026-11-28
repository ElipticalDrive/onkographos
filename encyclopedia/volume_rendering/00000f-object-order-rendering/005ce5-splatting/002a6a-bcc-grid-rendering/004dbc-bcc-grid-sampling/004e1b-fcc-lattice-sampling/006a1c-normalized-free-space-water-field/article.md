# Normalized Free-Space Water Field

Also called: Optical-Mass Water Gaussians

## Overview
The Normalized Free-Space Water Field is a volumetric representation for underwater rendering that models light attenuation and backscatter using fixed Gaussian bases on an FCC lattice [S1, unofficial]. It solves the problem of simulating participating media by optimizing only two non-negative RGB density vectors per lattice point (direct and backscatter attenuation) while keeping geometry fixed to prevent transport mass changes via geometric resizing [S1, unofficial]. It differs from standard 3D Gaussian Splatting by decoupling the spatial sampling lattice (FCC) from the optical parameter optimization, ensuring that the integral of the optical mass remains consistent under lattice refinement [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Scene bounds defined by Structure-from-Motion (SfM) points plus a fixed world-space margin.
*   Lattice resolution parameter $N$ (cells per axis).
*   Standard deviation parameter $s_\Delta$ for the Gaussian covariance.
*   Initial RGB density vectors $\rho_{D,i}$ and $\rho_{B,i}$ for each lattice point $i$.

**Outputs:**
*   Continuous direct extinction field $\sigma_D(x)$.
*   Continuous backscatter attenuation field $\sigma_B(x)$.

### Data Structures
The method uses a fixed face-centered-cubic (FCC) lattice.
*   **Lattice Generation:** For $N$ cells per axis, there are $4N^3$ centers. Each axis-aligned cell contains four offsets.
*   **Center Position ($\mu_i$):** Fixed 3D coordinates on the FCC lattice within the axis-aligned bounds.
*   **Covariance ($\Sigma_i$):** Fixed axis-aligned diagonal matrix with standard deviation $s_\Delta$ along each dimension.
*   **Optimized Parameters:** Two RGB vectors per center $i$:
    *   $\rho_{D,i}$: Direct extinction density.
    *   $\rho_{B,i}$: Backscatter attenuation density.

### Formulas
The basis function is a normalized 3D Gaussian [S1, unofficial]:

$$
\phi_i(x) = \frac{\exp \left[ -\frac{1}{2} (x - \mu_i)^\top \Sigma_i^{-1} (x - \mu_i) \right]}{(2\pi)^{3/2} |\Sigma_i|^{1/2}}
$$

Where $\int_{\mathbb{R}^3} \phi_i(x) dx = 1$.

The volume assigned to each lattice point is derived from the cell volume $\Delta V = \Delta x \Delta y \Delta z$:

$$
V_i = \frac{\Delta V}{4}
$$

Optical mass is assigned using the softplus function to ensure non-negativity [S1, unofficial]:

$$
m_{D,i} = V_i \text{softplus}(\rho_{D,i})
$$
$$
m_{B,i} = V_i \text{softplus}(\rho_{B,i})
$$

The continuous optical fields are computed as the sum of weighted basis functions [S1, unofficial]:

$$
\sigma_D(x) = \sum_i m_{D,i} \phi_i(x)
$$
$$
\sigma_B(x) = \sum_i m_{B,i} \phi_i(x)
$$

### Parameters
*   **Standard Deviation ($s_\Delta$):** The compact configuration uses $s = 1.3$. An initial layout with $s = 0.7$ is retained as a control [S1, unofficial].
*   **Softplus:** $\text{softplus}(x) = \ln(1 + e^x)$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_water_field(lattice_centers, lattice_covariances, cell_volume, rho_D, rho_B):
    """
    Compute continuous optical fields for the Normalized Free-Space Water Field.
    
    Args:
        lattice_centers: Array of shape (N_centers, 3) containing mu_i.
        lattice_covariances: Array of shape (N_centers, 3) containing standard deviations s_delta for each axis.
        cell_volume: Scalar Delta V.
        rho_D: Array of shape (N_centers, 3) for direct extinction densities.
        rho_B: Array of shape (N_centers, 3) for backscatter attenuation densities.
        
    Returns:
        sigma_D: Function or array representing direct extinction field.
        sigma_B: Function or array representing backscatter attenuation field.
    """
    N_centers = lattice_centers.shape[0]
    V_i = cell_volume / 4.0
    
    # Calculate covariance determinants and inverses for each center
    # Covariance is diagonal: diag(s^2, s^2, s^2)
    # |Sigma| = s^6, Sigma^-1 = diag(1/s^2, 1/s^2, 1/s^2)
    s_sq = lattice_covariances ** 2
    det_Sigma = np.prod(s_sq, axis=1) # Shape (N_centers,)
    inv_Sigma = 1.0 / s_sq # Shape (N_centers, 3)
    
    # Precompute normalization constant denominator part
    # (2*pi)^(3/2) * |Sigma|^(1/2)
    norm_const = (2 * np.pi) ** 1.5 * np.sqrt(det_Sigma) # Shape (N_centers,)
    
    # Compute optical masses
    # softplus(x) = log(1 + exp(x))
    softplus_D = np.log(1 + np.exp(rho_D))
    softplus_B = np.log(1 + np.exp(rho_B))
    
    m_D = V_i * softplus_D # Shape (N_centers, 3)
    m_B = V_i * softplus_B # Shape (N_centers, 3)
    
    # To evaluate at a query point x_query (shape (3,)):
    # sigma_D(x) = sum_i m_D,i * phi_i(x)
    # phi_i(x) = exp(-0.5 * (x-mu)^T Sigma^-1 (x-mu)) / norm_const_i
    
    def evaluate_fields(x_query):
        # x_query shape (3,)
        # Broadcast to (N_centers, 3)
        diff = lattice_centers - x_query # Shape (N_centers, 3)
        
        # Quadratic form: (x-mu)^T Sigma^-1 (x-mu)
        # Since Sigma^-1 is diagonal, this is sum over dims of diff^2 * inv_Sigma
        # diff_sq = diff ** 2
        # exponent_arg = np.sum(diff_sq * inv_Sigma, axis=1) # Shape (N_centers,)
        
        # More efficient vectorized calculation
        exponent_arg = np.sum(diff ** 2 * inv_Sigma, axis=1)
        
        # Basis functions
        phi = np.exp(-0.5 * exponent_arg) / norm_const # Shape (N_centers,)
        
        # Weighted sum
        sigma_D_val = np.sum(m_D * phi[:, np.newaxis], axis=0) # Shape (3,)
        sigma_B_val = np.sum(m_B * phi[:, np.newaxis], axis=0) # Shape (3,)
        
        return sigma_D_val, sigma_B_val

    return evaluate_fields
```

### Edge Cases and Pitfalls
*   **Truncation:** Gaussian bases are not truncated at declared bounds. The method distinguishes between assigned full-space mass and mass inside the hard box [S1, unofficial]. Implementers must account for the infinite support of the Gaussian when integrating or bounding the field.
*   **Mass Conservation:** Keeping position and covariance fixed prevents transport mass from changing through geometric resizing. Optimizing only density vectors ensures optical mass scales with the assigned volume under lattice refinement [S1, unofficial].

## Operating characteristics
*   **Complexity:** The number of centers scales as $4N^3$ for $N$ cells per axis. Evaluation of the field at a point requires summing over all centers, leading to $O(N^3)$ complexity per evaluation unless spatial acceleration structures are used.
*   **Memory:** Requires storing $4N^3$ centers, covariances, and two RGB density vectors.
*   **Strengths:** Fixed geometry prevents artifacts from geometric resizing. The FCC lattice offers better sampling efficiency than Cartesian grids.
*   **Limitations:** The infinite support of Gaussians requires careful handling of bounds for integration. The method is specific to underwater rendering scenarios requiring direct and backscatter attenuation modeling.

## Background
The Normalized Free-Space Water Field was introduced in the context of optical-mass water Gaussians for per-scene underwater rendering [S1, unofficial]. It builds upon 3D Gaussian Splatting techniques but modifies the splatted surface statistics query by the water field. The approach uses a face-centered-cubic (FCC) lattice for sampling, which is a subset of the Cartesian lattice offering better sampling efficiency [S1, unofficial].

## Sources
[S1] Optical-Mass Water Gaussians for Per-Scene Underwater Rendering, https://doi.org/10.21203/rs.3.rs-10938586/v1, fetched 2026-10-26 (unofficial)
