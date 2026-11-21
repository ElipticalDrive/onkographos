# 2D Gaussian Probability Density (Conic)
Also called: 2D Gaussian Probability Density from Conic and Rescale
## Overview
Computes the unnormalized probability density function (PDF) value of a 2D Gaussian at a specific 2D point `xy`, given the Gaussian's mean and a conic matrix (the inverse covariance matrix) along with a rescaling factor [S1, unofficial]. It differs from its parent [2D Gaussian Probability Density (Standard)](3466-2d-gaussian-probability-density-standard.md) by operating directly on the conic representation and including a multiplicative rescaling term [G].
## How it works
**Inputs:**
- `xy`: A 2D point $(x, y)$ where the density is evaluated [S1, unofficial].
- `gaussian_mean`: The mean vector $(\mu_x, \mu_y)$ of the 2D Gaussian [S1, unofficial].
- `conic_and_rescale`: A 4D vector containing the conic matrix components and the rescaling factor [S1, unofficial]. The layout is:
  - `x`: $C_{00}$ (inverse covariance element $(0,0)$) [S1, unofficial].
  - `y`: $C_{01}$ (inverse covariance element $(0,1)$) [S1, unofficial].
  - `z`: $C_{11}$ (inverse covariance element $(1,1)$) [S1, unofficial].
  - `w`: $R$ (rescaling factor) [S1, unofficial].

**Outputs:**
- `p`: The unnormalized probability density value [S1, unofficial].

**Data Structures:**
The `conic_and_rescale` vector is typically stored as a single 4-component vector (e.g., `vec4` in GLSL, `ti.Vector4f` in Taichi, or a struct of four floats) [G]. The components are accessed by index or named fields as defined above [G].

**Steps:**
1. Compute the difference vector $\Delta = xy - gaussian\_mean$ [S1, unofficial].
   $$ \Delta x = xy.x - gaussian\_mean.x $$
   $$ \Delta y = xy.y - gaussian\_mean.y $$
2. Compute the exponent term using the conic matrix components and the difference vector [S1, unofficial]. The quadratic form is:
   $$ E = -0.5 (\Delta x^2 C_{00} + \Delta y^2 C_{11}) - \Delta x \Delta y C_{01} $$
3. Compute the exponential of the exponent term [S1, unofficial].
   $$ \text{exp\_val} = \exp(E) $$
4. Multiply by the rescaling factor $R$ [S1, unofficial].
   $$ p = \text{exp\_val} \cdot R $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_2d_gaussian_pdf_conic(xy, gaussian_mean, conic_and_rescale):
    # Step 1: Compute difference vector
    dx = xy.x - gaussian_mean.x
    dy = xy.y - gaussian_mean.y
    
    # Step 2: Compute exponent term
    C00 = conic_and_rescale.x
    C01 = conic_and_rescale.y
    C11 = conic_and_rescale.z
    
    # Quadratic form: -0.5 * (dx^2 * C00 + dy^2 * C11) - dx * dy * C01
    exponent = -0.5 * (dx * dx * C00 + dy * dy * C11) - dx * dy * C01
    
    # Step 3: Exponential
    exp_val = math.exp(exponent)
    
    # Step 4: Rescale
    R = conic_and_rescale.w
    p = exp_val * R
    
    return p
```

**Parameters:**
- $C_{00}, C_{01}, C_{11}$: Elements of the inverse covariance matrix (conic matrix) [S1, unofficial]. These must be derived from the 3D Gaussian's covariance matrix projected to 2D and inverted [G].
- $R$: Rescaling factor [S1, unofficial]. Often related to opacity or normalization constants [G].

**Edge Cases and Pitfalls:**
- **Precision:** The exponent can become very large negative numbers, leading to underflow to zero [G]. This is expected behavior for points far from the mean [G].
- **Conic Matrix Validity:** The conic matrix must be positive definite for the Gaussian to be well-defined [G]. If the matrix is singular or negative definite, the result may be invalid or unstable [G].
- **Rescaling Factor:** If $R=0$, the result is always 0 [G]. If $R$ is very large, it may cause overflow if the exponent is close to 0 [G].

## Subtopics

### [2D Gaussian Evaluation](4728-2d-gaussian-evaluation.md)
Computes the exponent term of the 2D Gaussian probability density function at given 2D points, used for rasterization or shading. It differs from its parent [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md) by returning only the unnormalized exponent value $P$ rather than the full normalized density, allowing the caller to apply normalization factors or opacity scaling separately [G].

## Operating characteristics
Not given in the sources.

## Background
This method is part of the Gaussian Splatting rendering pipeline [S1, unofficial]. The conic representation (inverse covariance matrix) is used in rasterization [S1, unofficial]. The rescaling factor is introduced to handle opacity and normalization [S1, unofficial]. The implementation is based on the Taichi 3D Gaussian Splatting library [S1, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
