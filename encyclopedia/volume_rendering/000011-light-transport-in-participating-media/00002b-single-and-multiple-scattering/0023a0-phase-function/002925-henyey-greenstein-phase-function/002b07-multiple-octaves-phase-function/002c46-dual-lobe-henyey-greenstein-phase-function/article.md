# Dual-Lobe Henyey-Greenstein Phase Function
Also called: Dual Phase Function, Dual-Lobe Henyey-Greenstein Scattering, Dual-lobe HG, Dual-lobe angular phase function

## Overview
The Dual-Lobe Henyey-Greenstein Phase Function models light scattering in participating media, specifically clouds, by combining a forward-scattering lobe and a backward-scattering lobe to create a more realistic angular distribution than a single lobe [S1, unofficial]. It sets itself apart from the parent [Multiple Octaves Phase Function](4939-multiple-octaves-phase-function.md) by specifically using a two-lobed structure (one forward, one backward) rather than a general composite of multiple octaves, providing a computationally efficient approximation of Mie scattering with significant back-scatter [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: Scattering angle $\theta$ (or its cosine, $\cos\theta$).
- Output: Phase function value $P(\theta)$, a scalar probability density.

**Data Structures**
No complex data structures are required. The method relies on scalar parameters:
- $g_{\text{forward}}$: Forward asymmetry parameter.
- $g_{\text{backward}}$: Backward asymmetry parameter.
- $w$: Blend weight for the forward lobe.

**Parameters**
Typical values for cloud rendering, as observed in production implementations:
- $g_{\text{forward}} \approx 0.82$ [S1, unofficial]
- $g_{\text{backward}} \approx -0.22$ [S1, unofficial]
- $w \approx 0.75$ [S1, unofficial]

**Formulas**
The total phase function is a weighted sum of two Henyey-Greenstein functions:
$$ P(\theta) = w \cdot P_{\text{HG}}(\theta, g_{\text{forward}}) + (1 - w) \cdot P_{\text{HG}}(\theta, g_{\text{backward}}) $$

Where $P_{\text{HG}}$ is the standard Henyey-Greenstein phase function:
$$ P_{\text{HG}}(\theta, g) = \frac{1}{4\pi} \frac{1 - g^2}{(1 + g^2 - 2g \cos\theta)^{3/2}} $$

Here:
- $\theta$ is the scattering angle between the incident light direction and the view direction.
- $g$ is the asymmetry parameter ($-1 \le g \le 1$).
- $\cos\theta$ is the dot product of the normalized incident and view vectors.

**Steps**
1. Compute $\cos\theta$ as the dot product of the normalized incident light direction vector and the normalized view direction vector.
2. Calculate the forward lobe value $P_{\text{fwd}}$ using $g = 0.82$ and $\cos\theta$ in the Henyey-Greenstein formula.
3. Calculate the backward lobe value $P_{\text{bwd}}$ using $g = -0.22$ and $\cos\theta$ in the Henyey-Greenstein formula.
4. Combine the lobes: $P_{\text{total}} = 0.75 \cdot P_{\text{fwd}} + 0.25 \cdot P_{\text{bwd}}$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def henyey_greenstein(cos_theta: float, g: float) -> float:
    """
    Computes the single-lobe Henyey-Greenstein phase function.
    
    Args:
        cos_theta: Cosine of the scattering angle.
        g: Asymmetry parameter.
        
    Returns:
        Phase function value.
    """
    g_squared = g * g
    denominator = 1.0 + g_squared - 2.0 * g * cos_theta
    # Clamp denominator to avoid division by zero or negative roots if precision issues occur
    # though for valid g and cos_theta, denominator >= (1-|g|)^2 > 0 for |g|<1
    return (1.0 / (4.0 * math.pi)) * ((1.0 - g_squared) / (denominator ** 1.5))

def dual_lobe_henyey_greenstein(cos_theta: float) -> float:
    """
    Computes the dual-lobe Henyey-Greenstein phase function for cloud scattering.
    
    Args:
        cos_theta: Cosine of the scattering angle.
        
    Returns:
        Combined phase function value.
    """
    # Parameters from [S1, unofficial]
    g_forward = 0.82
    g_backward = -0.22
    w = 0.75
    
    p_fwd = henyey_greenstein(cos_theta, g_forward)
    p_bwd = henyey_greenstein(cos_theta, g_backward)
    
    return w * p_fwd + (1.0 - w) * p_bwd
```

**Edge Cases and Pitfalls**
- **Precision**: The denominator $(1 + g^2 - 2g \cos\theta)$ can become very small when $g$ is close to 1 and $\cos\theta$ is close to 1 (forward scattering), leading to large values. Standard floating-point precision is usually sufficient, but care should be taken if $g=1$ (singular). The sources use $g=0.82$, which is safe.
- **Boundary Conditions**: The formula assumes $\cos\theta \in [-1, 1]$. If the input vectors are not normalized, $\cos\theta$ may fall outside this range, leading to invalid results.
- **Normalization**: The Henyey-Greenstein function is normalized such that $\int P(\theta) d\Omega = 1$. The weighted sum of two normalized functions with weights summing to 1 ($0.75 + 0.25 = 1$) remains normalized.

## Subtopics

### Single-Lobe Henyey-Greenstein
The base component. Simpler, but lacks significant back-scatter. Choose when computational budget is extremely tight or back-scatter is negligible. [Link](4938-henyey-greenstein-phase-function.md)

### Rayleigh Phase Function
Models scattering by particles much smaller than the wavelength (e.g., air molecules). Highly symmetric with strong back-scatter. Choose for clear sky atmospheres, not clouds. [Link](4937-rayleigh-phase-function.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per scattering event. Requires a few multiplications, additions, and a power function.
- **Memory Complexity**: $O(1)$. No additional memory beyond scalar variables.
- **Measured Performance**: In WebGPU implementations for volumetric clouds, this function is computed per raymarch step. It is significantly faster than tabulated or spherical harmonic approximations while providing better visual fidelity than a single lobe for cloud rendering [S1, unofficial].
- **Strengths**: Physically plausible for cloud droplets (Mie scattering approximation); computationally cheap; easy to implement.
- **Limitations**: It is an approximation. Real Mie phase functions can have more complex structures (e.g., glory, rainbow) that this simple two-lobe model cannot capture.
- **When to use**: Real-time volumetric cloud rendering where performance is critical but single-lobe HG looks too "forward-only".
- **When not to use**: When high-fidelity atmospheric phenomena like rainbows or glories are required, or for non-cloud media with different scattering properties.

## Background
The Dual-Lobe Henyey-Greenstein phase function is an adaptation of the classic Henyey-Greenstein phase function, which was introduced by H.C. Henyey and P. Greenstein in 1941 to describe the scattering of light by turbid media [S1, unofficial]. The specific dual-lobe configuration with parameters $g_{\text{forward}} \approx 0.82$ and $g_{\text{backward}} \approx -0.22$ is derived from production techniques for real-time volumetric clouds, notably popularized by Andrew Schneider and Nathan Vos in their SIGGRAPH 2015 paper "The Real-Time Volumetric Cloudscapes" [S1, unofficial]. This approach allows for the modeling of softer backward scattering peaks, which are characteristic of cloud droplets, by blending a strongly forward-scattering lobe with a weaker backward-scattering lobe.

## Sources
[S1] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
