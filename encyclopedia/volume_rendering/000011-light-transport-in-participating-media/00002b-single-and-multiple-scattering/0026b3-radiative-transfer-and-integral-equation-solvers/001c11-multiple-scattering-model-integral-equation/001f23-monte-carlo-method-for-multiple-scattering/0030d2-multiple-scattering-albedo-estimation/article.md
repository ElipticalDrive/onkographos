# Multiple Scattering Albedo Estimation

Also called: Computation of multiple scattering albedo, $\sigma_{ms}$ estimation

## Overview
This method estimates the multiple scattering albedo $\alpha_{ms}$ (RGB) for a low-resolution voxel such that its effective albedo $\alpha_e$ matches the effective albedo $\alpha_e^{\text{input}}$ of the corresponding high-resolution input block [S1]. It preserves the color of light scattered at least twice at the microscopic scale due to self-shadowing during volume downsampling [S1]. It differs from single-scattering approximations by explicitly modeling the microscopic self-shadowing probability and solving for $\alpha_{ms}$ via a numerical root-finding process based on scattering event probabilities [S1].

## How it works
**Inputs:**
- High-resolution input voxel block with known extinction coefficient $\sigma_t$, single-scattering albedo $\alpha_{ss}$, and phase function [S1].
- Low-resolution target voxel geometry [S1].
- Maximum number of scattering events $N_{\text{max}}$ for probability estimation [S1].

**Outputs:**
- Multiple scattering albedo $\alpha_{ms}$ for the low-resolution voxel (RGB) [S1].

**Data Structures:**
- $P_{\text{sh}}(n)$: Array of size $N_{\text{max}}+1$, storing the probability of exactly $n$ scattering events [S1].
- $P_{\text{ss}}$: Scalar, microscopic self-shadowing probability [S1].
- $\alpha_e^{\text{input}}$: Vector (RGB), effective albedo of the input block [S1].

**Steps:**
1. **Estimate $\alpha_e^{\text{input}}$:** Cast rays through the input block of high-resolution voxels. For each ray, compute the light path until it leaves the block, exactly as a volume path tracer would do. Average the ray throughputs to obtain $\alpha_e^{\text{input}}$ [S1].
2. **Estimate $P_{\text{sh}}(n)$:** Cast rays in the low-resolution voxel and estimate probabilities for $n$ scattering events up to $N_{\text{max}}$. These probabilities do not depend on $\alpha_{ms}$ [S1].
3. **Estimate $P_{\text{ss}}$:** Compute the microscopic self-shadowing probability using the approximation $P_{\text{ss}} \approx \frac{\int \sigma_{\text{SS}}}{\alpha_{\text{ss}}(\lambda) \int \sigma_t}$, where $\int \sigma_{\text{SS}}$ is the integral of the self-shadowing cross-section, $\alpha_{\text{ss}}(\lambda)$ is the single-scattering albedo at wavelength $\lambda$, and $\int \sigma_t$ is the integral of the extinction coefficient over the voxel [S1].
4. **Solve for $\alpha_{ms}$:** Numerically solve $\alpha_e^{\text{input}} = \sum_{n=0}^{N_{\text{max}}} P_{\text{sh}}(n) (P_{\text{ss}} \alpha_{\text{ss}} + (1 - P_{\text{ss}}) \alpha_{\text{ms}})^n$ for each wavelength [S1].

**Parameters:**
- Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
- The method tends to slightly underestimate the effective albedo $\alpha_e$ [S1].
- Accuracy depends on the density and the amount of shadowing in the voxel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def estimate_multiple_scattering_albedo(input_block, low_res_voxel, N_max):
    # Step 1: Estimate effective albedo of input block [G]
    alpha_e_input = estimate_effective_albedo_path_tracing(input_block)

    # Step 2: Estimate scattering event probabilities for low-res voxel [G]
    P_sh = estimate_scattering_probabilities(low_res_voxel, N_max)

    # Step 3: Estimate self-shadowing probability [G]
    integral_sigma_ss = sum_coefficients(input_block, 'sigma_ss')
    integral_sigma_t = sum_coefficients(input_block, 'sigma_t')
    alpha_ss = get_average_single_scattering_albedo(input_block)

    P_ss = integral_sigma_ss / (alpha_ss * integral_sigma_t)

    # Step 4: Solve for alpha_ms for each color channel
    alpha_ms = [0.0, 0.0, 0.0]

    for c in range(3):
        target = alpha_e_input[c]
        alpha_ss_c = alpha_ss[c] if isinstance(alpha_ss, list) else alpha_ss

        def f(x):
            val = 0.0
            for n in range(N_max + 1):
                term = (P_ss * alpha_ss_c + (1 - P_ss) * x) ** n
                val += P_sh[n] * term
            return val - target

        alpha_ms[c] = bisection_root(f, lower=0.0, upper=1.0)

    return alpha_ms

