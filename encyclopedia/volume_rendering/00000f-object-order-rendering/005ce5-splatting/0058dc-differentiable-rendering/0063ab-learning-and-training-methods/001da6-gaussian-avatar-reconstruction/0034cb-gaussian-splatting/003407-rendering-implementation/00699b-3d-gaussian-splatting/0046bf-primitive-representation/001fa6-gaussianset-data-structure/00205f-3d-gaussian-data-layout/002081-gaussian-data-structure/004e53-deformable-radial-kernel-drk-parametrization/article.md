# Deformable Radial Kernel (DRK) Parametrization

Also called: DRK attribute modeling

## Overview
Deformable Radial Kernel (DRK) parametrization defines the unconstrained learnable parameters for the attributes of 3D Gaussian kernels, ensuring they remain within valid physical ranges through specific activation functions [S1]. It distinguishes itself from standard 3D Gaussian Splatting by introducing a specialized three-step activation for basis angles $\theta_k$ to maintain angular separation and monotonicity, and by providing configurable density control thresholds for sparse training scenarios [S1].

## How it works
**Inputs:** Unconstrained learnable parameters for opacity, scale, rotation, sharpness, blending weight, and basis angles, all initialized in $(-\infty, \infty)$.
**Outputs:** Validated Gaussian attributes ready for rendering.

**Data Structures:**
The method operates on the scalar learnable variables associated with each Gaussian kernel. No specific bit-packing is defined in the sources; parameters are treated as floating-point values during optimization [S1].

**Activation Functions:**
Each attribute is transformed from its unconstrained space to its valid range using the following functions [S1]:

1.  **Opacity ($o$):** Constrained to $(0, 1)$ using a sigmoid function.
    $$ o = \sigma(\hat{o}) = \frac{1}{1 + e^{-\hat{o}}} $$
    where $\hat{o}$ is the unconstrained parameter.

2.  **Scale ($s_k$):** Activated using an exponential function to ensure positive values.
    $$ s_k = \exp(\hat{s}_k) $$
    where $\hat{s}_k$ is the unconstrained parameter.

3.  **Rotation ($q$):** Constrained to the unit hypersphere using normalization.
    $$ q = \frac{\hat{q}}{\|\hat{q}\|} $$
    where $\hat{q}$ is the unconstrained 4D quaternion vector.

4.  **Sharpness ($\tau$):** Bounded within $(-0.1, 0.99)$ using a composite function combining sigmoid and linear remapping. The exact linear remapping coefficients are not specified in the sources [S1].

5.  **L1&L2 Blending Weight ($\eta$):** Activated using a sigmoid function.
    $$ \eta = \sigma(\hat{\eta}) $$
    where $\hat{\eta}$ is the unconstrained parameter.

6.  **Basis Angles ($\theta_k$):** For $K$ basis angles, a three-step activation process is employed to enforce monotonic increase and minimum angular separation [S1]:
    *   **Step 1:** Apply sigmoid to unconstrained angles $\hat{\theta}_k$.
    *   **Step 2:** Add a residual term $\frac{1}{K-2}$ to maintain minimum angular separation.
    *   **Step 3:** Apply cumulative summation to enforce monotonic increase.
    *   **Step 4:** Normalize the resulting angles so that the final angle $s_K = 2\pi$.

    The residual term $\frac{1}{K-2}$ prevents basis polar angles from exceeding $\pi$, thereby avoiding degradation in representation [S1].

