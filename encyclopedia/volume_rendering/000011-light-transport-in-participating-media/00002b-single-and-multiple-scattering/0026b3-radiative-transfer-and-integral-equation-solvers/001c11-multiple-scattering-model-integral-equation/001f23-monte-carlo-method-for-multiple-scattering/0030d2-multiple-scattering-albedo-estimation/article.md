# Multiple Scattering Albedo Estimation

Also called: Computation of multiple scattering albedo, $\sigma_{ms}$ estimation

## Overview
This method estimates the multiple scattering albedo $\alpha_{ms}$ (RGB) for a low-resolution voxel such that its effective albedo $\alpha_e$ matches the effective albedo $\alpha_e^{\text{input}}$ of the corresponding high-resolution input block [S1]. It solves the problem of preserving the color and density of light scattered at least twice at the microscopic scale during volume downsampling [S1]. Unlike [Light Tracing for Participating Media](12255-light-tracing-for-participating-media.md), which simulates light transport from the light source, this method operates as a pre-computation step to derive parameters for a standard Monte Carlo renderer, ensuring the low-res voxel reproduces the multi-scattering behavior of the original high-res data without iterative optimization [S1].

## How it works
**Inputs:**
- A block of high-resolution input voxels with known scattering properties [S1].
- A target low-resolution voxel geometry corresponding to the input block [S1].
- The single-scattering albedo $\alpha_{ss}$ and self-shadowing probability $P_{ss}$ for the low-res voxel [S1].

**Outputs:**
- The multiple scattering albedo $\alpha_{ms}$ (RGB) for the low-res voxel [S1].

**Data Structures:**
- `P_sh`: Array of probabilities for a ray undergoing exactly $n$ scattering events before leaving the voxel. Size: $N_{max} + 1$, where $N_{max}$ is the maximum number of scattering events considered [S1].
- `alpha_e_input`: Scalar (RGB) representing the average throughput of rays leaving the input block [S1].

**Steps:**

1. **Estimate $\alpha_e^{\text{input}}$:** Cast rays through the block of input voxels [S1]. For each ray, compute the light path until it leaves the block, treating the block as a participating medium [S1]. The throughput is the product of transmittance and scattering contributions along the path [S1]. Average the throughputs of all cast rays to obtain $\alpha_e^{\text{input}}$ [S1].

2. **Estimate $P_{sh}(n)$:** Cast rays through the low-resolution voxel [S1]. For each ray, count the number of scattering events $n$ before the ray leaves the voxel [S1]. Compute the histogram of these counts and normalize to obtain the probability $P_{sh}(n)$ for each $n \in [0, N_{max}]$ [S1]. These probabilities do not depend on $\alpha_{ms}$ [S1].

3. **Approximate $P_{ss}$:** Calculate the microscopic self-shadowing probability $P_{ss}$ using the approximation:
   $$P_{SS} \approx \frac{\int \sigma_{SS}}{\alpha_{SS}(\lambda) \int \sigma_t}$$
   where $\sigma_{SS}$ is the self-shadowing cross-section, $\alpha_{SS}(\lambda)$ is the single-scattering albedo, and $\sigma_t$ is the extinction coefficient [S1].

