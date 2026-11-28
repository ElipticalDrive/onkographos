# Discrete Volume Transmittance Integration > Discrete Radiance Step Integration

Also called: CPU Volume Transport (Single Ray), Discrete RTE Step, Discrete Radiance Step Integration, Small Step Radiance Update, Transport Step, Transport Step Function

## Overview
Discrete Radiance Step Integration computes the outgoing radiance after traversing a small step of length $ds$ through a participating medium with extinction coefficient $\sigma$ and emission coefficient $q$, given incoming radiance $I$ [S1, unofficial]. It serves as the core update kernel for the discrete radiative transfer equation, iteratively applying the transport equation along a single ray to compute final intensity from initial intensity, absorption coefficient, and source term [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $I$: Incoming radiance (scalar or vector).
    *   $\sigma$: Extinction coefficient (scalar).
    *   $q$: Source term (emission coefficient, scalar or vector).
    *   $ds$: Step size along the ray (scalar).
*   **Output:**
    *   $I_{out}$: Outgoing radiance after the step.

**Data Structures:**
No complex data structures are required; the method operates on scalar or vector values representing radiance, density, and source terms at a single sample point [S1, unofficial].

**Numbered Steps:**
1.  Calculate the optical depth $\tau$ for the step:
    $$ \tau = \sigma \cdot ds $$
2.  Check for small optical depth. If $|\tau| < 10^{-5}$, use the linear approximation:
    $$ I_{out} = I + q \cdot ds - I \cdot \tau $$
3.  If $|\tau| \ge 10^{-5}$, calculate the transmittance loss factor $T_{loss}$:
    $$ T_{loss} = 1 - e^{-\tau} $$
    Calculate the outgoing radiance:
    $$ I_{out} = I \cdot (1 - T_{loss}) + \frac{q}{\sigma} \cdot T_{loss} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transport_step(I, sigma, source, ds):
    tau = sigma * ds
    if abs(tau) < 1e-5:
        return I + source * ds - I * tau
    else:
        # T_loss = 1 - exp(-tau)
        # Using expm1 for precision: expm1(x) = exp(x) - 1
        # So 1 - exp(-tau) = -expm1(-tau)
        one_minus_T = -(math.exp(-tau) - 1)
        if sigma == 0:
            return I
        return I * (1 - one_minus_T) + (source / sigma) * one_minus_T

def discrete_radiance_integration(I0, sigma_samples, source_samples, ds, num_samples):
    I = I0
    for i in range(num_samples):
        I = transport_step(I, sigma_samples[i], source_samples[i], ds)
    return I
```

**Parameters:**
*   $ds$: Step size along the ray. Typical value in benchmarks is $0.01$ [S1, unofficial].
*   Threshold for small optical depth: $10^{-5}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** When $\tau$ is very small, direct exponentiation may lose precision or be inefficient. The linear approximation branch for $|\tau| < 10^{-5}$ handles this [S1, unofficial].
*   **Division by Zero:** Division by $\sigma$ occurs in the non-small $\tau$ branch. If $\sigma$ is zero, this results in infinity or NaN unless handled. However, if $\sigma=0$, then $\tau=0$ (assuming $ds \neq 0$), which falls into the small $\tau$ branch, avoiding the division [S1, unofficial]. If $ds=0$ and $\sigma=0$, the small $\tau$ branch returns $I$, which is correct [S1, unofficial].

## Subtopics

### Ray Integration
Computes the color and brightness (energy) accumulated as light passes through the interior of an object, accounting for the medium’s absorption, scattering, and emission effects [S2, unofficial]. It is defined as $C(t) = \int_{t_{near}}^{t_{far}} T(t) \cdot \sigma(t) \cdot c(t) \, dt$, where $C(t)$ is the final color or brightness, $\sigma(t)$ is the sum of absorption and scattering, $c(t)$ is the medium’s intrinsic color and intensity information, and $t_{near}$ and $t_{far}$ are the start and end points where the ray intersects the medium [S2, unofficial]. The transmittance $T(t)$ describes how much the intensity of the light is attenuated as it passes through the object, defined as $T(t) = \exp\left(-\int_{t_{\text{near}}}^{t} \sigma(s) \, ds\right)$, which is a generalized form of the Beer-Lambert Law [S2, unofficial]. First introduced by James T. Kajiya in his 1984 Siggraph paper, Ray Tracing Volume Densities [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step. The integration over $N$ samples is $O(N)$ [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per step [S1, unofficial].
*   **Strengths:** Simple to implement, numerically stable for small optical depths using the linear approximation, efficient for single-ray transport [S1, unofficial].
*   **Limitations:** Requires careful handling of small optical depths. Not directly applicable to heterogeneous media without sampling the coefficients at each step [S1, unofficial].
*   **When to use:** When performing discrete integration of the radiative transfer equation along a ray, particularly when high precision is needed for small steps or when the medium properties are known at discrete sample points [S1, unofficial].
*   **When not to use:** For large-scale volume rendering where transmittance is the primary concern and radiance update is separate, or when analytic solutions are available and preferred [S1, unofficial].

## Background
This method is derived from the discrete solution of the Radiative Transfer Equation (RTE) for participating media. It is commonly used in CPU-based volume rendering implementations and as a foundational step in more complex transport algorithms. The specific implementation details, including the use of `expm1f` for precision and the threshold for linear approximation, are observed in open-source HPC demonstrations [S1, unofficial].

## Sources
[S1] wyh1309/hpc_foundational_demos, https://github.com/wyh1309/hpc_foundational_demos, fetched 2026-10-26 (unofficial)
[S2] GLSL Volumetric Rendering (The Rigorous, Harder Version) · Okdalto, https://okdalto.github.io/en/glsl-volumetric-rendering-rigorous/, fetched 2026-10-30 (unofficial)
