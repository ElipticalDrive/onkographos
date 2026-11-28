# Homogeneous Medium Sampling

Also called: Homogeneous Volume Tracking, Tracking in Homogeneous Volumes

## Overview
Homogeneous Medium Sampling determines the distance a ray travels through a participating medium with uniform optical properties before interacting, handling both scattering within the volume and intersection with surface-emitting geometry [S3, S7]. It differs from [Delta tracking](985-delta-tracking.md) by directly sampling the interaction distance from the extinction coefficient rather than using a majorant bound, and it serves as the foundational step for [Free-path sampling](1844-free-path-sampling.md) [S1, unofficial][S7].

## How it works
**Inputs and Outputs**
*   **Input:** A ray $r$ defined by origin $\mathbf{o}$, direction $\boldsymbol{\omega}$, and maximum distance $t_{\text{max}}$; the medium's absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and extinction coefficient $\sigma_t = \sigma_a + \sigma_s$; a random number generator or sampler [S6, unofficial][S7].
*   **Output:** A boolean indicating if a medium interaction occurred, and if so, a `MediumInteraction` structure containing the position $\mathbf{p}$, the sampled distance $t$, the phase function, and a `Spectrum` throughput weight to update the path radiance estimator [S7].

**Data Structures**
*   `MediumInteraction`: Contains fields for position $\mathbf{p}$, direction $-\boldsymbol{\omega}$, time, a pointer to the medium, and a pointer to the phase function (e.g., Henyey-Greenstein) [S7]. An `IsValid()` method returns true if the phase function pointer is non-null [S7].
*   `Spectrum`: A vector of values representing spectral channels (e.g., RGB) [S7]. In spectral MIS implementations, $\sigma_t$ and $\sigma_s$ are stored per channel [S7].

**Steps**
1.  **Select Spectral Channel:** If the medium supports spectral variation, uniformly select a channel index $c$ from the available spectral samples (e.g., $c \in \{0, 1, 2\}$ for RGB) [S7][S4, unofficial].
2.  **Sample Free Path:** Generate a uniform random number $u \in [0, 1)$ [S3][S6, unofficial]. Compute the collision-free distance $d$ using the inverse CDF of the exponential distribution for the selected channel's extinction coefficient $\sigma_t[c]$:
    $$ d = -\frac{\ln(\max(1 - u, 0))}{\sigma_t[c]} $$
    [S4, unofficial][S7].
3.  **Convert to Ray Parameter:** Convert the physical distance $d$ to the ray parameter $t$:
    $$ t = \min\left( \frac{d}{\|\boldsymbol{\omega}\|}, t_{\text{max}} \right) $$
    [S6, unofficial][S7].
4.  **Determine Interaction Type:** If $t < t_{\text{max}}$, a medium interaction occurs [S7]. If $t \ge t_{\text{max}}$, the ray exits the volume without scattering; the interaction is treated as a surface hit at $t_{\text{max}}$ [S3, S7].
5.  **Initialize Interaction (if medium hit):** Set position $\mathbf{p} = \mathbf{o} + t \boldsymbol{\omega}$ [S7]. Initialize the phase function (e.g., Henyey-Greenstein with asymmetry parameter $g$) [S7][S8, unofficial]. Compute the transmittance from the ray origin to $\mathbf{p}$:
    $$ T_r = e^{-\sigma_t t} $$
    [S3, S7]. Compute the probability density function (PDF) for the sampled distance. For a single channel, $p(t) = \sigma_t e^{-\sigma_t t}$ [S7]. For spectral sampling with uniform channel selection probability $p_c = 1/n$, the combined PDF is the average of the channel densities:
    $$ \hat{p}_t(t) = \frac{1}{n} \sum_{i=1}^{n} \sigma_t^i e^{-\sigma_t^i t} $$
    [S7].
6.  **Compute Throughput Weight:** If a medium interaction occurred:
    $$ \beta = \frac{T_r \cdot \sigma_s}{\hat{p}_t(t)} $$
    [S7]. If the ray exited (surface hit):
    $$ \beta = \frac{T_r}{\hat{p}_t(t_{\text{max}})} $$
    [S7].
