# Homogeneous Medium Sampling

Also called: Homogeneous Volume Tracking, Tracking in Homogeneous Volumes

## Overview
Homogeneous Medium Sampling determines the distance to the next interaction (scattering or absorption) for a ray traversing a participating medium with uniform optical properties [S1, unofficial][S4, unofficial]. It solves the problem of efficiently simulating light transport in volumes like fog or smoke by probabilistically selecting interaction points based on the extinction coefficient, rather than integrating over the entire volume [S4, unofficial]. It differs from [Delta tracking](985-delta-tracking.md) by using the exact inverse CDF of the exponential distribution for constant density, which is more efficient than rejection sampling, and from [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md) by assuming constant coefficients, allowing for closed-form solutions without numerical integration [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin, $\mathbf{d}$ is the direction, and $t_{\text{max}}$ is the maximum distance (often the distance to the nearest surface intersection or medium boundary) [S6, unofficial].
*   **Input:** Medium coefficients: absorption $\sigma_a$, scattering $\sigma_s$, and extinction $\sigma_t = \sigma_a + \sigma_s$ [S4, unofficial][S5, unofficial]. These are constant throughout the volume [S4, unofficial]. For spectral rendering, these are vectors per wavelength channel [S6, unofficial].
*   **Input:** Random samples from a sampler.
*   **Output:** A `MediumInteraction` object if a scattering event occurs within $t_{\text{max}}$, or a signal indicating the ray exits the medium without interaction (hitting the surface/boundary) [S6, unofficial].
*   **Output:** A throughput weight (Spectrum) to update the path contribution [S6, unofficial].

**Data Structures**
*   **MediumInteraction:** Stores position $\mathbf{p}$, incoming direction $\mathbf{d}$, time, pointer to the medium, and a phase function instance (e.g., Henyey-Greenstein) [S6, unofficial]. The `IsValid()` method checks if the phase function pointer is non-null [S6, unofficial].
*   **Spectrum:** A vector of floating-point values representing radiance/coefficients per spectral channel (typically 3 for RGB) [S6, unofficial].

**Steps**
1.  **Select Spectral Channel (Spectral MIS):** If the medium has wavelength-dependent coefficients, uniformly select a channel index $c \in [0, N-1]$ where $N$ is the number of samples [S6, unofficial].
    *   $c = \lfloor N \cdot u_1 \rfloor$, clamped to $[0, N-1]$ [S6, unofficial].
    *   Probability of selecting channel $c$ is $p_{\text{wavelength}} = 1/N$ [S6, unofficial].
2.  **Sample Free Path Length:** Generate a uniform random number $u_2 \in [0, 1)$ [S6, unofficial]. Compute the distance $d$ to the next interaction using the inverse CDF of the exponential distribution for the selected channel's extinction coefficient $\sigma_t[c]$ [S1, unofficial][S4, unofficial][S6, unofficial]:
    $$ d = -\frac{\ln(1 - u_2)}{\sigma_t[c]} $$
    *   Using $1-u_2$ keeps the logarithm argument in $(0, 1]$, avoiding $\ln(0)$ when $u_2=0$ [S6, unofficial].
3.  **Convert to Ray Parameter:** Compute the distance along the ray $t$ [S6, unofficial]:
    $$ t = \min\left( d \|\mathbf{d}\|, t_{\text{max}} \right) $$
    *   If $\mathbf{d}$ is normalized, $\|\mathbf{d}\|=1$, so $t = \min(d, t_{\text{max}})$ [S6, unofficial].
4.  **Determine Interaction Type:**
    *   If $t < t_{\text{max}}$, a medium interaction occurs [S6, unofficial].
    *   Otherwise, the ray exits the medium and the surface interaction at the boundary is used [S3].
5.  **Handle Medium Interaction:**
    *   Set position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S6, unofficial].
    *   Initialize `MediumInteraction` with $\mathbf{p}$, incoming direction $-\mathbf{d}$, and a phase function (e.g., Henyey-Greenstein with asymmetry parameter $g$) [S6, unofficial].
    *   Compute transmittance from origin to $\mathbf{p}$:
        $$ T_r = e^{-\sigma_t \cdot d_{\text{traveled}}} $$
        where $d_{\text{traveled}} = t \|\mathbf{d}\|$ [S6, unofficial].
    *   Determine if the event is scattering or absorption [S4, unofficial]:
        *   Generate $u_3 \in [0, 1]$.
        *   If $u_3 < \frac{\sigma_s}{\sigma_t}$, it is scattering [S4, unofficial]. The ray continues.
        *   Else, it is absorption. The ray terminates [S4, unofficial].
    *   If scattering, sample a new outgoing direction $\omega$ using the phase function [S4, unofficial].