**Parameters:**
*   **$K$ (Number of basis angles):** Set to 8 to balance flexibility and memory efficiency [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def activate_opacity(unconstrained_o):
    """Constrain opacity to (0, 1)."""
    return 1.0 / (1.0 + math.exp(-unconstrained_o))

def activate_scale(unconstrained_s):
    """Constrain scale to positive values."""
    return math.exp(unconstrained_s)

def activate_rotation(unconstrained_q):
    """Normalize quaternion to unit length."""
    norm = math.sqrt(sum(x**2 for x in unconstrained_q))
    return [x / norm for x in unconstrained_q]

def activate_sharpness(unconstrained_tau):
    """
    Bound sharpness within (-0.1, 0.99).
    Note: Exact linear remapping coefficients are not given in sources.
    Placeholder for composite sigmoid + linear remapping [G].
    """
    # Generic sigmoid
    sig = 1.0 / (1.0 + math.exp(-unconstrained_tau))
    # Linear remapping from (0,1) to (-0.1, 0.99)
    # Assuming standard linear mapping: y = min + (max - min) * x
    return -0.1 + (0.99 - (-0.1)) * sig

def activate_blending_weight(unconstrained_eta):
    """Constrain blending weight to (0, 1)."""
    return 1.0 / (1.0 + math.exp(-unconstrained_eta))

def activate_basis_angles(unconstrained_thetas, K=8):
    """
    Activate basis angles with monotonic increase and minimum separation.
    Args:
        unconstrained_thetas: List of K unconstrained float values.
        K: Number of basis angles (default 8).
    Returns:
        List of K normalized angles summing to 2*pi.
    """
    # Step 1: Apply sigmoid
    sigmas = [1.0 / (1.0 + math.exp(-t)) for t in unconstrained_thetas]
    
    # Step 2: Add residual term 1/(K-2)
    residual = 1.0 / (K - 2)
    adjusted = [s + residual for s in sigmas]
    
    # Step 3: Cumulative summation
    cumulative = []
    current_sum = 0.0
    for val in adjusted:
        current_sum += val
        cumulative.append(current_sum)
        
    # Step 4: Normalize to make sK = 2*pi
    final_sum = cumulative[-1]
    if final_sum == 0:
        # Avoid division by zero; return uniform distribution if sum is zero
        return [2 * math.pi / K] * K
    
    normalized = [val / final_sum * (2 * math.pi) for val in cumulative]
    
    return normalized
```

## Subtopics

### DRK Model Training
This variant describes the optimization process for DRK parameters. It follows the adaptive training process of 3D-GS, dynamically adjusting kernel density [S1].
*   **Learning Rate:** DRK-specific parameters (sharpness $\tau$, blending weight $\eta$, basis angles $\theta_k$, and scales $s_k$) use a uniform learning rate of $5 \times 10^{-3}$ [S1].
*   **Decay:** The learning rate decays gradually to $1 \times 10^{-2} \times$ (initial rate) at the end of training (35K steps) [S1]. Note: The source text "1e-2 ×" is ambiguous regarding the multiplier target; it is cited as written [S1].
*   **Density Control:** Three configurations are defined by pairs of (2D screen gradient densification threshold, opacity pruning threshold) [S1]:
    1.  **Comparable to 3D-GS:** $(5 \times 10^{-4}, 5 \times 10^{-2})$
    2.  **Sparse Level 1 (S1):** $(1 \times 10^{-3}, 5 \times 10^{-2})$
    3.  **Sparse Level 2 (S2):** $(2 \times 10^{-3}, 1 \times 10^{-1})$

## Operating characteristics
**Time and Memory Complexity:**
The sources do not provide explicit Big-O complexity for the activation functions. The choice of $K=8$ for basis angles is noted to balance flexibility and memory efficiency [S1].

**Measured Performance:**
The method demonstrates robustness under noisy conditions, with rendering results shown for noise standard deviations of 0 and $10^{-3}$ [S1].

**Strengths and Limitations:**
*   **Strengths:** The residual term in basis angle activation prevents polar angles from exceeding $\pi$, avoiding representation degradation [S1]. The configurable density thresholds allow for controlled sparsity (S1, S2) [S1].
*   **Limitations:** The exact linear remapping formula for sharpness is not provided [S1]. The learning rate decay target is ambiguously stated as "1e-2 ×" [S1].

## Background
DRK parametrization is part of the Deformable Radial Kernel Splatting method, introduced in a paper published in 2025 [S1]. It extends the Gaussian Data Structure by providing specialized parameterizations for dynamic and deformable scenes, building on the foundations of 3D Gaussian Splatting [S1].

## Sources
[S1] Deformable Radial Kernel Splatting, https://doi.org/10.1109/CVPR52734.2025.02004, fetched 2026-10-30