7.  **Return:** Return the boolean flag and the weight [S7]. If a medium interaction occurred, the caller must also sample a new direction $\boldsymbol{\omega}'$ using the phase function and the incident direction to continue the path [S5, S6, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Spectrum(list):
    """Simple list-based spectrum for spectral MIS."""
    def __mul__(self, other):
        return Spectrum([a * b for a, b in zip(self, other)])
    def __truediv__(self, other):
        return Spectrum([a / other for a in self])

def sample_homogeneous_medium(ray, sigma_a, sigma_s, sampler, n_channels=3):
    """
    Samples a homogeneous medium for a given ray.
    Args:
        ray: Object with .origin (list), .direction (list), .t_max (float).
        sigma_a, sigma_s: Spectrum objects of length n_channels.
        sampler: Object with .get_1d() returning uniform [0, 1).
        n_channels: Number of spectral channels.
    Returns:
        (hit_medium, interaction, weight)
    """
    sigma_t = Spectrum([a + s for a, s in zip(sigma_a, sigma_s)])
    
    # 1. Select spectral channel
    u_channel = sampler.get_1d()
    c = int(u_channel * n_channels)
    c = min(c, n_channels - 1)
    
    # 2. Sample free path distance for selected channel
    u_dist = sampler.get_1d()
    d = -math.log(max(1.0 - u_dist, 0.0)) / sigma_t[c]
    
    # 3. Convert to ray parameter t
    ray_len = math.sqrt(sum(x * x for x in ray.direction))
    t = min(d / ray_len, ray.t_max)
    
    # 4. Determine if medium interaction occurred
    hit_medium = t < ray.t_max
    
    # 5. Compute Transmittance
    t_calc = t if hit_medium else ray.t_max
    tr = Spectrum([math.exp(-sigma_t[i] * t_calc / ray_len) for i in range(n_channels)])
    
    # 6. Compute PDF
    # density[i] = sigma_t[i] * tr[i] if hit else tr[i]
    density = Spectrum([(sigma_t[i] * tr[i] if hit_medium else tr[i]) for i in range(n_channels)])
    pdf = sum(density) / n_channels
    
    # 7. Compute Throughput Weight
    weight = Spectrum([(tr[i] * sigma_s[i] / pdf if hit_medium else tr[i] / pdf) for i in range(n_channels)])
    
    # 8. Initialize Interaction if hit
    interaction = None
    if hit_medium:
        pos = [ray.origin[i] + t * ray.direction[i] for i in range(3)]
        interaction = {'pos': pos, 'phase_func': HenyeyGreenstein(g=0.0), 'valid': True}
        
    return hit_medium, interaction, weight

class HenyeyGreenstein:
    """Placeholder for phase function initialization."""
    def __init__(self, g):
        self.g = g
