# Differential Ratio Tracing

Also called: Differential Ratio Tracking, Mitsuba 3's Differential Ratio Tracing, volpathsimple-drt

## Overview
Differential Ratio Tracing (DRT) is a sampling strategy for estimating gradients of the radiative transfer equation with respect to medium density in differentiable volume rendering [S1]. It solves the problem of biased and high-variance gradients produced by standard volumetric free-flight sampling by sampling distances proportional to the unweighted transmittance rather than extinction-weighted transmittance [S1]. Unlike [Single-Scattering Differentiable Volume Rendering](4143-single-scattering-differentiable-volume-rendering.md), which relies on pre-computed view-independent terms, DRT is a Monte Carlo estimator that must recompute gradients for each view [S2]. It is designed for physically-based scattering and absorbing volumes, as opposed to NeRF-style emissive volumes [S1].

## How it works
DRT is an online algorithm that samples a distance $t$ proportional to transmittance $T(t)$ and returns a weight $w$ estimating $C = \int_0^{t_s} T(s) ds$, by combining ratio tracking with weighted reservoir sampling to avoid storing all segments [S1].

### Inputs and Outputs
- **Inputs**:
  - $t_{max}$: The maximum distance to sample along the ray (typically the exit distance of the volume or the ray length) [S1].
  - $\bar{\sigma}$: The majorant extinction coefficient, assumed constant for the homogenized medium sampling [S1].
  - $\sigma_t(t)$: The function returning the actual extinction coefficient at distance $t$ [S1].
  - `rng()`: A function returning a uniform random number $u \sim U[0,1)$ [S1].
- **Outputs**:
  - $t$: The sampled distance, where $0 \le t \le t_{max}$ [S1].
  - $w_{acc}$: The total accumulated weight, an unbiased estimate of the integral of transmittance [S1].

### Data Structures
- **Reservoir**: A tuple $(t_r, d_r)$ storing the start distance $t_r$ and length $d_r$ of the currently selected segment [S1]. Initialized as `None`.
- **Accumulators**:
  - $T_r$: Transmittance estimate, initialized to $1.0$ [S1].
  - $t_{run}$: Current run distance, initialized to $0.0$ [S1].
  - $w_{acc}$: Accumulated weight, initialized to $0.0$ [S1].

### Algorithm Steps
The method proceeds in a loop until the ray is fully traversed or the maximum distance is reached:

1. **Sample Step**: Sample a step size $dt$ from the homogenized medium with majorant $\bar{\sigma}$:
   $$ dt = -\ln(1 - u) / \bar{\sigma} $$
   where $u \sim U[0,1)$ [S1].
2. **Clamp Step**: Clamp $dt$ to the remaining distance:
   $$ dt = \min(dt, t_{max} - t_{run}) $$ [S1].
3. **Calculate Segment Weight**: Calculate the weight of the current segment, representing the area under the transmittance curve for this step:
   $$ w_{step} = T_r \cdot dt $$ [S1].
4. **Accumulate Weight**: Add the segment weight to the total accumulated weight:
   $$ w_{acc} = w_{acc} + w_{step} $$ [S1].
5. **Reservoir Update**: Perform weighted reservoir sampling to select a segment proportional to its weight $w_{step}$ relative to the total weight $w_{acc}$:
   - Generate a random number $u \sim U[0,1)$.
   - If $u \cdot w_{acc} < w_{step}$, update the reservoir to the current segment:
     $$ \text{reservoir} = (t_{run}, dt) $$ [S1].
6. **Update Transmittance Estimate**: Update the transmittance estimate for the next step based on the probability of interacting with a real particle:
   $$ T_r = T_r \cdot \left(1 - \frac{\sigma_t(t_{run})}{\bar{\sigma}}\right) $$ [S1].
7. **Advance Distance**:
   $$ t_{run} = t_{run} + dt $$ [S1].
8. **Loop Termination**: Repeat steps 1-7 while $t_{run} < t_{max}$ [S1].
9. **Final Sampling**: After the loop, retrieve the selected segment $(t_r, d_r)$ from the reservoir [S1]. Sample the final distance $t$ uniformly within this segment:
   $$ t = t_r + u \cdot d_r $$
   where $u \sim U[0,1)$ [S1].
10. **Return**: Return the sampled distance $t$ and the total accumulated weight $w_{acc}$ [S1].

