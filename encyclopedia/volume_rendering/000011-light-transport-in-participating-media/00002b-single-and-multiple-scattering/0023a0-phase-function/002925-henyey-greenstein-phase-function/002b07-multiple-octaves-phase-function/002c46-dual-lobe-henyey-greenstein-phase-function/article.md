# Dual-Lobe Henyey-Greenstein Phase Function
Also called: Double Henyey-Greenstein Phase Function, Dual Phase Function, Dual-Lobe Henyey-Greenstein Scattering, Dual-lobe HG, Dual-lobe angular phase function, MHG phase function, Multi-Henyey-Greenstein Phase Function

## Overview
The Dual-Lobe Henyey-Greenstein Phase Function models light scattering in participating media by combining a forward-scattering lobe and a backward-scattering lobe to create a more realistic angular distribution than a single lobe [S2, unofficial]. It sets itself apart from the parent [Multiple Octaves Phase Function](4939-multiple-octaves-phase-function.md) by specifically using a two-lobed structure rather than a general composite, providing a computationally efficient approximation of Mie scattering with significant back-scatter [S2, unofficial]. It is not scientifically correct but is used for visual aesthetics [S3, unofficial].

## How it works
**Inputs and Outputs**
- Input: Scattering angle $\theta$ (or its cosine, $\cos\theta$) [G].
- Output: Phase function value $P(\theta)$, a scalar probability density [G].

**Data Structures**
No complex data structures are required. The method relies on scalar parameters:
- $g_{\text{forward}}$: Forward asymmetry parameter [G].
- $g_{\text{backward}}$: Backward asymmetry parameter [G].
- $w$: Blend weight for the forward lobe [G].

**Parameters**
Typical values for cloud rendering, as observed in production implementations:
- $g_{\text{forward}} \approx 0.82$ [S2, unofficial]
- $g_{\text{backward}} \approx -0.22$ [S2, unofficial]
- $w \approx 0.75$ [S2, unofficial]

**Formulas**
The total phase function is a weighted sum of two Henyey-Greenstein functions:
$$ P(\theta) = w \cdot P_{\text{HG}}(\theta, g_{\text{forward}}) + (1 - w) \cdot P_{\text{HG}}(\theta, g_{\text{backward}}) $$
Where $P_{\text{HG}}$ is the standard Henyey-Greenstein phase function:
$$ P_{\text{HG}}(\theta, g) = \frac{1}{4\pi} \frac{1 - g^2}{(1 + g^2 - 2g \cos\theta)^{1.5}} $$
Here:
- $\theta$ is the scattering angle between the incident light direction and the view direction [G].
- $g$ is the asymmetry parameter ($-1 \le g \le 1$) [G].
- $\cos\theta$ is the dot product of the normalized incident and view vectors [G].

**Steps**
1. Compute $\cos\theta$ as the dot product of the normalized incident light direction vector and the normalized view direction vector [G].
2. Calculate the forward lobe value $P_{\text{fwd}}$ using $g = 0.82$ and $\cos\theta$ in the Henyey-Greenstein formula [S2, unofficial].
3. Calculate the backward lobe value $P_{\text{bwd}}$ using $g = -0.22$ and $\cos\theta$ in the Henyey-Greenstein formula [S2, unofficial].
4. Combine the lobes: $P_{\text{total}} = 0.75 \cdot P_{\text{fwd}} + 0.25 \cdot P_{\text{bwd}}$ [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def henyey_greenstein(cos_theta: float, g: float) -> float:
    g_squared = g * g
    denominator = 1.0 + g_squared - 2.0 * g * cos_theta
    # Guard against division by zero when g=1 and cos_theta=1 [G]
    denominator = max(denominator, 1e-8)
    return (1.0 / (4.0 * math.pi)) * ((1.0 - g_squared) / (denominator ** 1.5))

def dual_lobe_henyey_greenstein(cos_theta: float) -> float:
    g_forward = 0.82
    g_backward = -0.22
    w = 0.75
    p_fwd = henyey_greenstein(cos_theta, g_forward)
    p_bwd = henyey_greenstein(cos_theta, g_backward)
    return w * p_fwd + (1.0 - w) * p_bwd
```

**Edge Cases and Pitfalls**
- **Precision**: The denominator $(1 + g^2 - 2g \cos\theta)$ can become very small when $g$ is close to $1$ and $\cos\theta$ is close to $1$ (forward scattering), leading to large values. Standard floating-point precision is usually sufficient, but care should be taken if $g=1$ (singular) [G].
- **Boundary Conditions**: The formula assumes $\cos\theta \in [-1, 1]$. If the input vectors are not normalized, $\cos\theta$ may fall outside this range, leading to invalid results [G].
- **Normalization**: The Henyey-Greenstein function is normalized such that $\int P(\theta) d\Omega = 1$. The weighted sum of two normalized functions with weights summing to $1$ ($0.75 + 0.25 = 1$) remains normalized [G].
- **Computational Cost**: The method is computationally expensive, involving double the heavy calculations for sampling phase functions and PDFs compared to a single Henyey-Greenstein [S3, unofficial].

## Subtopics

### Single-Lobe Henyey-Greenstein
Not given in the sources. [Link](4938-henyey-greenstein-phase-function.md) [G]

### Rayleigh Phase Function
Not given in the sources. [Link](4937-rayleigh-phase-function.md) [G]

## Operating characteristics
- **Time Complexity**: $O(1)$ per scattering event. Requires a few multiplications, additions, and a power function [G].
- **Memory Complexity**: $O(1)$. No additional memory beyond scalar variables [G].
- **Measured Performance**: Not given in the sources. [G]
- **Strengths**: Models softer backward back-scattering peaks characteristic of cloud droplets [S2, unofficial].
- **Limitations**: It is not scientifically correct but used for visual aesthetics [S3, unofficial]. Real Mie phase functions can have more complex structures that this simple two-lobe model cannot capture [G].
- **When to use**: Real-time volumetric cloud rendering where performance is critical but single-lobe HG looks too "forward-only" [S2, unofficial].
- **When not to use**: Not given in the sources. [G]

## Background
The Dual-Lobe Henyey-Greenstein phase function is an adaptation of the classic Henyey-Greenstein phase function [S2, unofficial]. The specific dual-lobe configuration with parameters $g_{\text{forward}} \approx 0.82$ and $g_{\text{backward}} \approx -0.22$ is derived from production techniques for real-time volumetric clouds [S2, unofficial]. This approach allows for the modeling of softer backward scattering peaks, which are characteristic of cloud droplets, by blending a strongly forward-scattering lobe with a weaker backward-scattering lobe [S2, unofficial]. It is not scientifically correct but is used for visual aesthetics [S3, unofficial].

## Sources
[S1] A real-time rendering method for high albedo anisotropic materials with multiple scattering, https://doi.org/10.48550/arxiv.2401.14051, fetched 2026-10-26 (unofficial)
[S2] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S3] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
