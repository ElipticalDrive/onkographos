# Delta Tracking (Uniform Majorant)

Also called: DeltaTrackingIter<false>, Simple Delta Tracking

## Overview
Delta Tracking (Uniform Majorant) is a simplified Monte Carlo method for simulating photon transport in participating media by assuming a uniform majorant extinction coefficient of 1.0 [S1, unofficial]. It solves the problem of sampling free paths in heterogeneous volumes by homogenizing the density to an upper bound, but specifically restricts this bound to a constant value, allowing direct sampling from the exponential distribution without rejection sampling for the distance itself [S1, unofficial]. It sets itself apart from [Ratio Tracking with Null Scattering](1044-ratio-tracking-with-null-scattering.md) and [Woodcock Tracking with Variable Majorants](1084-woodcock-tracking-with-variable-majorants.md) by eliminating the need to evaluate the local extinction coefficient at every step or perform rejection tests against a variable majorant, at the cost of significant inefficiency in non-uniform media [S1, unofficial].

## How it works
**Inputs:**
- `rng`: A random number generator capable of producing uniform floats in $[0, 1)$.
- `ray_tnear`: The near clipping plane distance of the ray.
- `ray_tfar`: The far clipping plane distance of the ray.
- `ray_t`: The current accumulated distance along the ray (initially 0 or the previous step's end).

**Outputs:**
- `rayt`: The absolute distance to the next interaction point.
- `bool`: `true` if the interaction occurs within the ray segment $[ray\_tnear, ray\_tfar]$, `false` otherwise.

**Data Structures:**
- No complex data structures are required. The method relies on scalar constants.

**Constants:**
- `sigma_t = 1.0`: The total extinction coefficient.
- `majorant = 1.0`: The uniform majorant density/extinction bound.

**Steps:**
1. Initialize constants: Set `sigma_t = 1.0` and `majorant = 1.0` [S1, unofficial].
2. Sample a uniform random number $u$ from the range $[0, 1)$ using `rng.get_floats().x` [S1, unofficial].
3. Calculate the distance to the next interaction $t$ using the inverse CDF of the exponential distribution:
   $$ t = \frac{-\ln(1.0 - u)}{majorant \cdot \sigma_t} $$
   Since `majorant` and `sigma_t` are 1.0, this simplifies to $t = -\ln(1.0 - u)$ [S1, unofficial].
4. Calculate the absolute ray position:
   $$ rayt = ray\_tnear + t $$
5. Check bounds: Return `true` if $rayt \le ray\_tfar$, otherwise return `false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delta_tracking_uniform_majorant(rng, ray_tnear, ray_tfar, ray_t):
    """
    Simulates a single step in Delta Tracking with a uniform majorant of 1.0.
    
    Args:
        rng: Random number generator with method get_floats() returning a vector-like object with .x attribute.
        ray_tnear: Start distance of the ray segment.
        ray_tfar: End distance of the ray segment.
        ray_t: Current accumulated distance (not used in calculation of t, but part of context).
        
    Returns:
        tuple: (rayt, hit) where rayt is the absolute distance to interaction, 
               and hit is a boolean indicating if the interaction is within bounds.
    """
    # Constants as defined in [S1, unofficial]
    sigma_t = 1.0
    majorant = 1.0
    
    # Sample uniform random number u in [0, 1)
    # Note: rng.get_floats().x is assumed to return a float in [0, 1)
    u = rng.get_floats().x
    
    # Calculate distance to next interaction t
    # Formula: t = -log(1 - u) / (majorant * sigma_t)
    # Since majorant=1.0 and sigma_t=1.0:
    t = -math.log(1.0 - u)
    
    # Calculate absolute ray position
    rayt = ray_tnear + t
    
    # Check bounds
    hit = rayt <= ray_tfar
    
    return rayt, hit
```

**Edge Cases and Pitfalls:**
- **Inefficiency in Heterogeneous Media:** The method assumes `majorant` and `sigma_t` are constant at 1.0. If the actual media has varying density or if the majorant is much larger than the actual extinction, the method becomes inefficient because it does not account for local variations, leading to many rejected samples if used in a broader rejection-tracking context, or simply inaccurate physics if used as a pure distance sampler without correction [S1, unofficial].
- **Precision:** The calculation $1.0 - u$ can suffer from precision issues if $u$ is extremely close to 1.0. Using $-\ln(u)$ where $u$ is sampled from $(0, 1]$ is often numerically more stable, but the source specifies $1.0 - u$ [S1, unofficial].

## Subtopics

### Exponential Sampling
A strategy for choosing propagation distances in homogeneous media where sampling is exponentially-decreasing with a PDF proportional to the source term $Q(t)$ [S2, unofficial]. It effectively places dipole sources at different depths and weights them according to the inverse exponential PDF, canceling out the exponential variation in $Q(t)$ [S2, unofficial]. The formula is $t_i = \frac{\log(1 - \xi_i)}{-t_0}$ with $pdf_{exp}(t_i) = t_0 e^{-t_0 t_i}$, where $\xi_i \in [0, 1)$ is a uniform random number [S2, unofficial]. This PDF only depends on $t_0$ and is independent of $\vec{x}$ and $\vec{\omega}$ [S2, unofficial]. Exponential sampling guarantees that the tail of the profile converges when taking just 1 sample, but it underestimates the peak near $x=0$ and introduces smoothing [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per step, as it involves only basic arithmetic and a logarithm.
- **Memory Complexity:** $O(1)$, requiring no storage for density fields or majorant grids.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Does not scale well with heterogeneous media. The uniform majorant assumption makes it unsuitable for scenes with high contrast in extinction coefficients.
- **Strengths:** Extremely simple to implement; no need to query volume textures or compute local extinction.
- **Limitations:** Highly inefficient for media where the actual extinction is significantly lower than the majorant (which is fixed at 1.0 here, implying it only works well if the media's extinction is close to 1.0 everywhere). It fails to capture the benefits of delta tracking's ability to handle heterogeneity efficiently [S1, unofficial].
- **When to use:** Only when the medium is homogeneous with an extinction coefficient of exactly 1.0, or as a baseline for comparison.
- **When not to use:** For heterogeneous volumes or when the extinction coefficient varies significantly from 1.0.

## Background
The method originates from the Open Volume Renderer project, specifically the `DeltaTrackingIter<false>` implementation, which simplifies the general delta tracking approach by fixing the majorant and extinction to 1.0 [S1, unofficial]. It is a specific instance of the broader delta tracking family, which was developed to handle participating media with varying density by using a majorant to homogenize the medium [Parent]. The concept of exponential sampling for homogeneous media is also discussed in the context of photon beam diffusion, where it is contrasted with equiangular sampling [S2, unofficial].

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S2] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
