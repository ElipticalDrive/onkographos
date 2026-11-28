# Optimal Layer-wise QP Offset Derivation

Also called: Analytical QP offset solution, RDO-based QP offset calculation

## Overview
Optimal Layer-wise QP Offset Derivation is a method for optimizing bit allocation across Recursive Adaptive Hierarchical Transform (RAHT) layers in point cloud attribute compression by minimizing a Lagrangian cost function [S1, unofficial]. It solves the problem of determining the optimal Quantization Parameter (QP) offset for each layer to balance bitrate and distortion, providing an analytical solution that serves as a ground truth for rate-distortion optimization (RDO) [S1, unofficial]. It differs from heuristic approaches by deriving offsets directly from fitted empirical models of bitrate and distortion behavior [S1, unofficial].

## How it works
**Inputs:**
- Number of RAHT layers $M$.
- Empirical parameters for each layer $i \in \{0, \dots, M-1\}$: bitrate slope $k_i$, bitrate constant $C_i$, distortion sensitivity $\nu_i$, intrinsic distortion floor $Z_i$, and model fitting parameter $\mu$.
- Lagrange multiplier $\lambda$.

**Outputs:**
- Optimal QP offset $\Delta qpi$ for each layer $i$.

**Data Structures:**
- Arrays of size $M$ for parameters $k, C, \nu, Z$.
- Scalar $\mu$ (shared across layers).
- Scalar $\lambda$.

**Steps:**
1. **Characterize Quantization Error Statistics:**
   - Set the target RAHT layer's QP to 24 and all other layers to QP=4 [S1, unofficial].
   - Verify that distortion introduced by quantization at each layer follows a Gaussian distribution [S1, unofficial].
   - Note: At high bit depths (e.g., for 3D Gaussian Splatting attributes), precision errors become significant. Mitigate this by signaling a higher configurable fixed-point precision (e.g., 20 bits) under the Attribute Parameter Set (APS) extension in the G-PCC standard [S1, unofficial].

2. **Fit Bitrate Model:**
   - Model the bitrate $R$ as a linear function of the QP offset $\Delta qpi$ for each layer $i$:
     $$ R = \sum_{i=0}^{M-1} (k_i \cdot \Delta qpi + C_i) $$
   - Determine $k_i$ and $C_i$ by fitting empirical data [S1, unofficial].

3. **Fit Distortion Model:**
   - Model the total distortion $D$ as the sum of layer-wise distortions $D_i$, where each $D_i$ follows an exponential relationship with $\Delta qpi$:
     $$ D = \sum_{i=0}^{M-1} (\nu_i \cdot 2^{\mu \cdot \Delta qpi} + Z_i) $$
   - Determine $\nu_i$, $Z_i$, and $\mu$ by fitting empirical data [S1, unofficial].

4. **Define Lagrangian Cost Function:**
   - Combine bitrate and distortion into the cost function $J$:
     $$ J = D + \lambda R $$
   - Substitute the models:
     $$ J = \sum_{i=0}^{M-1} \left( \nu_i \cdot 2^{\mu \cdot \Delta qpi} + Z_i + \lambda (k_i \cdot \Delta qpi + C_i) \right) $$

5. **Minimize Cost Function:**
   - Solve for $\Delta qpi$ by setting the partial derivative $\frac{\partial J}{\partial \Delta qpi} = 0$:
     $$ \frac{\partial}{\partial \Delta qpi} \left( \nu_i \cdot 2^{\mu \cdot \Delta qpi} + \lambda k_i \Delta qpi \right) = 0 $$
     $$ \nu_i \cdot 2^{\mu \cdot \Delta qpi} \cdot \mu \ln(2) + \lambda k_i = 0 $$
   - Rearrange to obtain the analytical solution:
     $$ \Delta qpi = \frac{1}{\mu} \log_2 \left( \frac{-\lambda k_i}{\nu_i \mu \ln(2)} \right) $$

**Parameters:**
- $\lambda$: Lagrange multiplier determining the trade-off between bitrate and distortion. Determined from the G-PCC RAHT anchor [S1, unofficial].
- $k_i$: Slope parameter for the bitrate model of layer $i$. Obtained by fitting empirical data [S1, unofficial].
- $C_i$: Constant parameter for the bitrate model of layer $i$. Obtained by fitting empirical data [S1, unofficial].
- $\nu_i$: Distortion sensitivity parameter for layer $i$. Obtained by fitting empirical data [S1, unofficial].
- $Z_i$: Intrinsic distortion floor for layer $i$. Obtained by fitting empirical data [S1, unofficial].
- $\mu$: Model fitting parameter for the exponential distortion relationship. Obtained by fitting empirical data [S1, unofficial].

