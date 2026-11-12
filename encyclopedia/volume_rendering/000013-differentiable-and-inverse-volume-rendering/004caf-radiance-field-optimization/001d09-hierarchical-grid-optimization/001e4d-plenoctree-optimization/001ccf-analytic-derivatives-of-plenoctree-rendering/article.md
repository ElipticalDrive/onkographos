# Analytic Derivatives of PlenOctree Rendering
Also called: Gradient Computation for PlenOctree Optimization, NeRF Piecewise Constant Volume Rendering Derivatives

## Overview
This method provides the closed-form gradients of the NeRF piecewise constant volume rendering equation with respect to the density ($\boldsymbol{\sigma}$) and color ($\mathbf{c}$) parameters [S1, unofficial]. It enables direct optimization of PlenOctree attributes using gradient-based methods (e.g., SGD) to refine image quality [S1, unofficial].

## How it works
**Inputs:**
- Ray segment endpoints $t_0, \dots, t_N$ where $t_0$ is near and $t_N$ is far [S1, unofficial].
- Segment lengths $\delta_i = t_{i+1} - t_i$ for $i=0, \dots, N-1$ [S1, unofficial].
- Density values $\boldsymbol{\sigma} = (\sigma_0, \dots, \sigma_{N-1})$ where $\sigma_i \ge 0$ [S1, unofficial].
- Color values $\mathbf{c} = (c_0, \dots, c_N)$ where $c_i \in [0, 1]^3$ are RGB colors. $c_N$ is the background color (typically a hyperparameter) [S1, unofficial].

**Outputs:**
- Gradient of rendered color $\hat{\mathbf{C}}$ with respect to density: $\frac{\partial \hat{\mathbf{C}}}{\partial \sigma_i}$ [S1, unofficial].
- Gradient of rendered color $\hat{\mathbf{C}}$ with respect to color: $\frac{\partial \hat{\mathbf{C}}}{\partial c_i}$ [S1, unofficial].

**Data Structures:**
- Arrays for $\delta$, $\sigma$, $c$, $T$, and $w$ of size $N$ or $N+1$ as appropriate [S1, unofficial].
- $T$ stores accumulated transmittance [S1, unofficial].
- $w$ stores segment weights [S1, unofficial].

**Steps:**

1. **Compute Segment Lengths:**
   For $i = 0$ to $N-1$:
   $$ \delta_i = t_{i+1} - t_i $$

2. **Compute Accumulated Transmittance $T_i$:**
   Initialize $T_0 = 1.0$.
   For $i = 0$ to $N-1$:
   $$ T_{i+1} = T_i \cdot \exp(-\delta_i \sigma_i) $$
   Note: $T_i$ represents the transmittance from $t_0$ to $t_i$ [S1, unofficial].

3. **Compute Segment Weights $w_i$:**
   For $i = 0$ to $N-1$:
   $$ w_i = T_i - T_{i+1} $$
   For the background:
   $$ w_N = T_N $$
   Note: $w_i(\boldsymbol{\sigma}) = T_i(\boldsymbol{\sigma})(1 - \exp(-\sigma_i \delta_i))$ [S1, unofficial].

4. **Compute Rendered Color (for verification or loss calculation):**
   $$ \hat{\mathbf{C}} = \sum_{i=0}^{N} w_i c_i $$

5. **Compute Color Gradients:**
   For $i = 0$ to $N$:
   $$ \frac{\partial \hat{\mathbf{C}}}{\partial c_i} = w_i $$
   If colors are represented as Spherical Harmonics (SH) coefficients, apply the chain rule [S1, unofficial]. Since SH basis function values are constant across the ray, the derivative with respect to SH coefficients is $w_i \cdot \text{SH\_basis}(i)$ [S1, unofficial].

6. **Compute Density Gradients:**
   For each $i \in \{0, \dots, N-1\}$:
   $$ \frac{\partial \hat{\mathbf{C}}}{\partial \sigma_i} = \delta_i \left[ c_i T_{i+1} - \sum_{k=i+1}^N c_k w_k \right] $$
   Derivation note: The general form is $\frac{\partial \hat{\mathbf{C}}}{\partial \sigma_i} = c_N \frac{\partial T_N}{\partial \sigma_i} + \sum_{k=0}^{N-1} c_k \left( \frac{\partial T_k}{\partial \sigma_i} - \frac{\partial T_{k+1}}{\partial \sigma_i} \right)$ [S1, unofficial]. Using $\frac{\partial T_k}{\partial \sigma_i} = -\delta_i T_k \mathbf{1}_{k>i}$, this simplifies to the expression above [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_gradients(t, sigma, c, c_bg):
    """
    t: array of length N+1, ray segment endpoints [t0, ..., tN]
    sigma: array of length N, densities [sigma0, ..., sigma_{N-1}]
    c: array of length N, colors [c0, ..., c_{N-1}]
    c_bg: scalar or vector, background color cN
    
    Returns:
    grad_sigma: array of length N, gradients wrt sigma
    grad_c: array of length N, gradients wrt c
    """
    N = len(sigma)
    
    # 1. Segment lengths
    delta = t[1:] - t[:-1]  # length N
    
    # 2. Accumulated Transmittance T
    # T[i] is transmittance from t0 to ti
    T = [0.0] * (N + 1)
    T[0] = 1.0
    for i in range(N):
        T[i+1] = T[i] * math.exp(-delta[i] * sigma[i])
        
    # 3. Segment Weights w
    # w[i] for i < N is T[i] - T[i+1]
    # w[N] is T[N]
    w = [0.0] * (N + 1)
    for i in range(N):
        w[i] = T[i] - T[i+1]
    w[N] = T[N]
    
    # 4. Color Gradients
    # grad_c[i] = w[i] for i < N
    grad_c = w[:N] 
    
    # 5. Density Gradients
    grad_sigma = [0.0] * N
    
    for i in range(N):
        # Term 1: c_i * T_{i+1}
        term1 = c[i] * T[i+1]
        
        # Term 2: sum_{k=i+1}^{N} c_k * w_k
        # Note: c_N is c_bg
        sum_term = 0.0
        for k in range(i+1, N):
            sum_term += c[k] * w[k]
        sum_term += c_bg * w[N]
        
        grad_sigma[i] = delta[i] * (term1 - sum_term)
        
    return grad_sigma, grad_c
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The derivative is only defined for $i < N$ because $\sigma_N$ does not exist (only $c_N$) [S1, unofficial]. The background color $c_N$ is typically not optimized [S1, unofficial].
- **SH Colors:** When optimizing SH coefficients, the gradient must be multiplied by the SH basis function value at the query direction [S1, unofficial]. The basis values are constant for a given ray direction [S1, unofficial].

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.

## Background
This method was introduced in the PlenOctrees paper by Yu et al. to enable the fine-tuning of PlenOctrees on original training images using NeRF loss with SGD [S1, unofficial]. It builds on the NeRF volume rendering formulation [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
