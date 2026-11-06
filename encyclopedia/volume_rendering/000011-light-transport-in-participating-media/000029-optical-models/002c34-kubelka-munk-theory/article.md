# Kubelka-Munk Theory

Also called: KM theory, Kubelka-Munk model, One-dimensional Radiosity (Kubelka-Munk)

## Overview
Kubelka-Munk Theory is a spectral optical model for light transport in scattering media that computes reflectance and transmittance by accounting for both absorption and scattering [S1, book]. It solves the problem of rendering homogeneous and inhomogeneous layers where simple extinction (Lambert-Bouguer law) is insufficient because it ignores the redirection of light [S1, book]. Unlike the [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) which models full angular scattering, Kubelka-Munk restricts light transport to two opposing fluxes (incident and reflected) along a single axis, making it suitable for spectral volume rendering of layered materials [S1, book].

## How it works
**Inputs and Outputs**
*   **Inputs:** Absorption coefficient $K(\lambda)$, scattering coefficient $S(\lambda)$, layer thickness $X$, and wavelength $\lambda$.
*   **Outputs:** Reflectance $R(\lambda)$ and Transmittance $T(\lambda)$.

**Data Structures**
The method operates on scalar spectral values per layer. For inhomogeneous media, layers are represented as a sequence of tuples $(K_i, S_i, X_i)$.

**Formulas and Definitions**
The theory divides the volume into homogeneous elementary layers of thickness $dx$. It defines two fluxes: incident flux $I_i(\lambda)$ flowing in direction $x$, and reflected flux $I_r(\lambda)$ flowing in direction $-x$ [S1, book].

The change in flux through an elementary layer is governed by:
$$ dI_i(\lambda) = -K(\lambda)I_i(\lambda)dx - S(\lambda)I_i(\lambda)dx + S(\lambda)I_r(\lambda)dx \quad (7) $$
$$ dI_r(\lambda) = -K(\lambda)I_r(\lambda)dx - S(\lambda)I_r(\lambda)dx + S(\lambda)I_i(\lambda)dx \quad (8) $$

Note: Source [S1, book] contains a typo in equation (8) where the backscatter term is written as $S(\lambda)I_r(\lambda)dx$; physically, it must be $S(\lambda)I_i(\lambda)dx$ to represent backscattering from the incident flux, consistent with equation (7) and the description "energy is also increased due to backscatter" [S1, book].

For a layer of finite thickness $X$, the reflectance $R(\lambda)$ and transmittance $T(\lambda)$ are calculated using auxiliary parameters $a(\lambda)$ and $b(\lambda)$:
$$ a(\lambda) = \frac{S(\lambda) + K(\lambda)}{S(\lambda)} $$
$$ b(\lambda) = \sqrt{a(\lambda)^2 - 1} $$
$$ R(\lambda) = \frac{b(\lambda) \sinh(b(\lambda)S(\lambda)X)}{a(\lambda) \sinh(b(\lambda)S(\lambda)X) + b(\lambda) \cosh(b(\lambda)S(\lambda)X)} \quad (9) $$
$$ T(\lambda) = \frac{b(\lambda)}{a(\lambda) \sinh(b(\lambda)S(\lambda)X) + b(\lambda) \cosh(b(\lambda)S(\lambda)X)} \quad (10) $$

The reflectance of an opaque medium ($X \to \infty$) is:
$$ R_\infty(\lambda) = 1 + \frac{K(\lambda)}{S(\lambda)} - \sqrt{\left(\frac{K(\lambda)}{S(\lambda)}\right)^2 + 2\frac{K(\lambda)}{S(\lambda)}} $$
*Note: Source [S1, book] presents the square root term as squared in the text description but the square root symbol in the formula block. The standard Kubelka-Munk equation uses the square root [S1, book].*

**Inhomogeneous Layers (Composite Layers)**
For inhomogeneous layers, the theory combines two adjacent layers with properties $(R_1, T_1)$ and $(R_2, T_2)$ by considering infinite interactions between them [S1, book]:
$$ R(\lambda) = R_1(\lambda) + \frac{T_1(\lambda)^2 R_2(\lambda)}{1 - R_1(\lambda)R_2(\lambda)} $$
$$ T(\lambda) = \frac{T_1(\lambda)T_2(\lambda)}{1 - R_1(\lambda)R_2(\lambda)} $$

