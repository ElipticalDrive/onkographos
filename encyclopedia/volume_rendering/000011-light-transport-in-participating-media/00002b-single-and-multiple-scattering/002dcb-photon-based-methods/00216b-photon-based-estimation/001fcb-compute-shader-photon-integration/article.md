# Compute Shader Photon Integration

Also called: MCM Renderer, MCCRenderer

## Overview
Compute Shader Photon Integration is a GPU-accelerated Monte Carlo photon integration method that simulates light transport through participating media by processing photons in parallel using global invocation IDs [S1, S2, S3, unofficial]. It differs from CPU-based photon mapping by leveraging compute shaders to perform random walk steps, handling absorption, scattering, and null collisions via Russian Roulette-like probability checks, while accumulating radiance in a feedback buffer [S2, unofficial]. Unlike traditional photon mapping which stores photons in a spatial data structure for later density estimation, this method directly updates radiance buffers per-frame, making it suitable for dynamic scenes or real-time feedback [S1, S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Photon Buffer:** A storage buffer (`std430`) containing photon state structures. Each photon has a position, direction, transmittance, radiance, bounce count, and sample count [S1, S2, S3, unofficial].
*   **Volume Texture:** A 3D texture storing volume density/color information. The `sampleVolumeColor` function returns a `vec4` where `rg` comes from the volume texture, `rgb` from the transfer function, and `a` (alpha) from the transfer function alpha [S2, S3, unofficial].
*   **Environment Map:** A 2D texture for sampling background radiance when photons exit the volume [S2, S3, unofficial].
*   **Uniforms:**
    *   `uMajorant`: Global majorant extinction coefficient [S3, unofficial].
    *   `uAbsorptionCoefficient`: Base absorption multiplier [S3, unofficial].
    *   `uScatteringCoefficient`: Base scattering multiplier [S3, unofficial].
    *   `uAnisotropy` / `uScatteringBias`: Henyey-Greenstein asymmetry parameter $g$ [S2, S3, unofficial].
    *   `uMaxBounces`: Maximum scattering events allowed [S2, S3, unofficial].
    *   `uSteps`: Fixed number of null-collision steps per photon per frame [S3, unofficial].
    *   `uColorBias`, `uAlphaBias`, `uAlphaTransfer`: Blending factors for transfer function application [S3, unofficial].
    *   `uMvpInverseMatrix`, `uInverseResolution`, `uBlur`: Parameters for ray unprojection and jittering [S2, unofficial].

**Outputs:**
*   **Radiance Image:** A `rgba32f` write-only image buffer (`image2D`) storing the accumulated radiance [S1, unofficial].
*   **Photon State Buffers (G-buffers):** Four 2D textures storing photon state for the next frame or feedback:
    *   `oPosition` (`vec4`): `xyz` = position, `w` = unused (or 0) [S2, S3, unofficial].
    *   `oDirection` (`vec4`): `xyz` = direction, `w` = bounce count [S2, S3, unofficial].
    *   `oTransmittance` (`vec4`): `rgb` = transmittance, `w` = unused (or 0) [S2, S3, unofficial].
    *   `oRadiance` (`vec4`): `rgb` = accumulated radiance, `w` = sample count [S2, S3, unofficial].

### Data Structures
The photon state is typically stored in a flat buffer or structured buffer accessible via `std430` layout [S1, unofficial]. The structure `Photon` contains:
*   `position`: `vec3`
*   `direction`: `vec3`
*   `transmittance`: `vec3`
*   `radiance`: `vec3`
*   `bounces`: `uint`
*   `samples`: `uint`

### Steps
1.  **Dispatch and Indexing:** The compute shader is dispatched with a grid size matching the number of photons. Each work item calculates its global index:
    $$ \text{globalIndex} = \text{gl\_GlobalInvocationID.x} + \text{gl\_GlobalInvocationID.y} \times \text{gl\_WorkGroupSize.x} $$
    [S1, unofficial]. It loads the photon state from `sPhotons[globalIndex]` [S1, unofficial].

2.  **Photon Initialization (Reset):**
    *   Generate a random ray origin and direction using `unprojectRand(state, vPosition, uMvpInverseMatrix, uInverseResolution, uBlur, from, to)` [S2, S3, unofficial].
    *   Set `photon.direction = normalize(to - from)` [S2, S3, unofficial].
    *   Intersect the ray with the volume cube (unit cube $[0,1]^3$) using `intersectCube(from, photon.direction)` to get `tbounds` [S2, S3, unofficial].
    *   Set `photon.position = from + tbounds.x * photon.direction` [S2, S3, unofficial].
    *   Reset state: `photon.transmittance = vec3(1)`, `photon.bounces = 0`, `photon.radiance = vec3(0)`, `photon.samples = 0` [S2, S3, unofficial].

3.  **Random Walk Loop:** Iterate `i` from 0 to `uSteps - 1` [S2, S3, unofficial].
    *   **Step Sampling:** Sample a random number $r_x$ and compute step size:
        $$ t = -\frac{\ln(r_x)}{\sigma_{\text{majorant}}} $$
        where $\sigma_{\text{majorant}}$ is `uMajorant` [S3, unofficial].
    *   **Advance Position:** `photon.position += t * photon.direction` [S3, unofficial].
    *   **Boundary Check:** If `photon.position` is out of bounds (any component $< 0$ or $> 1$) [S3, unofficial]:
        *   Sample environment map: `envSample = sampleEnvironmentMap(photon.direction)` [S2, S3, unofficial].
        *   Compute radiance contribution: `radiance = photon.transmittance * envSample.rgb` [S3, unofficial].
        *   Update radiance incrementally:
            $$ L_{\text{new}} = L_{\text{old}} + \frac{L_{\text{sample}} - L_{\text{old}}}{N} $$
            where $N$ is `photon.samples` [S3, unofficial].
        *   Increment `photon.samples` and reset photon state (return to Step 2) [S2, S3, unofficial].
    *   **Volume Sampling:** Sample volume color at `photon.position` using `sampleVolumeColor` [S2, S3, unofficial]. Let the result be `volumeSample` (`vec4`).
    *   **Coefficient Calculation:**
        *   Compute absorption coefficient: $\mu_{\text{absorption}} = \text{volumeSample.a} \times \text{uAbsorptionCoefficient}$ [S3, unofficial].
        *   Compute scattering coefficient: $\mu_{\text{scattering}} = \text{volumeSample.a} \times \text{uScatteringCoefficient}$ [S3, unofficial].
        *   Compute null coefficient: $\mu_{\text{null}} = \sigma_{\text{majorant}} - \mu_{\text{absorption}} - \mu_{\text{scattering}}$ [S3, unofficial].
        *   Adjust majorant: $\sigma_{\text{majorant}}' = \mu_{\text{absorption}} + \mu_{\text{scattering}} + |\mu_{\text{null}}|$ [S3, unofficial].
        *   Compute probabilities:
            $$ P_{\text{null}} = \frac{|\mu_{\text{null}}|}{\sigma_{\text{majorant}}'} $$
            $$ P_{\text{absorption}} = \frac{\mu_{\text{absorption}}}{\sigma_{\text{majorant}}'} $$
            $$ P_{\text{scattering}} = \frac{\mu_{\text{scattering}}}{\sigma_{\text{majorant}}'} $$
            [S3, unofficial].
        *   *Note:* Some implementations use a simplified probability model based on volume alpha and max RGB: $P_{\text{scattering}} = \text{volumeSample.a} \times \max(\text{volumeSample.rgb})$ if `bounces < uMaxBounces`, else 0; $P_{\text{absorption}} = 1 - P_{\text{null}} - P_{\text{scattering}}$ [S2, unofficial]. The majorant-based approach [S3, unofficial] is more rigorous for null-collision Monte Carlo.
    *   **Interaction Selection:** Sample a random number $r_y$ [S3, unofficial].
        *   If $r_y < P_{\text{absorption}}$:
            *   **Absorption:** Update transmittance:
                $$ \text{photon.transmittance} \times= 1.0 - \frac{\mu_{\text{absorption}}}{\sigma_{\text{majorant}}' \times P_{\text{absorption}}} $$
                [S3, unofficial]. (Alternatively, set radiance contribution to 0 and reset [S2, unofficial]).
            *   Reset photon state (return to Step 2) [S2, S3, unofficial].
        *   Else if $r_y < P_{\text{absorption}} + P_{\text{scattering}}$:
            *   **Scattering:** Update transmittance:
                $$ \text{photon.transmittance} \times= \text{volumeSample.rgb} \times \frac{\mu_{\text{scattering}}}{\sigma_{\text{majorant}}' \times P_{\text{scattering}}} $$
                [S3, unofficial].
            *   Sample new direction using Henyey-Greenstein phase function with parameter $g$ (`uAnisotropy`) [S2, S3, unofficial].
                *   Compute $\cos\theta = \frac{1 + g^2 - c^2}{2g}$, where $c = \frac{1 - g^2}{1 - g + 2gU}$ and $U$ is a uniform random number [S3, unofficial].
                *   Compute new direction $\mathbf{d}'$ using a random vector $\mathbf{u}$ [S3, unofficial].
            *   Increment `photon.bounces` [S2, S3, unofficial].
        *   Else:
            *   **Null Collision:** Update transmittance:
                $$ \text{photon.transmittance} \times= \frac{\mu_{\text{null}}}{\sigma_{\text{majorant}}' \times P_{\text{null}}} $$
                [S3, unofficial]. Continue loop without changing direction or bounces.

4.  **Output:** Write the final photon state to the output buffers (`oPosition`, `oDirection`, `oTransmittance`, `oRadiance`) [S2, S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Entry Point
def compute_shader(globalInvocationID, gl_WorkGroupSize, sPhotons, oPosition, oDirection, oTransmittance, oRadiance, volumeTexture, envMap, uMajorant, uAbsorptionCoefficient, uScatteringCoefficient, uAnisotropy, uMaxBounces, uSteps, uMvpInverseMatrix, uInverseResolution, uBlur):
    globalIndex = globalInvocationID.x + globalInvocationID.y * gl_WorkGroupSize.x
    
    # Load photon
    photon = sPhotons[globalIndex]
    
    # Reset Photon
    from_pos, to_dir = unprojectRand(state, vPosition, uMvpInverseMatrix, uInverseResolution, uBlur)
    photon.direction = normalize(to_dir - from_pos)
    tbounds = intersectCube(from_pos, photon.direction)
    photon.position = from_pos + tbounds.x * photon.direction
    photon.transmittance = vec3(1.0)
    photon.bounces = 0
    photon.radiance = vec3(0.0)
    photon.samples = 0
    
    for i in range(uSteps):
        # Step Size
        r_x = rand()
        t = -log(r_x) / uMajorant
        photon.position += t * photon.direction
        
        # Boundary Check
        if is_out_of_bounds(photon.position):
            env_sample = sample_environment_map(envMap, photon.direction)
            radiance_contrib = photon.transmittance * env_sample.rgb
            # Incremental Update
            N = float(photon.samples)
            photon.radiance = photon.radiance + (radiance_contrib - photon.radiance) / N
            photon.samples += 1
            # Reset for next photon
            break # Or return to reset logic depending on implementation flow
            
        # Volume Sample
        volume_sample = sample_volume_color(volumeTexture, photon.position)
        
        # Coefficients
        mu_abs = volume_sample.a * uAbsorptionCoefficient
        mu_scat = volume_sample.a * uScatteringCoefficient
        mu_null = uMajorant - mu_abs - mu_scat
        sigma_majorant_prime = mu_abs + mu_scat + abs(mu_null)
        
        # Probabilities
        P_null = abs(mu_null) / sigma_majorant_prime
        P_abs = mu_abs / sigma_majorant_prime
        P_scat = mu_scat / sigma_majorant_prime
        
        # Interaction
        r_y = rand()
        if r_y < P_abs:
            # Absorption
            photon.transmittance *= 1.0 - mu_abs / (sigma_majorant_prime * P_abs)
            photon.radiance = vec3(0.0) # Or incremental update with 0
            photon.samples += 1
            break
        elif r_y < P_abs + P_scat:
            # Scattering
            photon.transmittance *= volume_sample.rgb * mu_scat / (sigma_majorant_prime * P_scat)
            photon.direction = sample_henyey_greenstein(uAnisotropy, photon.direction)
            photon.bounces += 1
        else:
            # Null Collision
            photon.transmittance *= mu_null / (sigma_majorant_prime * P_null)
            
    # Write Output
    oPosition[globalInvocationID.xy] = vec4(photon.position, 0.0)
    oDirection[globalInvocationID.xy] = vec4(photon.direction, float(photon.bounces))
    oTransmittance[globalInvocationID.xy] = vec4(photon.transmittance, 0.0)
    oRadiance[globalInvocationID.xy] = vec4(photon.radiance, float(photon.samples))
```

### Parameters
*   `uMajorant`: Global majorant coefficient. Must be $\ge \sigma_t(\mathbf{p})$ for all $\mathbf{p}$ in the volume [S3, unofficial].
*   `uSteps`: Fixed number of steps per photon. Higher values increase accuracy but cost more [S3, unofficial].
*   `uMaxBounces`: Limits scattering depth. Set to 0 for single-scattering approximation [S2, S3, unofficial].
*   `uAnisotropy`: Henyey-Greenstein $g$ parameter. $g=0$ is isotropic, $g>0$ is forward-scattering [S2, S3, unofficial].

### Pitfalls
*   **Out-of-Bounds:** If `photon.position` is out of bounds, sampling the volume texture may cause undefined behavior or division by zero if not checked. Always check bounds before sampling [S3, unofficial].
*   **Majorant Selection:** If `uMajorant` is too close to the local extinction coefficient, variance increases. If too large, null collisions dominate, reducing efficiency [S3, unofficial].
*   **Precision:** Using `rgba32f` for radiance accumulation is critical to avoid overflow and maintain dynamic range [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** $O(N \times S)$, where $N$ is the number of photons and $S$ is `uSteps` [S2, S3, unofficial]. Each step involves texture sampling and random number generation.
*   **Memory:** $O(N)$ for photon storage plus $O(W \times H)$ for output buffers, where $W, H$ are screen dimensions [S1, S2, unofficial].

### Measured Performance
*   Performance depends heavily on GPU compute throughput and memory bandwidth. Parallel processing of photons allows high throughput on modern GPUs [S1, unofficial].
*   Scaling is linear with the number of photons and steps [S3, unofficial].

### Strengths and Limitations
*   **Strengths:** Fully parallelizable on GPU; suitable for dynamic scenes; no need for complex spatial data structures like BVH or KD-trees for photon storage [S1, S2, unofficial].
*   **Limitations:** High variance with few photons; requires many samples for convergence; null collisions can be inefficient if the majorant is poorly chosen [S3, unofficial].

### When to Use
*   Use for real-time or near-real-time volume rendering where dynamic changes are expected [S1, S2, unofficial].
*   Avoid for static scenes requiring high-quality offline rendering where traditional photon mapping or path tracing might offer better convergence with spatial caching [S2, unofficial].

## Background

### Origin
Compute Shader Photon Integration is an implementation of Monte Carlo photon integration using GPU compute shaders. It is associated with the "MCM Renderer" and "MCCRenderer" projects [S1, S2, S3, unofficial].

### History
The method leverages the parallel processing capabilities of modern GPUs to accelerate photon transport simulation, which was traditionally done on CPUs. The use of null-collision Monte Carlo allows for efficient handling of heterogeneous media [S3, unofficial].

### Usage
Used in WebGL/OpenGL compute shader implementations for volumetric rendering [S1, S2, S3, unofficial].

## Sources

[S1] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S2] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S3] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
