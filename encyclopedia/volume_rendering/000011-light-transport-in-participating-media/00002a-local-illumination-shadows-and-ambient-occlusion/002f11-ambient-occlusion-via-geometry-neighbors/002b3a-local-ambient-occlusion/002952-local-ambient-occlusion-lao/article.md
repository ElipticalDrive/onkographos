# Local Ambient Occlusion (LAO)

Also called: LAO Renderer, Volumetric Ambient Occlusion

## Overview
Local Ambient Occlusion (LAO) is a technique to estimate local occlusion in volume rendering by sampling the volume along random directions around the light vector, weighted by distance, to simulate ambient occlusion effects [S1, unofficial]. It differs from standard shadow mapping by integrating density along stochastic rays to approximate the blocking of indirect light, providing a softer, more volumetric shadow appearance compared to hard geometric shadows.

## How it works
**Inputs and Outputs:**
*   **Input:** Volume density field $V(\mathbf{x})$, light direction $\mathbf{l}$, light radius $R_{light}$, current position $\mathbf{p}$, and parameters $N_{samples}$, $step\_size$, $weight$.
*   **Output:** A scalar occlusion factor $AO \in [0, 1]$ representing the fraction of ambient light blocked by local volume density.

**Data Structures:**
*   **Accumulator:** A floating-point value (or vector if color is tracked) initialized to zero.
*   **Random Number Generator:** A pseudo-random function returning values in $[0, 1]$.

**Parameters:**
*   $N_{samples}$: Number of LAO samples per pixel. Default: 1 [S1, unofficial].
*   $step\_size$: Step size for LAO ray marching. Default: 0.05 [S1, unofficial].
*   $R_{light}$: Radius of the light source used for LAO sampling. Default: 0.19 [S1, unofficial].
*   $weight$: Weight applied to the LAO contribution when mixing into the final color. Default: 0.69 [S1, unofficial].
*   $coef$: Coefficient used to normalize LAO accumulation. Default: 1.0 [S1, unofficial].
*   $enable$: Boolean flag to enable/disable LAO calculation. Default: true [S1, unofficial].

**Algorithm Steps:**
1.  Initialize `accumuLAOContribution` to 0.
2.  Loop $i$ from 0 to $N_{samples} - 1$:
    a.  Initialize `inner_accum` to 0.
    b.  Loop $t$ from 0.001 to 1.0 with increment $step\_size$:
        i.  Generate a random direction $\mathbf{d}_{rand}$:
            $$ \mathbf{d}_{rand} = \text{normalize}((-1.0 + 2.0 \cdot \mathbf{u}_1) \cdot u_2) $$
            where $\mathbf{u}_1$ is a 3D vector of random numbers in $[0,1]$ and $u_2$ is a scalar random number in $[0,1]$.
        ii. Compute the LAO half vector $\mathbf{h}$:
            $$ \mathbf{h} = \text{normalize}(\mathbf{l} + \mathbf{d}_{rand} \cdot \text{mix}(0.0, R_{light}, t) - \mathbf{p}) $$
            Note: The term $\text{mix}(0.0, R_{light}, t)$ interpolates between 0 and $R_{light}$ based on $t$.
        iii. Sample the volume at $\mathbf{p} + \mathbf{h} \cdot t$. Let this sample be $S$.
        iv. Add to `inner_accum` weighted by $(1.0 - t)^2$:
            $$ \text{inner\_accum} \leftarrow \text{inner\_accum} + S \cdot (1.0 - t)^2 $$
    c.  Divide `inner_accum` by $coef$.
    d.  Clamp the result to $[0, 1]$ and add to `accumuLAOContribution`.
