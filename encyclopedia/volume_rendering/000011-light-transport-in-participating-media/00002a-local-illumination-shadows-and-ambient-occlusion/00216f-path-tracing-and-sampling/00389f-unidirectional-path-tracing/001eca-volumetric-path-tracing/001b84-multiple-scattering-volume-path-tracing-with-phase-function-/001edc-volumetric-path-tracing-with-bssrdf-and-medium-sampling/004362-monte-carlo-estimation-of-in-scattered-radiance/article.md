# Monte Carlo estimation of in-scattered radiance

Also called: Monte Carlo In-scattering, Source Function Estimation

## Overview
This method estimates the in-scattered radiance $L_{\text{ins}}(x, \boldsymbol{\omega})$ for interior points $x \in \Omega \setminus \partial \Omega$ and boundary points $x \in \partial \Omega$ [S1, unofficial]. It computes the angular integral of the source function by sampling directions according to the phase function [S2, unofficial].

## How it works
**Inputs:**
- Point $x$ (position in 3D space) [S1, unofficial].
- Direction $\boldsymbol{\omega}$ (outgoing/view direction) [S1, unofficial].
- Phase function $f_p(x, \boldsymbol{\omega}', \boldsymbol{\omega})$ or $p(x, \boldsymbol{\omega}', \boldsymbol{\omega})$ [S1, unofficial][S2, unofficial].
- Surface normal $\mathbf{n}(x)$ (if $x$ is on the boundary) [S1, unofficial].
- Radiance evaluation function $L(x, \boldsymbol{\omega}')$ (returns interior radiance) [S1, unofficial].
- Boundary radiance function $L_s(x, \boldsymbol{\omega}')$ (returns refracted surface radiance) [S1, unofficial].
- Epsilon $\epsilon$ (small scalar, $\epsilon \to 0^+$) [S1, unofficial].
- Boolean `is_boundary` (indicates if $x \in \partial \Omega$) [G].

**Outputs:**
- Estimated in-scattered radiance $L_{\text{ins}}$ [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Steps:**
1. Determine if $x$ is in the interior ($x \in \Omega \setminus \partial \Omega$) or on the boundary ($x \in \partial \Omega$) [S1, unofficial].
2. If $x$ is in the interior: draw a direction $\boldsymbol{\omega}'$ from the phase function distribution $f_p(x, -\boldsymbol{\omega}', \boldsymbol{\omega})$ and set $L_{\text{ins}} \leftarrow L(x, \boldsymbol{\omega}')$ [S1, unofficial].
3. If $x$ is on the boundary: define a point slightly inside the medium $x_{\text{in}} = x + \epsilon \boldsymbol{\omega}$, draw $\boldsymbol{\omega}' \sim f_p(x_{\text{in}}, -\boldsymbol{\omega}', \boldsymbol{\omega})$, and check the dot product $\langle \mathbf{n}(x), \boldsymbol{\omega}' \rangle > 0$ [S1, unofficial].
4. If $\langle \mathbf{n}(x), \boldsymbol{\omega}' \rangle > 0$, set $L_{\text{ins}} \leftarrow L(x, \boldsymbol{\omega}')$; otherwise set $L_{\text{ins}} \leftarrow L_s(x, \boldsymbol{\omega}')$ [S1, unofficial].
5. Return $L_{\text{ins}}$ [S1, unofficial].

**Formulas:**
The source function $J(s')$ represents in-scattering from all directions at $s'$ [S2, unofficial]:
$$ J(s') = \int_{4\pi} p(s', \boldsymbol{\omega}', \boldsymbol{\omega}) L(s', \boldsymbol{\omega}') \, d\boldsymbol{\omega}' $$
The Monte Carlo estimator for the source function with $M$ samples is [S2, unofficial]:
$$ J_N = \frac{1}{M} \sum_{k=1}^M \frac{p(s', \boldsymbol{\omega}_k', \boldsymbol{\omega}) L(s', \boldsymbol{\omega}_k')}{\text{pdf}(\boldsymbol{\omega}_k')} $$
Importance sampling uses a pdf similar to the phase function to reduce variance [S2, unofficial]. For isotropic media, $p = \frac{1}{4\pi}$ and directions are sampled uniformly on the unit sphere [S2, unofficial].

**Parameters:**
- $M$: Number of samples for the source function estimator [S2, unofficial]. Value not given in the sources.
- $\epsilon$: Small positive scalar to offset boundary points into the interior, $\epsilon \to 0^+$ [S1, unofficial]. Value not given in the sources.

**Pitfalls:**
- **Boundary Precision:** When $x$ is on the boundary, sampling directly at $x$ can cause incorrect normal evaluation or phase function sampling outside the medium. Offsetting by $\epsilon$ along $\boldsymbol{\omega}$ is required [S1, unofficial].
- **Direction Validity:** On the boundary, if the sampled direction $\boldsymbol{\omega}'$ points out of the medium ($\langle \mathbf{n}(x), \boldsymbol{\omega}' \rangle \le 0$), the radiance must come from the surface source $L_s(x, \boldsymbol{\omega}')$ rather than the interior volume $L(x, \boldsymbol{\omega}')$ [S1, unofficial].
- **Distance to Next Scatter:** When estimating diffusion terms involving distance sampling, if the sampled distance to the next scattering event $d_j$ is greater than the distance to the surface $s$, the event corresponds to refraction (direct transmission), not an internal scatter [S2, unofficial]. This method handles the angular part; the distance part is handled by the parent method's distance sampling.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_in_scattered_radiance(x, omega, phase_function, normal, L_eval, L_s_eval, epsilon, is_boundary):
    """
    Estimates in-scattered radiance L_ins(x, omega) for interior or boundary points.
    """
    if not is_boundary:
        # Interior point: x in Omega \ partial Omega
        omega_prime = sample_phase_function(phase_function, x, -omega)
        L_ins = L_eval(x, omega_prime)
    else:
        # Boundary point: x in partial Omega
        x_in = x + epsilon * omega
        omega_prime = sample_phase_function(phase_function, x_in, -omega)
        if dot(normal, omega_prime) > 0:
            L_ins = L_eval(x, omega_prime)
        else:
            L_ins = L_s_eval(x, omega_prime)
    return L_ins

def sample_phase_function(phase_function, x, incoming_direction):
    """
    Samples a direction omega_prime from the phase function f_p(x, -omega_prime, incoming_direction).
    Importance sampling uses a pdf similar to the phase function [S2, unofficial].
    For isotropic media (p = 1/4pi), samples uniformly on the unit sphere [S2, unofficial].
    For anisotropic media, applies the specific inverse CDF or rejection sampling for the given phase function [S2, unofficial].
    """
    # Standard uniform sampling on unit sphere [G]
    phi = 2.0 * pi * random()
    cos_theta = 1.0 - 2.0 * random()
    sin_theta = sqrt(1.0 - cos_theta * cos_theta)
    return vec3(sin_theta * cos(phi), sin_theta * sin(phi), cos_theta)
```

## Subtopics

### [BSDF Sampling for Volumetric Paths](1995-bsdf-sampling-for-volumetric-paths.md)
Samples the BSDF at the surface to determine the next direction, often used in conjunction with volume sampling for hybrid paths.

### [Reference Path Tracing](3170-reference-path-tracing.md)
A baseline implementation for validating the in-scattering estimator against a brute-force or analytical solution.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** $O(1)$ auxiliary memory per evaluation [G].
- **Strengths:** Unbiased estimator for the in-scattering term. Importance sampling using a pdf similar to the phase function reduces variance [S2, unofficial].
- **Limitations:** High variance if the phase function is highly anisotropic and not sampled correctly. Requires accurate evaluation of $L(x, \boldsymbol{\omega}')$ or $L_s(x, \boldsymbol{\omega}')$. Boundary handling requires offsetting by $\epsilon$ along $\boldsymbol{\omega}$ [S1, unofficial].
- **When to use:** When accurate volumetric lighting is required and the in-scattering term is significant.
- **When not to use:** When performance is critical and approximate lighting is sufficient.

## Background
The method estimates the source function $J(s')$ in the Radiative Transfer Equation (RTE) [S2, unofficial]. The first definitive text on radiative transfer was published by Chandrasekhar [1950] [S2, unofficial]. The specific algorithm for estimating $L_{\text{ins}}$ at interior and boundary points, including the handling of discontinuities and side paths, is described in "A Differential Theory of Radiative Transfer" [S1, unofficial]. Phase function sampling techniques, such as for the Henyey-Greenstein phase function, are based on work by Henyey and Greenstein [1940] [S2, unofficial].

## Sources
[S1] A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
[S2] Volume Rendering and Scattering Theory, https://www.scribd.com/document/912805133/02941-Slides-07-Volumes, fetched 2026-10-29 (unofficial)
