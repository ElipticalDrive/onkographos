# Universal Beta Splatting (UBS)
Also called: Beta kernel, N-dimensional Beta Kernel, Universal Beta Splatting (UBS) / N-dimensional Beta Kernel

## Overview
Universal Beta Splatting (UBS) is an N-dimensional splatting kernel that replaces the Gaussian distribution with a Beta function to enable independent per-dimension shape control [S1, unofficial]. Unlike Gaussian kernels where all dimensions are coupled through a joint distribution, UBS allows each dimension to adopt fundamentally different profiles (e.g., flat vs. peaked) independently, addressing the limitations of methods like 6DGS and 7DGS [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A point $\mathbf{x} \in \mathbb{R}^N$, mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, and shape parameters $\mathbf{b} = [b_1, \dots, b_N]$.
*   **Output:** A scalar kernel value $\mathcal{B}(\mathbf{x}) \in [0, 1]$.

**Data Structures**
*   $\mathbf{b}$: An array of $N$ floating-point values representing the shape parameters for each dimension.
*   $d_i(\mathbf{x})$: The bounded distance for dimension $i$, computed from the spatial/angular/temporal coordinates relative to the mean.

**Formulas**
The N-dimensional Beta kernel is defined as a gated product:
$$ \mathcal{B}(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, \mathbf{b}) = \prod_{i=1}^N (1 - d_i(\mathbf{x}))^{\beta_i} $$
where $d_i \in [0, 1)$ is the bounded distance and $\beta_i$ is the shape exponent [S1, unofficial].

The explicit per-dimension shape control formula is:
$$ \mathcal{B}(\mathbf{x}) = \prod_{i=1}^N (1 - d_i(\mathbf{x}))^{4 \exp(b_i)} $$
where $b_i$ is the shape parameter for dimension $i$ [S1, unofficial].

The bounded distance $d_i(\mathbf{x})$ is defined differently based on the dimension index:
1.  For the first three dimensions ($i = 1, 2, 3$), $d_i$ is defined by $r_i$ in Equation [1] of the source [S1, unofficial].
2.  For the remaining dimensions ($i = 4, \dots, N$), $d_i$ is defined by Equation 11 of the source [S1, unofficial].

**Parameters**
*   $b_i$: Shape parameter controlling the kernel profile. Typical practical range is $b_i \in [-5, 5]$ [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Flat Kernel:** When $b_i < 0$, the kernel profile becomes flatter with broader support ($\beta_i < 4$). Specifically, when $b_i = -5$, $\beta_i \approx 0.027$ and the kernel becomes nearly constant ($ (1-d_i)^{0.027} \approx 1 $) [S1, unofficial].
*   **Peaked Kernel:** When $b_i > 0$, the kernel profile becomes more peaked with sharper localization ($\beta_i > 4$). Specifically, when $b_i = 5$, $\beta_i \approx 594$ [S1, unofficial].
*   **Gaussian Approximation:** When $b_i = 0$, $\beta_i = 4$, and the Beta kernel approximates a Gaussian via the relation $(1 - d)^4 \approx \exp(-4.5d^2)$ for $d \in [0, 1)$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def universal_beta_splatting(x, mu, sigma, b):
    """
    Computes the Universal Beta Splatting kernel value.
    
    Args:
        x: Input point (N-dimensional vector).
        mu: Mean vector (N-dimensional).
        sigma: Covariance matrix (N x N).
        b: Shape parameters vector (N-dimensional).
        
    Returns:
        Kernel value.
    """
    # Compute bounded distances d_i for each dimension
    # Note: The exact computation of d_i depends on the specific 
    # definitions in Equation [1] and Equation 11 of the source.
    # This implementation assumes a helper function `compute_bounded_distance`
    # is available which handles the specific metric for spatial vs 
    # non-spatial dimensions as described in the source.
    
    N = len(x)
    kernel_value = 1.0
    
    for i in range(N):
        # Compute bounded distance d_i
        # For i < 3: d_i uses spatial metric (Eq 1)
        # For i >= 3: d_i uses non-spatial metric (Eq 11)
        d_i = compute_bounded_distance(x[i], mu[i], sigma, i)
        
        # Clamp d_i to [0, 1) to ensure numerical stability
        d_i = max(0.0, min(d_i, 0.999999))
        
        # Compute shape exponent beta_i
        beta_i = 4.0 * math.exp(b[i])
        
        # Multiply kernel component
        kernel_value *= (1.0 - d_i) ** beta_i
        
    return kernel_value

def compute_bounded_distance(val, mean, sigma, dim_index):
    """
    Computes the bounded distance d_i.
    This is a placeholder for the specific logic in Eq [1] and Eq 11.
    In a full implementation, this would involve Mahalanobis distance 
    normalization and bounding.
    """
    # Placeholder for source-specific distance calculation
    # Typically involves: d = sqrt((x-mu)^T Sigma^-1 (x-mu))
    # followed by a bounding function to map to [0, 1)
    pass
```

## Subtopics

### Deformable Beta Splatting
An extension that allows the Beta kernel to deform, adapting to complex geometries. It builds on the UBS framework by introducing deformation parameters to the kernel shape. [Link](9228-deformable-beta-splatting.md)

## Operating characteristics
*   **Complexity**: The kernel evaluation is $O(N)$ where $N$ is the number of dimensions, due to the product form.
*   **Strengths**: Enables independent modeling of spatial, angular, and temporal properties. A single primitive can represent static spatial surfaces with view-dependent reflectance, which Gaussian methods require multiple coupled primitives to approximate [S1, unofficial].
*   **Limitations**: Requires careful tuning of $b_i$ parameters. Extreme values of $b_i$ can lead to numerical instability or overly flat/peaked kernels that may not represent the data well [S1, unofficial].
*   **Performance**: Not given in the sources.

## Background
UBS was introduced to address the inherent dimensional coupling limitations in 6DGS and 7DGS, where changes in view direction or time affect all dimensions simultaneously [S1, unofficial]. The method uses the Beta function to replace the Normal distribution, offering smaller footprints and easier tuning for visual parameters like opacity and zoom [S1, unofficial].

## Sources
[S1] Universal Beta Splatting, https://doi.org/10.48550/arXiv.2510.03312, fetched 2026-10-30 (unofficial)