6.  **Handle Surface/Boundary Exit:**
    *   If no medium interaction occurred, the ray hits the surface at $t_{\text{max}}$ [S3].
    *   The contribution is attenuated by the transmittance through the medium to the surface: $T_r = e^{-\sigma_t \cdot t_{\text{max}} \|\mathbf{d}\|}$ [S3].
7.  **Compute PDF and Weight:**
    *   The PDF for the sampled distance $t$ (given channel $c$) is:
        $$ p_t(t) = \sigma_t[c] e^{-\sigma_t[c] \cdot d_{\text{traveled}}} $$
    *   The marginal PDF for the channel selection is $1/N$ [S6, unofficial].
    *   The combined PDF for a medium interaction is:
        $$ p_{\text{med}} = \frac{1}{N} \sigma_t[c] e^{-\sigma_t[c] \cdot d_{\text{traveled}}} $$
    *   The probability of hitting the surface (no medium interaction) is the complement of the integral of the medium PDF over $[0, t_{\text{max}}]$:
        $$ p_{\text{surf}} = e^{-\sigma_t[c] \cdot t_{\text{max}} \|\mathbf{d}\|} $$
    *   The returned throughput weight $\beta$ updates the path:
        *   If medium interaction: $\beta = \frac{T_r \cdot \sigma_s}{p_{\text{med}}}$ [S6, unofficial]. (Note: $\sigma_s$ accounts for the scattering coefficient in the source term, and the phase function PDF is handled separately during direction sampling or included in the estimator if using direct lighting sampling) [S6, unofficial].
        *   If surface hit: $\beta = \frac{T_r}{p_{\text{surf}}}$ [S6, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_homogeneous_medium(ray, sampler, sigma_a, sigma_s, g, n_channels=3):
    """
    Samples a homogeneous medium for a given ray.
    
    Args:
        ray: Ray object with origin 'o', direction 'd', and max distance 't_max'.
        sampler: Sampler object with method 'get1d()' returning uniform [0,1).
        sigma_a: Absorption coefficient (Spectrum or Vec3).
        sigma_s: Scattering coefficient (Spectrum or Vec3).
        g: Henyey-Greenstein asymmetry parameter.
        n_channels: Number of spectral channels.
        
    Returns:
        tuple: (interaction_occurred: bool, medium_interaction: dict or None, throughput: Spectrum)
    """
    sigma_t = sigma_a + sigma_s
    
    # 1. Select spectral channel uniformly
    u1 = sampler.get1d()
    channel = int(u1 * n_channels)
    channel = max(0, min(channel, n_channels - 1))
    pmf_wavelength = 1.0 / n_channels
    
    # 2. Sample free path length for selected channel
    u2 = sampler.get1d()
    # Use 1-u2 to keep argument in (0, 1], avoiding log(0)
    dist = -math.log(1.0 - u2) / sigma_t[channel]
    
    # 3. Convert to ray parameter t
    ray_len = math.sqrt(dot(ray.d, ray.d))
    t = min(dist * ray_len, ray.t_max)
    
    # 4. Determine if medium interaction occurred
    sampled_medium = t < ray.t_max
    
    # 5. Compute Transmittance
    d_traveled = t * ray_len
    tr = math.exp(-sigma_t * d_traveled)
    
    # 6. Compute PDFs (Spectral MIS)
    # Density per channel: sigma_t * Tr if medium, else Tr
    density = sigma_t * tr if sampled_medium else tr
    pdf = sum(density) / n_channels
    
    # 7. Compute Throughput Weight and Interaction
    if sampled_medium:
        throughput = (tr * sigma_s) / pdf
        # Initialize MediumInteraction with position, direction, medium, and phase function
        mi = {
            'position': ray.o + ray.d * t,
            'direction': -ray.d,
            'medium': None, # Reference to medium instance
            'phase_function': {'type': 'HenyeyGreenstein', 'g': g}
        }
        return True, mi, throughput
    else:
        throughput = tr / pdf
        return False, None, throughput
```

**Parameters**
*   $\sigma_a, \sigma_s$: Absorption and scattering coefficients [S4, unofficial][S5, unofficial].
*   $g$: Henyey-Greenstein asymmetry parameter. $g=0$ is isotropic, $g>0$ forward scattering, $g<0$ backward scattering. Values range from $-1$ to $1$ [S6, unofficial].
*   $t_{\text{max}}$: Distance to the nearest surface or medium boundary [S6, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** If $u_2=0$, $\ln(1)$ is 0, leading to $d=0$. Using $1-u_2$ keeps the argument in $(0,1]$ and avoids $\ln(0)$ [S6, unofficial].
*   **Ray Normalization:** Shadow rays must be normalized before calculating distances; unnormalized rays cause incorrect self-hit offsets and visual artifacts [S7].
*   **Spectral Consistency:** When using spectral MIS, the PDF must account for the channel selection probability ($1/N$) [S6, unofficial].
*   **Boundary Handling:** If the ray originates inside the medium, $t_{\text{max}}$ is the distance to the exit boundary [S6, unofficial].

## Subtopics

### [Free-path sampling](1844-free-path-sampling.md)
Generalizes the distance sampling to heterogeneous media by integrating the extinction coefficient. Homogeneous sampling is the base case where the integral simplifies to $\sigma_t t$.

### [Majorant Iterator Ray Sampling](3075-majorant-iterator-ray-sampling.md)
Handles heterogeneous media by iterating through segments with a majorant bound. Homogeneous sampling is a single-segment case with no iteration overhead.

### [Heterogeneous Medium Sampling](6654-heterogeneous-medium-sampling.md)
Handles spatially varying density. Homogeneous sampling is a special case where $\sigma_t$ is constant, allowing the use of the simple exponential inverse CDF instead of numerical integration.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray segment. It involves a few arithmetic operations and a logarithm [S4, unofficial].
*   **Memory Complexity:** $O(1)$. No additional data structures are required beyond the ray and medium coefficients [S6, unofficial].
*   **Performance:** Extremely fast compared to delta tracking or heterogeneous sampling, as it avoids rejection loops or numerical integration [S4, unofficial].
*   **Strengths:** Exact sampling for constant density; unbiased; efficient for simple volumes like fog or smoke [S4, unofficial][S7].
*   **Limitations:** Only applicable to homogeneous media. Cannot handle complex density variations without voxelization or approximation [S1, unofficial].
*   **When to use:** Scenes with uniform participating media (e.g., fog, smoke, clouds with constant density) [S7].
*   **When not to use:** Heterogeneous media (e.g., smoke plumes with varying density) [S1, unofficial].

## Background
Homogeneous medium sampling is a fundamental technique in volumetric rendering, derived from the Beer-Lambert law and the theory of radiative transfer [S1, unofficial][S4, unofficial]. It was formalized in early computer graphics literature for simulating atmospheric effects like fog and smoke [S7]. The method is widely used in path tracers and photon mapping algorithms for participating media [S4, unofficial][S5, unofficial]. It is implemented in standard rendering libraries and educational path tracers [S3][S6, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S3] PDF Volumetric Path Tracing - University of California, San Diego, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S4] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S5] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S6] PDF Volumetric Path Tracing, https://jugoodma.github.io/docs/740-paper.pdf, fetched 2026-10-27 (unofficial)
[S7] Light Transport II: Volume Rendering / Sampling Volume Scattering, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/Sampling_Volume_Scattering.html, fetched 2026-10-28
[S8] Constant-density participating media (fog), https://github.com/adriendjn/mypbr/pull/4, fetched 2026-10-28 (unofficial)
