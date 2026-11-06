# Heterogeneous Medium Transmittance — Beer's Law

Also called: Beer's Law with Optical Depth, Beer-Lambert Law, General Transmittance Equation, Transmittance (Beer-Lambert Law)

## Overview
Heterogeneous Medium Transmittance — Beer's Law computes the fraction of radiance transmitted between two points through a participating medium where the extinction coefficient varies spatially [S1, S2, unofficial][S3, book]. It generalizes the homogeneous case by integrating the extinction coefficient along the ray path to determine the optical depth [S1, S2, unofficial]. Unlike [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md), which assumes a constant extinction coefficient, this method handles spatially varying media by evaluating the integral of $\sigma_t$ along the ray segment [S1, S2, unofficial]. It serves as the foundational transmittance model for [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) and [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md) [S1, S2, unofficial].

## How it works
**Inputs:**
- Start point $\mathrm{p}$ (3D position).
- End point $\mathrm{p}^\prime$ (3D position).
- Extinction coefficient function $\sigma_t(\mathbf{x}, \omega)$, which returns the extinction coefficient at position $\mathbf{x}$ for direction $\omega$. In many implementations, $\sigma_t$ depends only on position $\mathbf{x}$ [S1, S2, unofficial].

**Outputs:**
- Transmittance $\mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime)$: A scalar in $[0, 1]$ representing the fraction of radiance transmitted [S2, unofficial].
- Optical depth $\tau (\mathrm{p} \to \mathrm{p}^\prime)$: A scalar representing the accumulated extinction along the path [S2, unofficial].

**Data Structures:**
- No specific complex data structures are required for the definition itself. The implementation typically requires a function or lookup table to evaluate $\sigma_t(\mathbf{x})$ at arbitrary 3D coordinates. If $\sigma_t$ is stored in a 3D grid (e.g., a density field), trilinear interpolation is used to sample $\sigma_t$ at non-grid positions [S1, G, unofficial].

**Steps:**
1. Calculate the ray direction $\omega$ and total distance $d$:
   $$ \mathbf{v} = \mathrm{p}^\prime - \mathrm{p} $$
   $$ d = \lVert \mathbf{v} \rVert $$
   $$ \omega = \mathbf{v} / d $$
   If $d = 0$, return $\mathcal{T} = 1$ and $\tau = 0$ [S2, G, unofficial].

2. Define the position along the ray at distance $t$ from $\mathrm{p}$:
   $$ \mathbf{x}(t) = \mathrm{p} + \omega t, \quad t \in [0, d] $$

3. Compute the optical depth $\tau$ by integrating the extinction coefficient along the ray:
   $$ \tau (\mathrm{p} \to \mathrm{p}^\prime) = \int_0^d \sigma_t (\mathbf{x}(t), \omega) \; \mathrm{d}t $$
   In practice, this integral is approximated numerically (e.g., via quadrature or ray marching) if $\sigma_t$ is not analytically integrable [S1, S2, G, unofficial].

4. Compute the transmittance:
   $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) = \exp \left( - \tau (\mathrm{p} \to \mathrm{p}^\prime) \right) $$
   Alternatively, directly:
   $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) = \exp \left( - \int_0^d \sigma_t (\mathrm{p} + \omega t, \omega) \; \mathrm{d}t \right) $$ [S2, unofficial]

**Properties:**
- **Multiplicative Property:** Transmittance is multiplicative along a ray. For three collinear points $\mathrm{p}, \mathrm{p}^\prime, \mathrm{p}^{\prime \prime}$:
  $$ \mathcal{T} (\mathrm{p} \to \mathrm{p}^{\prime \prime}) = \mathcal{T} (\mathrm{p} \to \mathrm{p}^\prime) \mathcal{T} (\mathrm{p}^\prime \to \mathrm{p}^{\prime \prime}) $$
  This follows from the linearity of the integral in the exponent [S2, unofficial].