```

**Parameters**
*   $\sigma_a, \sigma_s$: Absorption and scattering coefficients defining how much light is absorbed or scattered per unit distance [S5, unofficial].
*   $g$: Henyey-Greenstein asymmetry parameter, range $[-1, 1]$ [S8, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** To avoid $\ln(0)$, clamp the logarithm argument using $\max(1 - u, 0)$ as shown in the sampling formula [S4, unofficial].
*   **Boundary Handling:** If the sampled distance $t$ exceeds the ray's maximum extent ($t_{\text{max}}$), the ray is considered to have hit the surface/boundary [S7]. The transmittance to the surface must be used, and the surface emission $L_e$ is evaluated [S3, S7].
*   **Spectral Variation:** If $\sigma_t$ varies by wavelength, selecting a single channel for distance sampling introduces bias if not handled by MIS [S7]. The PDF must be averaged over all channels to ensure unbiasedness [S7].
*   **Anisotropy:** Highly anisotropic media ($g \approx 1$) require careful phase function sampling to avoid noise, as the scattering is concentrated in a narrow cone [S2].

## Subtopics

### [Free-path sampling](1844-free-path-sampling.md)
Free-path sampling is a technique for simulating light transport in participating media by randomly determining interaction distances based on the medium's extinction coefficient, allowing rays to bounce within the volume before potentially hitting a surface. It differs from its siblings by explicitly tracking the stochastic flight distance through the homogeneous medium to handle volume interactions (such as single scattering) and surface hits within the same traversal logic, whereas [Constant Distance Sampling](12250-constant-distance-sampling.md) uses fixed intervals and [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md) handles spatially varying coefficients.

### [Majorant Iterator Ray Sampling](3075-majorant-iterator-ray-sampling.md)
Majorant Iterator Ray Sampling is a concrete implementation of ray sampling in participating media that handles variable density and null scattering by using stack-allocated iterators. It avoids dynamic allocation and dynamic dispatch overhead by directly calling the medium's `SampleRay` method and iterating through `RayMajorantSegment` objects. It differs from generic free-path sampling by explicitly managing segment boundaries and transmittance accumulation across heterogeneous regions defined by the medium's majorant iterator.

### [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md)
Heterogeneous Medium Sampling determines the distance to the next interaction in a participating medium where optical properties vary spatially, such as in a `GridDensityMedium`. It generalizes homogeneous sampling by integrating the spatially varying extinction coefficient $\sigma_t$ along the ray to compute transmittance and sample an interaction distance. Unlike regular tracking, which applies homogeneous techniques voxel-by-voxel and is costly with many voxels, this method handles continuous spatial variation and linear interpolation directly.

### [Constant Distance Sampling](12250-constant-distance-sampling.md)
Constant Distance Sampling is a Monte Carlo light tracing technique for rendering participating media that advances light bundles in discrete steps of fixed length $\delta$. It solves the problem of simulating multiple scattering in homogeneous media by approximating the continuous transport equation with a grid-aligned discrete process. Unlike [Free-path sampling](1844-free-path-sampling.md), which samples interaction distances from the exact transmittance distribution, Constant Distance Sampling restricts scattering events to multiples of $\delta$, introducing a deterministic bias in exchange for simplified storage and traversal logic. It is distinct from [Majorant Iterator Ray Sampling](3075-majorant-iterator-ray-sampling.md) in its use of a fixed step rather than a majorant-based rejection loop, and from [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md) as it assumes constant coefficients within the medium.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray segment [S7].
*   **Memory Complexity:** $O(1)$ auxiliary memory [S7].
*   **Performance:** Highly efficient for homogeneous volumes [S7]. Convergence can be slow for high-order scattering media (many interactions) or highly anisotropic media, requiring many samples or variance reduction techniques like Next Event Estimation (NEE) [S2][S5, unofficial].
*   **Strengths:** Unbiased, simple to implement, exact for homogeneous media [S7].
*   **Limitations:** Inefficient for heterogeneous media (requires integration or majorant tracking) [S7]. Noisy for highly scattering or anisotropic media without additional techniques [S2].
*   **When to use:** Scenes with uniform fog, smoke, or clouds where the medium properties do not vary spatially [S5, S8, unofficial].
*   **When not to use:** Scenes with complex, spatially varying density distributions (use [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md)) or when performance is critical and the medium is sparse (consider [Delta tracking](985-delta-tracking.md) if the majorant is tight, though homogeneous sampling is optimal for truly homogeneous media) [S7].

## Background
Homogeneous medium sampling is a fundamental technique in volumetric rendering, derived from the Beer-Lambert law and the theory of radiative transfer [S3][S5, unofficial]. It was formalized in the context of Monte Carlo path tracing for participating media, notably in the work of Lafortune and Willems [S2] and later detailed in standard references like *Physically Based Rendering* (PBRT) [S7]. The method relies on the statistical property that the distance to the next interaction in a homogeneous medium follows an exponential distribution [S5, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S3] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S4] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S5] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S6] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S7] Light Transport II: Volume Rendering / Sampling Volume Scattering, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/Sampling_Volume_Scattering.html, fetched 2026-10-28
[S8] Constant-density participating media (fog), https://github.com/adriendjn/mypbr/pull/4, fetched 2026-10-28 (unofficial)