3.  Divide `accumuLAOContribution` by $N_{samples}$ to get the final $AO$ factor.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_lao(position, light_direction, volume_sampler, params):
    """
    Computes Local Ambient Occlusion for a given position.
    
    Args:
        position: vec3, current shading position.
        light_direction: vec3, normalized direction to the light source.
        volume_sampler: function, samples volume density at a given position.
        params: dict containing uNumLAOSamples, uLAOStepSize, uLightRadious, 
                uLightCoeficient, uLAOWeight.
                
    Returns:
        float: The LAO contribution factor.
    """
    n_samples = params.get('uNumLAOSamples', 1)
    step_size = params.get('uLAOStepSize', 0.05)
    light_radius = params.get('uLightRadious', 0.19)
    coef = params.get('uLightCoeficient', 1.0)
    
    accumu_lao_contribution = 0.0
    
    for _ in range(n_samples):
        inner_accum = 0.0
        t = 0.001
        while t <= 1.0:
            # Generate random direction
            # rand_vec returns vec3 of random numbers in [0, 1]
            # rand_scalar returns a scalar random number in [0, 1]
            rand_vec = rand_vec3()
            rand_scalar = rand_scalar()
            
            # randomDirection = normalize(-1.0 + 2.0 * vec3(rand(...))) * rand(...)
            # Note: The source code implies the scalar rand() multiplies the normalized vector
            # or is part of the direction generation. Based on "normalize(-1.0 + 2.0 * vec3(rand(...))) * rand(...)"
            # it seems the scalar is a magnitude factor.
            raw_dir = -1.0 + 2.0 * rand_vec
            normalized_dir = normalize(raw_dir)
            random_direction = normalized_dir * rand_scalar
            
            # laoHalfVector = normalize(vLight + randomDirection * mix(0.0, uLightRadious, t) - position)
            # Note: The source code subtracts 'position' inside the normalize, which is unusual for a direction vector 
            # unless 'position' is treated as an offset or the light is at origin. 
            # Assuming 'vLight' is the light direction vector and 'position' is the current point.
            # The formula in source: normalize(vLight + randomDirection * mix(0.0, uLightRadious, t) - position)
            # This implies the vector being normalized is (LightDir + Perturbation - Position).
            # This is likely a specific implementation detail where the light is considered to be at the origin 
            # or 'position' is relative. We follow the source exactly.
            
            mix_val = mix(0.0, light_radius, t)
            lao_half_vector = normalize(light_direction + random_direction * mix_val - position)
            
            # Sample volume at position + laoHalfVector * t
            lao_sample = volume_sampler(position + lao_half_vector * t)
            
            # Add to accumulator weighted by (1.0 - t)^2
            weight = pow(1.0 - t, 2.0)
            inner_accum += lao_sample * weight
            
            t += step_size
            
        # Divide by uLightCoeficient, clamp to [0,1], and add to LAOContribution
        lao_val = inner_accum / coef
        lao_val = clamp(lao_val, 0.0, 1.0)
        accumu_lao_contribution += lao_val
        
    # After outer loop, divide LAOContribution by uNumLAOSamples
    final_lao = accumu_lao_contribution / n_samples
    
    return final_lao
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of `mix(0.0, uLightRadious, t)` and subtraction of `position` inside the normalization vector can lead to numerical instability if `position` is large. [G] Ensure coordinates are normalized or relative to the light source if necessary.
*   **Boundaries:** Sampling outside the volume bounds should return 0 density.
*   **Degenerate Input:** If $N_{samples} = 0$, the result is undefined. Default to 1.0 (no occlusion).

## Operating characteristics
*   **Time Complexity:** $O(N_{samples} \cdot \frac{1.0}{step\_size})$. With default values ($N_{samples}=1, step\_size=0.05$), this is approximately 20 volume samples per pixel.
*   **Memory Complexity:** $O(1)$ auxiliary memory per pixel.
*   **Strengths:** Provides soft, volumetric shadows that react to local density variations. Simple to implement in shaders.
*   **Limitations:** Stochastic noise due to random sampling. Performance scales linearly with sample count. The specific vector arithmetic in the half-vector calculation is non-standard and may not generalize to all lighting setups.
*   **When to use:** When soft, ambient occlusion-like shadows are desired in volumetric scenes and performance allows for multiple volume samples per pixel.
*   **When not to use:** For hard shadows or when performance is critical.

## Background
Local Ambient Occlusion in this specific context originates from the `terier/vpt` project, an unofficial volume rendering toolkit [S1, unofficial]. It is designed to enhance the visual quality of volume rendering by approximating the occlusion of ambient light by the volume itself.

## Sources
[S1] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