**Pitfalls:**
- **Numerical Integration:** If $\sigma_t$ varies rapidly, naive sampling (e.g., single-point evaluation) may introduce significant error. Adaptive sampling or higher-order quadrature may be needed [G].
- **Zero Distance:** Division by zero when computing $\omega$ if $\mathrm{p} = \mathrm{p}^\prime$. Handle explicitly [G].
- **Precision:** For very large optical depths, $\exp(-\tau)$ may underflow to 0. This is physically correct (opaque medium) but may cause issues in gradient-based optimization if not handled carefully [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def heterogeneous_transmittance(p, p_prime, sigma_t_func):
    """
    Computes transmittance and optical depth for a ray from p to p_prime
    through a heterogeneous medium defined by sigma_t_func.
    
    Args:
        p: 3D vector, start point
        p_prime: 3D vector, end point
        sigma_t_func: function(x, omega) -> float, returns extinction coefficient at x
        
    Returns:
        T: float, transmittance
        tau: float, optical depth
    """
    v = p_prime - p
    d = length(v)
    
    if d == 0:
        return 1.0, 0.0
    
    omega = v / d
    
    # Numerical integration of sigma_t along the ray
    # Using simple midpoint rule with N steps for demonstration
    # In practice, N should be chosen based on required accuracy and sigma_t variation
    N = 100
    dt = d / N
    tau = 0.0
    
    for i in range(N):
        t = (i + 0.5) * dt
        x = p + omega * t
        sigma_t = sigma_t_func(x, omega)
        tau += sigma_t * dt
        
    T = exp(-tau)
    return T, tau
```

## Subtopics

### Spectral Volume Rendering (Lambert-Bouguer Basis)
This variant treats visible light as electromagnetic radiation with a spectral power distribution $F(\lambda)$, typically limited to $\lambda \in [380nm, 770nm]$ or $[400nm, 700nm]$ [S3, book]. The transmittance is computed per wavelength $\lambda$ using the spectral extinction coefficient $\tau(\lambda)$ (absorptivity) [S3, book]. For a homogeneous medium, the transmitted intensity is $L(\lambda) = L_0(\lambda) \cdot \exp(-s \cdot \tau(\lambda))$ [S3, book]. Beer's extension for a solution with solute concentration $\nu$ and solute extinction coefficient $\kappa(\lambda)$ is $L(\lambda) = L_0(\lambda) \cdot \exp(-s \cdot \nu \cdot \kappa(\lambda))$ [S3, book]. A key feature is the depth correction formula: if transparency $T(\lambda, \Delta)$ is known for a standard path length $\Delta$, the transparency for an arbitrary path length $\delta$ is $T(\lambda, \delta) = T(\lambda, \Delta)^{\frac{\delta}{\Delta}}$ [S3, book]. This is often expressed as opacity $\alpha(\lambda, \delta) = 1 - (1 - \alpha(\lambda, \Delta))^{\frac{\delta}{\Delta}}$ [S3, book]. This model is a special case of the general Beer's Law where homogeneity is assumed and spectral dependence is explicit [S3, book].

## Operating characteristics
- **Time Complexity:** The computation of optical depth requires numerical integration. If the ray is sampled at $N$ points, the complexity is $O(N)$, where $N$ depends on the desired accuracy and the variation of $\sigma_t$ [S1, S2, G, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory, plus the memory required to store or evaluate $\sigma_t(\mathbf{x})$ [S1, S2, unofficial].
- **Scaling:** Performance scales linearly with the number of integration steps. For complex media, adaptive sampling can reduce $N$ in regions of low $\sigma_t$ variation [G].
- **Strengths:** Physically accurate for heterogeneous media; handles spatially varying extinction; multiplicative property allows efficient composition of transmittance over multiple segments [S2, unofficial].
- **Limitations:** Requires evaluation of $\sigma_t$ at many points along the ray, which can be expensive if $\sigma_t$ is complex or stored in a large 3D texture [G]. Numerical integration errors can occur if sampling is insufficient [G].
- **When to use:** When the medium's extinction coefficient varies significantly in space (e.g., clouds, smoke, biological tissue) [S1, S2, unofficial].
- **When not to use:** When the medium is homogeneous (use [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md) for efficiency) or when the extinction coefficient is constant along the ray [S2, unofficial].

## Background
The Beer-Lambert law originates from the work of Bouguer (1727) and Lambert (1760) on the transmission of light through filters, and was extended by Beer (1983) to liquid solutions [S3, book]. In computer graphics, it is generalized to heterogeneous media by integrating the extinction coefficient along the ray, forming the basis of the volume rendering equation [S1, S2, unofficial]. The concept of optical depth $\tau$ is central to radiative transfer theory [S1, S2, unofficial].

## Sources
[S1] From the Radiative Transfer Equation to the Volume Rendering Equation, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-summary-equations.html, fetched 2026-10-25 (unofficial)
[S2] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