def estimate_effective_albedo_path_tracing(block):
    # Cast rays through block, trace paths until exit, average throughput [G]
    throughputs = []
    for ray in cast_rays(block):
        throughput = 1.0
        pos = ray.origin
        while in_voxel(pos, block):
            dist = compute_distance_to_boundary(pos, ray.direction, block)
            tau = block.extinction_coefficient * dist
            if random.uniform(0, 1) > math.exp(-tau):
                throughput *= 0.0
                break
            pos = [pos[i] + ray.direction[i] * dist for i in range(3)]
            if random.uniform(0, 1) < block.single_scattering_albedo:
                throughput *= block.single_scattering_albedo
            else:
                break
        throughputs.append(throughput)
    return sum(throughputs) / len(throughputs)

def estimate_scattering_probabilities(voxel, N_max):
    # Cast rays, count scattering events, normalize to probabilities [G]
    counts = [0] * (N_max + 1)
    total_rays = 0
    for ray in cast_rays(voxel):
        n = 0
        pos = ray.origin
        while in_voxel(pos, voxel):
            dist = compute_distance_to_boundary(pos, ray.direction, voxel)
            tau = voxel.extinction_coefficient * dist
            if random.uniform(0, 1) > math.exp(-tau):
                break
            pos = [pos[i] + ray.direction[i] * dist for i in range(3)]
            n += 1
            if n > N_max:
                break
        counts[n] += 1
        total_rays += 1
    return [c / total_rays for c in counts]

def sum_coefficients(block, coeff_name):
    # Sum coefficients over all high-res voxels in block [G]
    total = 0.0
    for voxel in block.voxels:
        total += getattr(voxel, coeff_name) * voxel.volume
    return total

def get_average_single_scattering_albedo(block):
    # Return average single-scattering albedo over block [G]
    return block.average_single_scattering_albedo

def bisection_root(f, lower, upper, tol=1e-6, max_iter=100):
    # Standard bisection method [G]
    a, b = lower, upper
    for _ in range(max_iter):
        mid = (a + b) / 2.0
        if f(mid) == 0.0 or (b - a) / 2.0 < tol:
            return mid
        if f(a) * f(mid) < 0:
            b = mid
        else:
            a = mid
    return (a + b) / 2.0

# Standard graphics/math helpers [G]
def cast_rays(voxel):
    # Generate a set of uniformly distributed rays through the voxel
    pass

def in_voxel(pos, voxel):
    # Check if position is inside voxel bounds
    pass

def compute_distance_to_boundary(pos, direction, voxel):
    # Compute distance to voxel boundary along direction
    pass
```

## Subtopics

## Operating characteristics
- **Time Complexity:** Pre-computation time depends on the number of samples used for estimating self-shadowing probabilities and multiple scattering albedos, which are adapted to the complexity of the block of input voxels [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Preserves the color of multiple scattering during downsampling by matching effective albedos [S1].
- **Limitations:** Approximation errors in $P_{\text{ss}}$ and the effective albedo formula can lead to inaccuracies, particularly underestimating $\alpha_e$ [S1].

## Background
This method is part of the "new microflake model with microscopic self-shadowing" for accurate volume downsampling [S1]. It was introduced to handle the color of light scattered at least two times at the microscopic scale due to self-shadowing [S1]. The approach is detailed in the paper "A new microflake model with microscopic self-shadowing for accurate volume downsampling" [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
