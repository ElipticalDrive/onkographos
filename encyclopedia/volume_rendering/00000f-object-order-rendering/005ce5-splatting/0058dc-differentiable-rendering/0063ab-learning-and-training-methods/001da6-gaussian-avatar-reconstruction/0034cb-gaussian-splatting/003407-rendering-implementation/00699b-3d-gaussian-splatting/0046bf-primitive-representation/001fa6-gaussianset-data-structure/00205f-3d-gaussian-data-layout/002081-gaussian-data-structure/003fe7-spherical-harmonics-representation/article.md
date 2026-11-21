# Spherical Harmonics Representation

Also called: SH in 3DGS, Spherical Harmonics (SH) in 3DGS SLAM, Spherical Harmonics Algorithm, View-dependent color via SH

## Overview
Spherical Harmonics (SH) is a mathematical basis used in 3D Gaussian Splatting to represent the view-dependent color of each Gaussian primitive [S1, S2, unofficial]. It solves the problem of modeling directional lighting and material properties that vary with the camera viewing angle, which a single static RGB value cannot capture [S1, unofficial]. Unlike static color representations, SH allows for an analytical Jacobian that propagates gradients from the SH coefficients to camera poses during optimization, enabling joint optimization of geometry and appearance [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- The viewing direction vector $\mathbf{v}$ (normalized) in the local coordinate system of the Gaussian.
- The SH degree parameter $L$ (denoted as `sh_degree` in implementation), which determines the maximum degree $l$ used.
- The SH coefficients $c_{l,m}^k$ for each color channel $k \in \{R, G, B\}$.

**Outputs:**
- The view-dependent color $\mathbf{C}(\mathbf{v}) = [C_R, C_G, C_B]^T$.

### Data Structures
Each 3D Gaussian stores SH coefficients. For a maximum degree $L$, the number of coefficients per color channel is $(L+1)^2$. The total number of coefficients per Gaussian is $3(L+1)^2$.
- **Layout:** The coefficients are typically stored as a flat array or a tensor of shape $[3, (L+1)^2]$ or $[(L+1)^2, 3]$.
- **Bit Packing:** Not specified in the sources. Standard floating-point representation (e.g., FP16 or FP32) is implied by the optimization context [S2, unofficial].

### Mathematical Foundation
Spherical Harmonics are derived from the solution of Laplace's equation in a 3D spherical coordinate system [S1, unofficial].

1.  **Laplace's Equation:**
    In Cartesian coordinates:
    $$ \frac{\partial^2 f}{\partial x^2} + \frac{\partial^2 f}{\partial y^2} + \frac{\partial^2 f}{\partial z^2} = 0 $$
    Converted to spherical coordinates $(r, \theta, \varphi)$ where $x = r\sin\theta\cos\varphi$, $y = r\sin\theta\sin\varphi$, $z = r\cos\theta$:
    $$ \frac{1}{r^{2}}\frac{\partial }{\partial r}\left(r^{2}\frac{\partial f}{\partial r}\right) + \frac{1}{r^{2}\sin\theta } \frac{\partial }{\partial \theta }\left(\sin\theta \frac{\partial f}{\partial \theta }\right) + \frac{1}{r^{2}\sin^{2}\theta } \frac{\partial ^{2} f}{\partial \varphi ^{2}} = 0 $$

2.  **Separation of Variables:**
    Assume $f(r, \theta, \varphi) = R(r)\Theta(\theta)\Phi(\varphi)$. This yields:
    - Radial part: $\frac{1}{R}\frac{\partial}{\partial r}\left(r^{2}\frac{\partial R}{\partial r}\right) = \lambda$
    - Azimuthal part: $\frac{\partial^{2}\Phi}{\partial \varphi^{2}} = -m^{2}\Phi$, where $m \in \mathbb{N}$
    - Polar part: $\sin\theta \frac{\partial}{\partial \theta}\left(\sin\theta \frac{\partial \Theta}{\partial \theta}\right) = (m^{2}-\lambda \sin^{2}\theta)\Theta$

3.  **Associated Legendre Equation:**
    Let $x = \cos\theta$, $\Theta \rightarrow P(x)$, and $\lambda = l(l+1)$ where $l \in \mathbb{Z}^+$. The equation becomes:
    $$ \frac{\partial}{\partial x}\left[(1-x^{2})\frac{\partial P(x)}{\partial x}\right] + \left[l(l+1)-\frac{m^{2}}{1-x^{2}}\right]P(x) = 0 $$
    The solution is the Associated Legendre Function $P_l^m(x)$. Rodrigues' formula is given as:
    $$ P_{l}(x)=\frac{1}{2^{l}l!}\frac{\partial ^{l}}{\partial x^{l}}(x^2-1)^{l} $$
    Orthonormality condition:
    $$ \int_{-1}^{1}\sqrt{\frac{2m+1}{2}}P_{m}(x) \sqrt{\frac{2n+1}{2}}P_{n}(x)dx=\delta_{mn} $$
    Recursion relation:
    $$ (l+1)P_{l+1}(x)=(2l+1)xP_{l}(x)-lP_{l-1}(x) $$

4.  **Spherical Harmonics Function:**
    The SH basis functions $Y_l^m(\theta, \varphi)$ are defined as:
    $$ Y_l^m(\theta, \varphi) = \begin{cases} \sqrt{2} K_{l}^{m}\cos(m\varphi)P_{l}^{m}(\cos\theta) & m > 0 \\ \sqrt{2} K_{l}^{m}\sin(-m\varphi)P_{l}^{-m}(\cos\theta) & m < 0 \\ K_{l}^{0}P_{l}^{0}(\cos\theta) & m = 0 \end{cases} $$
    The normalization scaling factor $K_{l}^{m}$ is:
    $$ K_{l}^{m}=\sqrt{\frac{2l+1}{4\pi}\cdot \frac{(l-|m|)!}{(l+|m|)!}} $$
    where $l \in \mathbb{Z}^{+}, m \in \mathbb{Z}, |m| \le l$.

### Rendering Formula
The color $\mathbf{C}(\mathbf{v})$ for a viewing direction $\mathbf{v}$ (represented by spherical angles $\theta, \varphi$) is computed by projecting the SH coefficients onto the basis functions:
$$ \mathbf{C}(\theta, \varphi) = \sum_{l=0}^{L} \sum_{m=-l}^{l} c_{l,m} Y_l^m(\theta, \varphi) $$
In practice, this is often implemented using a pre-computed lookup table or efficient polynomial evaluation for the SH basis functions to avoid expensive trigonometric and factorial calculations during rendering [S1, unofficial].

### Parameters
- **`sh_degree` ($L$):** The maximum degree of SH coefficients. Default value is 3 [S1, unofficial].
    - $L=0$: View-independent color (constant).
    - $L=1$: Captures linear variations (e.g., simple lighting).
    - $L=2, 3$: Captures more complex view-dependent effects.
- **Omission:** SH can be omitted (set $L=0$ or disabled) for synthetic datasets like Replica that lack view-directional effects [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_sh_basis(theta, phi, max_degree):
    """
    Computes the Spherical Harmonics basis functions for a given direction.
    Returns a list of basis values for degrees 0 to max_degree.
    """
    basis = []
    cos_theta = math.cos(theta)
    sin_theta = math.sin(theta)
    
    # Precompute factorials if needed, or use math.factorial
    # Note: Efficient implementations often use recursion or precomputed tables.
    
    for l in range(max_degree + 1):
        for m in range(-l, l + 1):
            # Compute Associated Legendre Function P_l^m(cos_theta)
            # Using Rodrigues' formula or recursion.
            # P_l^m(x) = (-1)^m * (1-x^2)^(m/2) * d^m/dx^m P_l(x)
            # For implementation, standard recursion is preferred.
            p_lm = associated_legendre(l, abs(m), cos_theta)
            
            # Compute normalization factor K_l^m
            k_lm = math.sqrt((2 * l + 1) / (4 * math.pi) * 
                             math.factorial(l - abs(m)) / math.factorial(l + abs(m)))
            
            # Compute angular part
            if m > 0:
                angular = math.sqrt(2) * math.cos(m * phi)
            elif m < 0:
                angular = math.sqrt(2) * math.sin(-m * phi)
            else: # m == 0
                angular = 1.0
            
            basis.append(k_lm * p_lm * angular)
            
    return basis

def associated_legendre(l, m, x):
    """
    Computes the Associated Legendre Function P_l^m(x).
    Uses recursion for efficiency.
    """
    if m < 0:
        # P_l^{-m} = (-1)^m * (l-m)!/(l+m)! * P_l^m
        # However, standard SH definitions handle signs in the basis function.
        # We compute P_l^{|m|} and handle sign in the caller or here.
        # Standard recurrence usually computes for m >= 0.
        return associated_legendre(l, -m, x) # Placeholder for sign handling logic
    
    if l == 0:
        return 1.0 if m == 0 else 0.0
    if l < m:
        return 0.0
        
    # Compute P_l^m using recurrence relations
    # P_0^0 = 1
    # P_1^0 = x
    # P_l^m = (1-2m) * ... (complex recurrence)
    # Simplified implementation using standard library or precomputed table is recommended.
    # Here is a basic recursive definition for demonstration:
    if m == 0:
        if l == 0: return 1.0
        if l == 1: return x
        # P_l^0 = ((2l-1)x P_{l-1}^0 - (l-1) P_{l-2}^0) / l
        return ((2*l - 1) * x * associated_legendre(l-1, 0, x) - (l-1) * associated_legendre(l-2, 0, x)) / l
    else:
        # P_m^m = (2m-1)!! * (1-x^2)^(m/2)
        if l == m:
            if m == 0: return 1.0
            # (2m-1)!! = product of odd numbers up to 2m-1
            double_factorial = 1
            for i in range(1, 2*m, 2):
                double_factorial *= i
            return double_factorial * ((1 - x*x)**(m/2))
        # P_l^m = ((2l-1)x P_{l-1}^m - (l+m-1) P_{l-2}^m) / (l-m)
        return ((2*l - 1) * x * associated_legendre(l-1, m, x) - (l+m-1) * associated_legendre(l-2, m, x)) / (l - m)

def render_sh(color_coeffs, theta, phi, max_degree):
    """
    Renders the view-dependent color using SH coefficients.
    color_coeffs: list of coefficients for R, G, B channels.
                  Shape: [3, (max_degree+1)^2]
    """
    basis = compute_sh_basis(theta, phi, max_degree)
    color = [0.0, 0.0, 0.0]
    
    # Map basis indices to coefficient indices
    # Basis is ordered by l, then m.
    # Coefficients are typically stored in the same order.
    
    idx = 0
    for l in range(max_degree + 1):
        for m in range(-l, l + 1):
            val = basis[idx]
            for c in range(3):
                color[c] += color_coeffs[c][idx] * val
            idx += 1
            
    return color
```

### Edge Cases and Pitfalls
- **Precision:** Higher degrees ($L > 3$) increase computational cost and memory usage without guaranteed quality improvement [S1, unofficial].
- **Boundary Conditions:** The SH basis is defined on the unit sphere. Directions must be normalized.
- **Degenerate Input:** If the viewing direction is undefined (e.g., at the poles where $\theta=0$ or $\pi$), the azimuthal angle $\varphi$ is undefined. The SH basis functions handle this via $\sin(m\varphi)$ and $\cos(m\varphi)$ terms which remain well-behaved or zero for $m \neq 0$ [S1, unofficial].
- **Overfitting:** Using high SH degrees on datasets without view-dependent effects (like Replica) may lead to overfitting or unnecessary memory usage [S1, S2, unofficial].

## Subtopics

### [GSPull](2773-gspull.md)
A method for optimizing Gaussian parameters, potentially interacting with SH gradients.

### [Composite Error Map and Densification/Pruning Scoring](3396-composite-error-map-and-densification-pruning-scoring.md)
Uses rendering error, which depends on SH accuracy, to guide densification.

### [Dynamic Buffer Management](3753-dynamic-buffer-management.md)
Manages memory for SH coefficients, which scale with $(L+1)^2$.

### [GS-W Network Architecture](5927-gs-w-network-architecture.md)
May use SH as an input or output representation.

### [VastGaussian](6000-vastgaussian.md)
Scales 3DGS, likely optimizing SH storage or computation.

### [Data Layout and Storage](6235-data-layout-and-storage.md)
Details on how SH coefficients are stored in memory.

### [SLAM and Tracking](6236-slam-and-tracking.md)
Uses SH gradients for camera pose optimization.

### [7D Gaussian Splatting](11462-7d-gaussian-splatting.md)
Extends Gaussian representation, possibly including SH.

### [Deformation and Dynamic Modeling](11474-deformation-and-dynamic-modeling.md)
Models time-varying SH coefficients.

## Operating characteristics
- **Time Complexity:** Rendering time increases with SH degree $L$. The number of basis functions is $(L+1)^2$. For $L=3$, this is 16 coefficients per channel. Evaluation involves trigonometric functions and polynomial evaluations.
- **Memory Complexity:** Memory per Gaussian increases by $3(L+1)^2$ floats. For $L=3$, this is 48 floats (192 bytes for FP32) per Gaussian.
- **Measured Performance:**
    - On TUM RGB-D, enabling SH increases memory from 3.97 MB to 11.47 MB [S2, unofficial].
    - Rendering PSNR improves from 21.89 to 24.37 with SH enabled [S2, unofficial].
    - ATE RMSE marginally worsens from 1.47 cm to 1.56 cm with SH enabled [S2, unofficial].
- **Strengths:** Accurately models view-dependent color; enables joint optimization of camera poses via analytical gradients [S2, unofficial].
- **Limitations:** Increases memory usage significantly; may degrade trajectory estimation if SH incorrectly models non-view-dependent effects caused by camera motion [S2, unofficial]. Higher degrees do not always yield better results and increase computational cost [S1, unofficial].
- **When to Use:** When view-dependent effects (specularities, complex lighting) are present.
- **When Not to Use:** For synthetic datasets without view-dependent effects (e.g., Replica) or when memory is severely constrained [S1, S2, unofficial].

## Background
Spherical Harmonics were introduced as a solution to Laplace's equation in spherical coordinates [S1, unofficial]. In the context of 3D Gaussian Splatting, SH was adopted to represent view-dependent color, building on the original 3DGS method [S1, S2, unofficial]. The method has been widely used in subsequent works, including 3DGS SLAM, dynamic 3DGS, and human 3DGS generation [S1, unofficial].

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