### Tracking Function
The tracking function used in the ratio tracking process is piecewise constant:
$$ f_{RT}(t) = \prod_{t_i \leq \min(t, t_s)} \left(1 - \frac{\sigma_t(t_i)}{\bar{\sigma}}\right) $$
where $\sigma_t(t_i)/\bar{\sigma}$ is the probability of interacting with a real particle at step $i$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def drt_sample(t_max, sigma_t_func, majorant_sigma, rng):
    """
    Samples a distance t proportional to transmittance T(t) and returns
    the accumulated weight w_acc estimating the integral of T(t).
    
    Args:
        t_max: Maximum distance along the ray.
        sigma_t_func: Function returning extinction coefficient sigma_t(t).
        majorant_sigma: Majorant extinction coefficient sigma_bar.
        rng: Function returning uniform random number in [0, 1).
        
    Returns:
        t: Sampled distance.
        w_acc: Accumulated weight.
    """
    Tr = 1.0
    t_run = 0.0
    w_acc = 0.0
    reservoir = None
    
    while t_run < t_max:
        # Sample from the homogenized medium
        dt = -math.log(1.0 - rng()) / majorant_sigma
        dt = min(dt, t_max - t_run)
        
        # Propose current constant segment with weight equal to area under the curve
        w_step = Tr * dt
        w_acc += w_step
        
        # Update reservoir (happens at least once due to reservoir sampling logic)
        if rng() * w_acc < w_step:
            reservoir = (t_run, dt)
            
        # Update transmittance estimate
        sigma_t_at_current = sigma_t_func(t_run)
        Tr *= 1.0 - (sigma_t_at_current / majorant_sigma)
        
        # Advance the run distance
        t_run += dt
        
    # Final uniform sampling over the chosen segment
    t_r, d_r = reservoir
    t = t_r + rng() * d_r
    assert t <= t_max
    
    return t, w_acc
```

### Parameters
- `use_drt`: A boolean parameter that enables the Differential Ratio Tracking method. If false, the free-flight sampling-based baseline is used [S3, unofficial].
- `use_drt_mis`: A boolean parameter that combines the gradients estimated with DRT and the baseline using Multiple Importance Sampling (MIS) [S3, unofficial].
- `use_drt_subsampling`: A boolean parameter that reduces the usage of the DRT sampling method to once per path rather than once at every path vertex, reducing overall runtime from $O(n^2)$ to $O(n)$ [S3, unofficial].

### Edge Cases and Pitfalls
- **Zero Density Regions**: Standard free-flight sampling places samples proportional to the product of transmittance and density (extinction-weighted) [S1]. In empty or low-density regions, this leads to high variance for gradients because samples are rarely placed there, yet gradients may have non-trivial magnitude (e.g., to reconstruct an object where there was previously just air) [S1]. DRT mitigates this by sampling proportional to just transmittance [S1].
- **Local Minima**: Optimization of scattering parameters can get stuck in local minima [S1]. This can be alleviated by bootstrapping the optimization from nonphysical emissive volumes (similar to voxel-based radiance fields) [S1].
- **Majorant Selection**: The majorant $\bar{\sigma}$ must be greater than or equal to the maximum extinction coefficient $\sigma_t(t)$ along the ray to ensure valid sampling from the homogenized medium [S1].

## Operating characteristics
- **Time Complexity**: Runs in linear time with respect to the number of steps [S1]. When integrated with path replay backpropagation, it uses constant memory [S1]. However, if `use_drt_subsampling` is false, the runtime can be $O(n^2)$ where $n$ is the number of path vertices, because DRT is applied at every vertex [S3, unofficial]. With subsampling enabled, it reduces to $O(n)$ [S3, unofficial].
- **Memory Complexity**: Constant memory when integrated with path replay backpropagation, as it uses a reservoir to store only one segment [S1].
- **Performance**: DRT's Monte Carlo estimation becomes more expensive with higher sample counts (SPP), leading to increasing speedup advantages for pre-computed methods like [Single-Scattering Differentiable Volume Rendering](4143-single-scattering-differentiable-volume-rendering.md) at higher SPP [S2]. For example, at 512 SPP, the single-scattering method shows a 6.3x speedup over DRT [S2].
- **Strengths**: Unbiased, yields low-variance gradients, and handles empty/low-density regions better than standard free-flight sampling [S1].
- **Limitations**: Must recompute gradients for each view, unlike pre-computed methods [S2]. More expensive than single-scattering methods at high sample counts [S2].
- **When to Use**: When using physically-based scattering and absorbing volumes and when gradient variance in low-density regions is a concern [S1]. When using [Single-Scattering Differentiable Volume Rendering](4143-single-scattering-differentiable-volume-rendering.md) is not feasible due to multiple scattering or complex geometry [G].
- **When Not to Use**: When high sample counts are required and pre-computed methods are applicable, as DRT's cost scales with SPP [S2]. When using NeRF-style emissive volumes, as DRT is designed for scattering/absorbing volumes [S1].

## Background
Differential Ratio Tracing was introduced in the paper "Unbiased inverse volume rendering with differential trackers" [S1]. It was implemented as a baseline in Mitsuba 3 [S2]. The method addresses the limitations of standard differentiable volume rendering techniques that use free-flight sampling, which can produce biased and high-variance gradients [S1]. It is part of the family of differentiable rendering methods that use automatic differentiation in reverse/adjoint mode [Parent].

## Sources
[S1] Unbiased inverse volume rendering with differential trackers, https://doi.org/10.1145/3528223.3530073, fetched 2026-11-01
[S2] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S3] https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, https://github.com/rgl-epfl/unbiased-inverse-volume-rendering, fetched 2026-10-26 (unofficial)