**Pitfalls:**
- **Precision Errors:** At high magnitudes (common in 3DGS attributes), precision errors become non-negligible, setting a high floor for minimum achievable distortion [S1, unofficial]. Mitigate by using higher fixed-point precision (e.g., 20 bits) signaled via APS extension [S1, unofficial].
- **Parameter Fitting:** Parameters $k_i, C_i, \nu_i, Z_i, \mu$ must be fitted empirically for each sequence or attribute type. Using incorrect fitted values will lead to suboptimal offsets [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_optimal_qp_offsets(M, k, C, nu, Z, mu, lambda_val):
    """
    Compute optimal QP offsets for M RAHT layers.
    
    Args:
        M (int): Number of RAHT layers.
        k (list[float]): Bitrate slope parameters for each layer.
        C (list[float]): Bitrate constant parameters for each layer.
        nu (list[float]): Distortion sensitivity parameters for each layer.
        Z (list[float]): Intrinsic distortion floor parameters for each layer.
        mu (float): Model fitting parameter for exponential distortion.
        lambda_val (float): Lagrange multiplier.
        
    Returns:
        list[float]: Optimal QP offsets for each layer.
    """
    offsets = []
    for i in range(M):
        # Calculate the term inside the logarithm
        # Note: lambda_val is typically negative in minimization context if R is subtracted,
        # but here J = D + lambda*R. For minimization, lambda > 0.
        # The derivative term is: nu_i * 2^(mu * delta_qpi) * mu * ln(2) + lambda * k_i = 0
        # => 2^(mu * delta_qpi) = (-lambda * k_i) / (nu_i * mu * ln(2))
        # Since 2^x > 0, the RHS must be positive. 
        # In standard RDO, lambda > 0, k_i < 0 (bitrate decreases as QP offset increases? Or k_i is slope of R vs delta_qpi).
        # According to source: R = sum(ki * delta_qpi + Ci). If delta_qpi increases, bitrate decreases? 
        # Usually QP offset increases -> quantization coarser -> bitrate decreases. So k_i should be negative.
        # Thus -lambda * k_i is positive.
        
        numerator = -lambda_val * k[i]
        denominator = nu[i] * mu * math.log(2)
        
        if denominator == 0:
            raise ValueError("Denominator is zero, cannot compute offset.")
            
        ratio = numerator / denominator
        
        if ratio <= 0:
            # This case should not happen if parameters are physically consistent
            # and lambda, k, nu, mu have correct signs.
            # If it does, clamp or handle as error.
            raise ValueError(f"Log argument {ratio} is not positive for layer {i}.")
            
        delta_qpi = (1.0 / mu) * math.log2(ratio)
        offsets.append(delta_qpi)
        
    return offsets
```

## Operating characteristics
- **Time Complexity:** $O(M)$, where $M$ is the number of RAHT layers. The calculation involves a constant number of arithmetic operations per layer.
- **Memory Complexity:** $O(M)$ to store the parameters and results.
- **Strengths:** Provides a theoretically optimal solution for bit allocation given the fitted models. It is computationally efficient compared to iterative RDO searches.
- **Limitations:** Relies heavily on the accuracy of the fitted parameters ($k_i, C_i, \nu_i, Z_i, \mu$). If the models do not accurately reflect the actual bitrate/distortion behavior of the data, the offsets will be suboptimal. It does not account for rendering-aware adjustments, which may require further modeling [S1, unofficial].
- **When to Use:** When an analytical baseline for QP offset is needed for RAHT-compressed point cloud attributes, particularly for high-precision data like 3D Gaussian Splatting attributes.
- **When Not to Use:** When computational resources are extremely limited and only a heuristic is acceptable, or when the data characteristics deviate significantly from the assumed Gaussian distortion and linear bitrate models.

## Background
The method was introduced in the context of optimizing RAHT coding for 3D Gaussian Splatting (3DGS) attributes [S1, unofficial]. It addresses the vulnerability of 3DGS data to precision errors due to high bit depths compared to standard point cloud attributes [S1, unofficial]. The approach builds on the observation that quantization errors in RAHT layers can be modeled statistically, allowing for an analytical solution to the rate-distortion optimization problem [S1, unofficial].

## Sources
[S1] A Versatile Rate-Distortion Optimized RAHT Coding Layer for 3D Gaussian Splatting, https://centralesupelec.hal.science/hal-05586765v1, fetched 2026-10-26 (unofficial)