**Algorithm: Spectral Volume Rendering**
To render a volume, the intersection of a ray with the volume is approximated as a series of homogeneous layers with thickness equal to the sampling distance [S1, book].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kubelka_munk_layer(K, S, X):
    """
    Computes R and T for a single homogeneous layer.
    K: absorption coefficient
    S: scattering coefficient
    X: thickness
    """
    if S == 0:
        # Fallback to Lambert-Bouguer if no scattering
        # T = exp(-(K+S)*X), R = 0
        T = exp(-(K + S) * X)
        R = 0.0
        return R, T

    a = (S + K) / S
    b = sqrt(a**2 - 1)
    
    # Avoid division by zero or numerical instability for very thin layers
    if X == 0:
        return 0.0, 1.0

    sinh_val = sinh(b * S * X)
    cosh_val = cosh(b * S * X)
    
    denominator = a * sinh_val + b * cosh_val
    
    R = (b * sinh_val) / denominator
    T = b / denominator
    
    return R, T

def composite_layers(R1, T1, R2, T2):
    """
    Combines two layers.
    """
    # Guard against 1 - R1*R2 = 0
    denom = 1.0 - R1 * R2
    if abs(denom) < 1e-9:
        # Degenerate case, return conservative estimates or handle error
        return 1.0, 0.0 
        
    R_new = R1 + (T1**2 * R2) / denom
    T_new = (T1 * T2) / denom
    return R_new, T_new

def render_volume_layered(K_samples, S_samples, X_samples):
    """
    Renders a volume by stacking layers.
    K_samples: list of K for each layer
    S_samples: list of S for each layer
    X_samples: list of thicknesses for each layer
    """
    # Start with background (opaque black or white depending on context, 
    # typically treated as a final layer or base transmittance)
    # Here we assume we are computing the total R and T of the stack.
    
    # Initialize with the first layer
    R_curr, T_curr = kubelka_munk_layer(K_samples[0], S_samples[0], X_samples[0])
    
    # Iteratively combine with subsequent layers
    for i in range(1, len(K_samples)):
        R_next, T_next = kubelka_munk_layer(K_samples[i], S_samples[i], X_samples[i])
        R_curr, T_curr = composite_layers(R_curr, T_curr, R_next, T_next)
        
    return R_curr, T_curr
```

**Edge Cases and Pitfalls**
*   **S=0:** If scattering is zero, the model reduces to the Lambert-Bouguer law (extinction only). The formulas for $a$ and $b$ become undefined; use exponential decay [S1, book].
*   **Numerical Stability:** For very large optical depths, $\sinh$ and $\cosh$ may overflow. Use logarithmic forms or asymptotic limits ($R \to R_\infty$) [S1, book].
*   **Directionality:** The model assumes isotropic scattering within the two-flux approximation. It does not account for anisotropic phase functions [S1, book].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of layers/samples along the ray. Each layer requires constant time evaluation of hyperbolic functions.
*   **Memory Complexity:** $O(1)$ per layer if processed iteratively, or $O(N)$ if storing all layer properties.
*   **Strengths:** Built-in distance attenuation; determines opacity and transparency optically based on $K$ and $S$; suitable for spectral rendering [S1, book].
*   **Limitations:** Restricted to one-dimensional transport (no angular resolution); less accurate for highly anisotropic scattering compared to full RTE solvers [S1, book].
*   **When to use:** Spectral volume rendering of layered, scattering media (e.g., skin, paper, paint) where computational efficiency is prioritized over full angular accuracy [S1, book].
*   **When not to use:** Scenes requiring complex shadowing, caustics, or strong directional scattering effects [S1, book].

## Background
The Kubelka-Munk theory was originally proposed by Kubelka and Munk [34] and built upon Schuster's two-flux concept [51]. It was extended to inhomogeneous layers by Kubelka [33]. In computer graphics, Abdul-Rahman and Chen [1] adapted it for spectral volume rendering, presenting a spectral volume rendering integral that differs from the Lambert-Bouguer law by modeling absorption and scattering in only two directions (incident and reflected) [S1, book].

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