4. **Solve for $\alpha_{ms}$:** For each wavelength channel, solve the following equation numerically for $\alpha_{ms}$:
   $$\alpha_e^{\text{input}} = \sum_{n=0}^{N_{max}} P_{sh}(n) \left( P_{ss} \alpha_{ss} + (1 - P_{ss}) \alpha_{ms} \right)^n$$
   The term $\left( P_{ss} \alpha_{ss} + (1 - P_{ss}) \alpha_{ms} \right)$ represents the effective albedo per scattering event at the microscopic scale [S1]. The summation accounts for the probability of having $n$ such events [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_multiple_scattering_albedo(input_block, low_res_voxel, alpha_ss, P_ss, N_max, num_rays):
    # Step 1: Estimate alpha_e_input
    # [G] Standard volume path tracing: march rays through input_block, accumulate throughput
    # until leaving, then average. The source does not specify the exact march routine.
    alpha_e_input = 0.0
    for _ in range(num_rays):
        throughput = 1.0
        # [G] Ray marching loop: sample interactions, apply transmittance and scattering
        # contributions until ray exits input_block
        alpha_e_input += throughput
    alpha_e_input /= num_rays

    # Step 2: Estimate P_sh(n)
    P_sh = [0.0] * (N_max + 1)
    for _ in range(num_rays):
        n = 0
        # [G] Ray marching loop: count scattering events n until ray exits low_res_voxel
        P_sh[n] += 1.0
    for n in range(N_max + 1):
        P_sh[n] /= num_rays

    # Step 3: Solve for alpha_ms per wavelength
    alpha_ms = [0.0, 0.0, 0.0] # RGB
    for channel in range(3):
        # Bisection root finding for alpha_ms in [0, 1]
        low, high = 0.0, 1.0
        for _ in range(50):
            mid = (low + high) / 2.0
            term = P_ss * alpha_ss[channel] + (1.0 - P_ss) * mid
            effective_albedo = sum(P_sh[n] * (term ** n) for n in range(N_max + 1))
            if effective_albedo > alpha_e_input[channel]:
                high = mid
            else:
                low = mid
        alpha_ms[channel] = (low + high) / 2.0
    return alpha_ms
```

**Parameters:**
- $N_{max}$: Maximum number of scattering events. The maximum number of scattering events is chosen up to a given limit [S1].
- Ray count: The number of samples is adapted to the complexity of the input block [S1].

**Edge Cases and Pitfalls:**
- The method tends to slightly underestimate the effective albedo $\alpha_e$ [S1]. Accuracy depends on the density and amount of shadowing in the voxel [S1].
- If $P_{ss}$ is inaccurate, the estimated $\alpha_{ms}$ will be biased [G].
- Numerical stability of the root-finding algorithm should be ensured, especially when $\alpha_{ms}$ is close to 0 or 1 [G].

## Subtopics

### [Optical Thickness Reduction via Similarity Relations](11202-optical-thickness-reduction-via-similarity-relations.md)
A preprocessing method that accelerates Monte Carlo rendering of optically dense, forward-scattering participating media by replacing original scattering parameters with altered parameters that yield fewer scattering events, without modifying the core rendering algorithm. Unlike methods that require renderer modifications, this technique outputs altered parameters ($\sigma^*_a, \sigma^*_s, f^*(\cdot)$) compatible with any standard Monte Carlo renderer (e.g., Mitsuba). It sets apart from simpler first-order approximations by utilizing higher-order similarity relations (up to order $N=5$) to better capture complex phase function moments, while explicitly guarding against overfitting via support regularization.

## Operating characteristics
- **Time Complexity:** Depends on the number of rays cast for estimating $\alpha_e^{\text{input}}$ and $P_{sh}(n)$ [S1]. Pre-computation time scales with the complexity of the input block and the number of samples [S1].
- **Memory Complexity:** Low, as it only stores probabilities $P_{sh}(n)$ and intermediate albedo values [G].
- **Strengths:** Avoids iterative optimization, making it faster than methods that require solving for $\alpha_{ms}$ via repeated rendering [S1]. Compatible with standard Monte Carlo renderers [S1].
- **Limitations:** Slight underestimation of effective albedo [S1]. Accuracy degrades in highly shadowed or dense voxels [S1].
- **When to use:** When downsampling participating media and preserving multi-scattering color fidelity is critical, and iterative methods are too slow [S1].

## Background
This method is part of a microflake model with microscopic self-shadowing for accurate volume downsampling [S1]. It was introduced to address the loss of multi-scattering information when reducing the resolution of participating media [S1]. The approach approximates the effective albedo using a series expansion based on scattering event probabilities, allowing for efficient pre-computation of scattering parameters [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
