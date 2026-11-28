# Monte Carlo Volume Rendering

Also called: Monte Carlo Scattering (MCS), Stochastic Volume Rendering

## Overview
Monte Carlo Volume Rendering is a stochastic approach to volumetric visualization that avoids uniform step sampling by using random distance sampling and null-collision techniques to estimate light transport in participating media [S1, S2, S3, unofficial]. It differs from its siblings, such as [Uniform Ray Sampling](1261-uniform-ray-sampling.md) and [Ray Casting with Jittered Sampling](4151-ray-casting-with-jittered-sampling.md), by explicitly modeling scattering events via random walks (photon tracing) or stochastic ray casting with exponential distance sampling, allowing for accurate handling of anisotropic scattering and complex light transport at the cost of convergence noise [S2, S3, unofficial].

## How it works
The method is implemented in two primary variants found in the sources: a GPU-based ray casting approach using exponential distance sampling [S3, unofficial] and a shader-based photon tracing approach using null-collision Monte Carlo [S2, unofficial]. Both variants rely on stochastic sampling to determine interaction points within the volume [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Volume data: 3D texture (`sampler3D`) containing density/opacity values [S2, unofficial].
*   Transfer function: 2D texture mapping density to color/opacity [S2, unofficial].
*   Environment map: 2D texture for background/lighting [S2, unofficial].
*   Camera parameters: Inverse Model-View-Projection (MVP) matrix for ray unprojection [S2, unofficial].
*   Volume parameters: Extinction coefficient ($\sigma_t$), albedo ($\omega$), anisotropy bias ($g$), majorant ratio [S2, unofficial].
*   Random seed: Derived from fragment position and frame offset [S2, S3, unofficial].

**Outputs:**
*   Accumulated color buffer (for ray casting) or photon state buffers (position, direction, transmittance, radiance, bounces, samples) for photon tracing [S2, S3, unofficial].

### Data Structures
**Photon State (Photon Tracing Variant):**
Stored in output buffers as `vec4` components:
*   `position`: `vec3` (photon position in world space).
*   `direction`: `vec3` (normalized scattering direction).
*   `bounces`: `uint` (stored in `direction.w` or separate buffer).
*   `transmittance`: `vec3` (accumulated transmittance weight).
*   `radiance`: `vec3` (accumulated radiance estimate).
*   `samples`: `uint` (stored in `radiance.w` or separate buffer) [S2, unofficial].

**Random Number Generation:**
A hash-based pseudo-random number generator is used, typically defined as [S2, unofficial]:
$$ rand(p) = fract( (cos(M \times p)_x, sin(M \times p)_y) \times D ) $$
where $M$ is a 2x2 matrix of constants and $D$ is a vec2 of constants [S2, unofficial].

### Algorithm: Stochastic Ray Casting (Exponential Sampling)
This variant traces rays from the camera, stochastically determining the first scattering event [S3, unofficial].

1.  **Ray Initialization:** Unproject screen coordinates to get ray origin $v_{RayFrom}$ and direction endpoint $v_{RayTo}$ using the inverse MVP matrix [S2, S3, unofficial].
2.  **Volume Intersection:** Intersect the ray with the volume bounding box (cube) [S3, unofficial]. Calculate entry distance $t_{bounds}.x$ and exit distance $t_{bounds}.y$ [S2, unofficial]. If $t_{bounds}.x \ge t_{bounds}.y$, the ray misses; sample the environment map and return [S2, S3, unofficial].
3.  **Random Seed:** Initialize random state using a hash of the pixel position and a frame offset [S2, S3, unofficial].
4.  **Stochastic Distance Sampling:** Determine the distance to the first scattering event using the exponential distribution [S3, unofficial]:
    *   Initialize $dist = 0.0$.
    *   Calculate $invSigmaMax = 1.0 / \sigma_{max}$, where $\sigma_{max}$ is the maximum extinction coefficient [S2, S3, unofficial].
    *   Loop:
        *   Generate random number $seed$.
        *   Update distance: $dist_{new} = dist_{old} - \frac{\log(1 - seed.x)}{\sigma_{max}}$ [S2, unofficial].
        *   If $dist > maxDistance$ (distance to volume exit), break loop [S2, unofficial].
        *   Sample volume color at $P = mix(v_{RayFrom}, v_{RayTo}, dist \times invMaxDistance)$ [S2, unofficial].
        *   Check for scattering: If $seed.y < \alpha_{sample} \times invSigmaMax$, break loop and return $dist$ [S2, unofficial].
5.  **Scattering Event:**
    *   If $dist > maxDistance$, sample environment map and return [S3, unofficial].
    *   Calculate scattering point $P_{scatter} = mix(v_{RayFrom}, v_{RayTo}, dist / maxDistance)$ [S2, unofficial].
    *   Sample diffuse color: $C_{diffuse} = sampleVolumeColor(P_{scatter})$ [S2, S3, unofficial].
    *   Sample light color from environment map in the scattering direction: $C_{light} = sampleEnvironmentMap(u_{ScatteringDirection})$ [S2, S3, unofficial].
    *   Calculate transmittance from $P_{scatter}$ to the volume exit in the scattering direction using `sampleTransmittance` [S2, S3, unofficial].
6.  **Transmittance Calculation:**
    *   Initialize $T = 1.0$.
    *   Loop similar to distance sampling:
        *   Generate random seed.
        *   Update distance.
        *   If $dist > maxDistance$, break.
        *   Update transmittance: $T_{new} = T_{old} \times (1 - \alpha_{sample} \times invSigmaMax)$ [S2, unofficial].
    *   Return $T$ [S2, unofficial].
7.  **Final Color:** $C_{final} = C_{diffuse} \times C_{light} \times T$ [S2, S3, unofficial].
8.  **Accumulation:** Average the current frame with previous frames to reduce noise [S2, unofficial]:
    $$ AvgRender = AvgRender + \frac{NewRender - AvgRender}{numRenders} $$
    where $numRenders$ is the current frame number [S2, unofficial].

### Algorithm: Photon Tracing (Null-Collision Monte Carlo)
This variant uses shaders to simulate photon paths through the volume, accumulating radiance estimates via multiple bounces [S2, unofficial].

1.  **Photon Initialization (Reset Shader):**
    *   Unproject screen position to get ray origin $from$ and direction $to$ [S3, unofficial]. Normalize $to$.
    *   Intersect ray with unit cube to get entry bounds $t_{bounds}$ [S2, unofficial].
    *   Set $photon.position = from + t_{bounds}.x \times direction$ [S2, unofficial].
    *   Initialize $photon.transmittance = vec3(1.0)$, $photon.radiance = vec3(0.0)$, $bounces = 0$, $samples = 0$ [S2, unofficial].
2.  **Photon Tracing Loop (Compute Shader):**
    *   Loop for $i$ from 0 to $uSteps$ (number of integration steps) [S2, unofficial]:
        *   Generate random number $r$ [S2, unofficial].
        *   Compute step size $t = -\frac{\log(r.x)}{\sigma_{majorant}}$, where $\sigma_{majorant}$ is the majorant density [S2, unofficial].
        *   Update position: $photon.position += t \times photon.direction$ [S2, unofficial].
        *   Sample volume color at new position to get absorption ($\mu_{abs}$), scattering ($\mu_{scat}$), and null collision ($\mu_{null}$) coefficients [S2, unofficial]:
            *   $\mu_{abs} = volumeSample.a \times uAbsorptionCoefficient$ [S2, unofficial].
            *   $\mu_{scat} = volumeSample.a \times uScatteringCoefficient$ [S2, unofficial].
            *   $\mu_{null} = \sigma_{majorant} - \mu_{abs} - \mu_{scat}$ [S2, unofficial].
            *   $\sigma_{majorant} = \mu_{abs} + \mu_{scat} + |\mu_{null}|$ [S2, unofficial].
        *   Compute probabilities:
            *   $P_{null} = \frac{|\mu_{null}|}{\sigma_{majorant}}$ [S2, unofficial].
            *   $P_{abs} = \frac{\mu_{abs}}{\sigma_{majorant}}$ [S2, unofficial].
            *   $P_{scat} = \frac{\mu_{scat}}{\sigma_{majorant}}$ [S2, unofficial].
        *   **Bounds Check:** If $photon.position$ is outside the volume cube, sample environment map, add to radiance estimate (running average), reset photon, and continue to next step [S2, unofficial].
        *   **Collision Type:**
            *   If $r.y < P_{abs} + P_{scat}$ (Scattering/Absorption event):
                *   Generate new random $r$.
                *   If $r.y < P_{abs}$, photon is absorbed (transmittance becomes 0, stop) [S2, unofficial].
                *   If $r.y \ge P_{abs}$ (Scattering):
                    *   Update transmittance: $T_{new} = T_{old} \times volumeSample.rgb \times \frac{\mu_{scat}}{\sigma_{majorant} \times P_{scat}}$ [S2, unofficial].
                    *   Sample new direction using Henyey-Greenstein phase function with bias $g$ [S2, unofficial].
                    *   Increment $bounces$ [S2, unofficial].
            *   Else (Null Collision):
                *   Update transmittance: $T_{new} = T_{old} \times \frac{|\mu_{null}|}{\sigma_{majorant} \times P_{null}}$ [S2, unofficial].
3.  **Output:** Write photon state to buffers. Radiance is accumulated over multiple samples [S2, unofficial].

### Parameters
*   $\sigma_{max}$ / $uSigmaMax$: Maximum extinction coefficient for inverse probability sampling. Typical value: positive float representing max volume density [S2, S3, unofficial].
*   $uExtinction$: Extinction coefficient for exponential distance sampling [S3, unofficial].
*   $uAlphaCorrection$: Scaling factor for sampled alpha values [S2, unofficial].
*   $uScatteringDirection$: Direction vector for secondary scattering ray (ray casting variant) [S2, S3, unofficial].
*   $\sigma_{majorant}$ / $uMajorant$: Majorant density for null-collision sampling. Typically $\max(\mu_{abs} + \mu_{scat})$ [S2, unofficial].
*   $uAbsorptionCoefficient$: Base absorption coefficient multiplier (default 1) [S2, unofficial].
*   $uScatteringCoefficient$: Base scattering coefficient multiplier (default 1) [S2, unofficial].
*   $uScatteringBias$ / $scatteringBias$: Anisotropy parameter $g$ for Henyey-Greenstein (default 0) [S2, unofficial].
*   $uMaxBounces$ / $maxBounces$: Maximum scattering bounces (default 8) [S2, unofficial].
*   $uSteps$ / $steps$: Number of integration steps per photon trace (default varies) [S2, unofficial].
*   $extinction$: UI input for total extinction [S2, unofficial].
*   $ratio$: UI input for majorant ratio [S2, unofficial].
*   $bounces$: UI input for max bounces [S2, unofficial].
*   $steps$: UI input for steps per ray [S2, unofficial].
*   $resolution$: Render buffer resolution (default 512) [S2, unofficial].
*   $workgroup$: Local workgroup size (default 8) [S2, unofficial].

### Edge Cases and Pitfalls
*   **Ray Miss:** If the ray does not intersect the volume cube, the environment map must be sampled to avoid black pixels [S2, S3, unofficial].
*   **Precision:** Logarithmic sampling requires careful handling of random numbers near 0 to avoid infinite step sizes [S2, S3, unofficial].
*   **Convergence:** Monte Carlo methods converge slowly ($1/\sqrt{N}$). Accumulation over multiple frames is required for noise-free images [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \times S)$, where $N$ is the number of pixels and $S$ is the number of steps/bounces per photon/ray [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the accumulator buffer or photon buffers [S2, unofficial].
*   **Performance:** Depends heavily on the density of the volume and the majorant ratio. High density or large majorant ratios lead to more null collisions and slower convergence [S2, unofficial].
*   **Strengths:** Handles anisotropic scattering (Henyey-Greenstein) naturally and avoids artifacts from uniform sampling in heterogeneous volumes [S1, S2, unofficial]. Accurate light transport is achieved through stochastic path tracing [S2, unofficial].
*   **Limitations:** Noisy output requiring temporal accumulation; computationally expensive compared to deterministic ray casting for simple volumes; sensitive to parameter tuning (majorant, steps) [S2, unofficial].
*   **When to use:** When accurate scattering and anisotropy are required, and temporal accumulation is acceptable [S2, unofficial].
*   **When not to use:** For real-time applications without accumulation buffers, or for homogeneous volumes where uniform sampling is sufficient [S1, unofficial].

## Background
The Monte-Carlo approach was introduced as an alternative to uniform sampling, which may result in artifacts since a homogeneous material is assumed [S1, unofficial]. The null-collision technique, also known as the virtual collision method, was introduced to handle heterogeneous media efficiently by sampling distances from a majorant distribution and rejecting non-scattering events [S2, unofficial]. The specific implementation described in the sources is a GPU-based approach using fragment shaders for ray casting and photon tracing, as seen in the `volumetric-ray-casting` and `vpt` repositories [S2, S3, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S3] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
