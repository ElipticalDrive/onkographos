# Shadow Volume Rendering

Also called: Volumetric Shadow Maps

## Overview
Shadow Volume Rendering is a specialized ray marching routine that computes only the accumulated alpha (transmittance) along a ray to generate shadow maps or occlusion values for volume scenes [S1]. It precomputes a shadow volume to store the amount of light arriving at each sample after attenuation by intervening material, allowing the renderer to multiply interpolated values from this map by colors from the transfer function during the main pass [S2]. Unlike [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md), which uses an opacity shadow map for occlusion, this method explicitly constructs a volumetric shadow buffer [S2].

## How it works
**Inputs:** Ray origin and direction in world space, world-to-object transformation matrix, `tMin`, `tMax`, `samplingScale`, `self.step`, and `nearly_one` threshold [S1].
**Outputs:** Accumulated alpha (transmittance) value for the ray [S1].

**Data Structures:**
*   `ShadowPayload`: Contains `rng` (pointer to `RandomTEA`) and `alpha` (float) [S1]. Packaged via `packPointer(&payload, u0, u1)` [S1].

**Numbered Steps:**
1. Transform the ray origin and direction from world space to object space using the provided transformation matrix [S1].
2. Initialize a `ShadowPayload` with `alpha = 0.0` and a random number generator [S1].
3. March along the ray from `tMin` to `tMax` using a step size adjusted by `samplingScale` [S1].
4. For each step, compute the sample position $p$ using jittered interpolation between the current and next step boundaries [S1].
5. Sample the volume density/alpha at $p$ and accumulate it into `payload.alpha` using front-to-back blending [S1].
6. Continue marching while `payload.alpha` remains below `nearly_one`; terminate otherwise [S1].
7. Return the final `payload.alpha` for use in the main rendering pass [S1].

**Formulas:**
*   Sample position: $p = \text{org} + \text{lerp}(\text{jitter}, t.x, t.y) \times \text{dir}$ [S1]
*   Alpha accumulation: $\text{payload.alpha} \leftarrow \text{payload.alpha} + (1.0 - \text{payload.alpha}) \times \text{sampleAlpha}$ [S1]
*   Termination condition: $\text{payload.alpha} < \text{nearly\_one}$ [S1]
Where `org` and `dir` are the transformed ray origin and direction, `jitter` is a random float, $t.x$ and $t.y$ are the step boundaries, and `sampleAlpha` is the sampled volume density [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shadow_volume_rendering(ray_origin, ray_direction, wto, t_min, t_max, sampling_scale, step_size, nearly_one):
    # Step 1: Transform ray to object space [S1]
    org = xfm_point(wto, ray_origin)
    dir = xfm_vector(wto, ray_direction)

    # Step 2: Initialize payload [S1]
    payload = {
        'rng': RandomTEA(),  # [G] standard RNG initialization
        'alpha': 0.0
    }

    # Step 3: Ray marching loop [S1]
    t = t_min
    while t < t_max:
        t_next = t + sampling_scale * step_size
        
        # Step 4: Compute sample position with jitter [S1]
        jitter = payload['rng'].next_float()  # [G] standard RNG call
        p = org + lerp(jitter, t, t_next) * dir

        # Step 5: Sample volume density/alpha at p [G]
        sample_alpha = sample_volume_density(p)

        # Step 5: Accumulate alpha using front-to-back blending [S1]
        payload['alpha'] += (1.0 - payload['alpha']) * sample_alpha

        # Step 6: Termination check [S1]
        if payload['alpha'] >= nearly_one:
            break

        t = t_next

    return payload['alpha']
```

**Parameters:**
*   `samplingScale`: Multiplier applied to `self.step` to adjust shadow ray sampling density [S1]. Typical values are not specified in the sources.

**Edge Cases and Pitfalls:**
*   **Blurry Shadows & Dark Images:** The method can result in blurry shadows and dark images due to interpolation and shadow mapping limitations [S2].
*   **Memory Usage:** The precomputed shadow volume requires extra memory [S2].
*   **Precision:** The sources do not specify precision issues, but the use of `float` for alpha accumulation suggests potential precision limits in high-opacity regions [S1].

## Subtopics

### Transmittance Function Mapping
*   **Definition:** An approximation method that uses orthogonal basis functions (e.g., Fourier series) to calculate self-shadowing more efficiently than ray marching [S3].
*   **How it works:**
    1. Generate a volume shadow map from the sun direction using orthographic projection [S3].
    2. Store the first 9 Fourier coefficients in a shadow map with format 512x512x3 R11G11B10 [S3].
    3. During rendering, read the Fourier coefficients and combine them linearly with basis functions to reconstruct optical depth and transmittance [S3].
*   **Comparison:** TFM efficiency is roughly equivalent to Ray Marching with 1-2 steps, but provides more accurate results than 2-step RM. It may produce banding artifacts [S3].

## Operating characteristics
*   **Time Complexity:** The basic shadow volume rendering involves a ray marching loop. The number of steps depends on the `samplingScale` and volume density. TFM is roughly equivalent to 1-2 steps of ray marching in efficiency [S3].
*   **Memory Complexity:** Requires storage for the volumetric shadow map. The TFM variant uses a 512x512x3 R11G11B10 texture [S3].
*   **Performance:** Not given in the sources for the basic method. TFM is noted as efficient [S3].
*   **Strengths:** Precomputes shadow information to store light attenuation, which is interpolated and multiplied by transfer function colors during rendering [S2]. TFM offers a balance of efficiency and accuracy [S3].
*   **Limitations:** Can produce blurry shadows and dark images [S2]. TFM may produce banding artifacts [S3]. Requires extra memory for the shadow volume [S2].
*   **When to use:** When self-shadowing is required for volumetric scenes and performance is a concern, but high accuracy is still desired (TFM) [S3].
*   **When not to use:** When memory is constrained, as the method uses extra memory for the shadow volume [S2].

## Background
The concept of precomputing a shadow volume for volumetric materials was introduced to address the unrealistic assumption that light arrives at a sample without interacting with the rest of the volume [S2]. This approach was proposed as an alternative to surface-based lighting models [S2]. The Transmittance Function Mapping variant has been used in games like Final Fantasy, utilizing Fourier series for approximation [S3].

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S2] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S3] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
