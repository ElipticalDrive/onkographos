# Mr-Voxel Volume Rendering

Also called: VolumeIntegrator::Li, Mr-Voxel Volume Integrator

## Overview
Mr-Voxel Volume Rendering is a direct volume ray casting method implemented within the Mr-Voxel library's `VolumeIntegrator` class. It solves the problem of computing the radiance contribution of a single light source through a single homogeneous or heterogeneous volume region by marching along a ray and accumulating emitted and scattered light [S1, unofficial]. It is distinguished from its siblings by its specific implementation of Russian Roulette for transmittance termination and its reliance on a pre-computed optical depth (`tau`) function for transmittance updates, rather than integrating extinction coefficients directly per step [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A viewing ray `ray`, a scene graph containing a list of volumes (assumed to be a single volume `volumes[0]` for this implementation) and a list of light sources (assumed to be a single light `lights[0]`).
*   **Output:** A `Spectrum` representing the accumulated radiance `Lv` and an update to the transmittance `*T` (passed by reference).

**Data Structures:**
*   `VolumeIntegrator`: Contains the method `Li(Ray ray, Spectrum *T)`.
*   `Scene`: Provides access to `getVolumeCount()`, `getVolumes()`, and `getLightSources()`.
*   `Volume`: Provides methods `tau(Ray, double, int)` for optical depth, `Lve(Point)` for emitted radiance, and `sigma_s(Point)` for scattering coefficient.
*   `Ray`: Defined by origin and direction; supports evaluation `ray(t)` to get position.
*   `Spectrum`: Represents color/radiance values.

**Parameters:**
*   `mStepSize`: The nominal step size for ray marching. Typical value: Not given in the sources.

**Algorithm Steps:**
1.  **Determine Sample Count:** Calculate the number of samples `numSamples` based on the ray segment length `(t1 - t0)` and `mStepSize`.
    $$ \text{numSamples} = \left\lfloor \frac{t_1 - t_0}{\text{mStepSize}} \right\rfloor $$
    If `numSamples` is 0, set it to 1 [S1, unofficial].
2.  **Calculate Actual Step:** Compute the precise step length `step` to ensure the ray covers the exact interval `[t0, t1]`.
    $$ \text{step} = \frac{t_1 - t_0}{\text{numSamples}} $$
3.  **Random Offset:** Apply a random offset to the starting position `t0` to reduce aliasing.
    $$ t_0 \leftarrow t_0 + \left( \frac{\text{rand()}}{\text{RAND\_MAX}} \right) \times \text{step} $$
4.  **Initialize Accumulators:** Set accumulated radiance `Lv = 0` and transmittance `Tr = 1` [S1, unofficial].
5.  **Ray Marching Loop:** Iterate `i` from 0 to `numSamples - 1`:
    a.  **Update Positions:** Store previous position `pPrev` and compute current position `p = ray(t0)`.
    b.  **Create Step Ray:** Define a short ray `tRay` from `pPrev` to `p` with length `step` [S1, unofficial].
    c.  **Compute Optical Depth:** Calculate the optical depth `stepTau` for the step using the volume's `tau` function. The implementation uses a fixed evaluation point of `0.5 * step` and a flag `0`.
        $$ \text{stepTau} = \text{volumes}[0] \to \text{tau}(tRay, 0.5 \times \text{step}, 0) $$
    d.  **Update Transmittance:** Multiply the current transmittance by the exponential of the negative optical depth.
        $$ T_r \leftarrow T_r \times e^{-\text{stepTau}} $$
    e.  **Accumulate Emission:** Add the emitted radiance at point `p` to `Lv`.
        $$ L_v \leftarrow L_v + \text{volumes}[0] \to L_{ve}(p) $$
    f.  **Check Scattering:** Get the scattering coefficient `ss = volumes[0]->sigma_s(p)`. If `ss` is effectively black (zero), skip lighting calculations for this step [S1, unofficial].
    g.  **Lighting Contribution:**
        i.  Compute light direction `wL` from `p` to the light source position, normalized.
        ii. Create a shadow ray `shadRay` from `p` in direction `wL`.
        iii. Check for occlusion. The light is unshadowed if the shadow ray does not intersect a surface closer than the light distance.
        $$ \text{unshadowed} = \neg (\text{mScene} \to \text{intersect}(shadRay, \&surface)) \lor (surface.t > \|p - wL\|) $$
        iv. If unshadowed, compute light transmittance `lTr` through the volume. The implementation uses a fixed step size of `mStepSize * 2` and flag `0.0`.
        $$ lTr = e^{-\text{volumes}[0] \to \text{tau}(shadRay, \text{mStepSize} \times 2, 0.0)} $$
        v.  Compute direct lighting `Ld` and add to `Lv`.
        $$ L_d = lTr \times \text{lights}[0] \to \text{color} $$
        $$ L_v \leftarrow L_v + L_d \times ss \times T_r $$
    h.  **Russian Roulette:** If `Tr < 1e-3`, perform Russian Roulette. With probability 0.5, break the loop. Otherwise, continue and scale `Tr` by 2 (i.e., `Tr = Tr / 0.5`) [S1, unofficial].
    i.  **Advance Ray:** Increment `t0` by `step` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def Li(ray, T):
    # Access scene data
    volumes = ray.scene.getVolumes()
    lights = ray.scene.getLightSources()
    
    # Assume single volume and single light as per implementation notes
    volume = volumes[0]
    light = lights[0]
    
    # 1. Determine sample count
    t0 = ray.origin
    t1 = ray.end # Or intersection distance
    num_samples = int((t1 - t0) / ray.scene.mStepSize)
    if num_samples == 0:
        num_samples = 1
        
    # 2. Calculate actual step
    step = (t1 - t0) / num_samples
    
    # 3. Random offset
    t0 += (random() / RAND_MAX) * step
    
    # 4. Initialize
    Lv = Spectrum(0, 0, 0)
    Tr = 1.0
    
    # 5. Loop
    for i in range(num_samples):
        p_prev = ray(t0)
        t0 += step
        p = ray(t0)
        
        # Create short ray for tau calculation
        tRay = Ray(p_prev, p - p_prev)
        
        # Compute optical depth
        step_tau = volume.tau(tRay, 0.5 * step, 0)
        
        # Update transmittance
        Tr *= exp(-step_tau)
        
        # Accumulate emission
        Lv += volume.Lve(p)
        
        # Check scattering
        ss = volume.sigma_s(p)
        if ss != Spectrum(0, 0, 0):
            # Light direction
            wL = normalize(light.position - p)
            
            # Shadow ray
            shadRay = Ray(p, wL)
            surface = Surface()
            intersects = ray.scene.intersect(shadRay, surface)
            light_dist = length(light.position - p)
            
            # Check occlusion
            unshadowed = (not intersects) or (surface.t > light_dist)
            
            if unshadowed:
                # Light transmittance
                lTr = exp(-volume.tau(shadRay, ray.scene.mStepSize * 2, 0.0))
                
                # Direct lighting
                Ld = lTr * light.color
                Lv += Ld * ss * Tr
        
        # Russian Roulette
        if Tr < 1e-3:
            if random() < 0.5:
                break
            else:
                Tr /= 0.5
                
    T.value = Tr
    return Lv
```

**Edge Cases and Pitfalls:**
*   **Zero Samples:** If the ray segment is shorter than `mStepSize`, `numSamples` becomes 0. The code explicitly sets it to 1 to prevent division by zero or empty loops [S1, unofficial].
*   **Single Volume/Light Assumption:** The implementation hardcodes `volumes[0]` and `lights[0]`. It does not support multiple volumes or lights in the current form, requiring future aggregators [S1, unofficial].
*   **Precision:** The use of `float` for random number generation and intermediate calculations may introduce precision artifacts in transmittance accumulation.
*   **Tau Function Dependency:** The correctness of the transmittance update relies entirely on the `Volume::tau` method correctly integrating extinction along the segment. The fixed evaluation points (`0.5 * step`, `mStepSize * 2`) suggest approximations that may vary in accuracy depending on volume heterogeneity.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples `numSamples`, which is proportional to the ray length divided by `mStepSize`. $O(L / \text{mStepSize})$.
*   **Memory Complexity:** Constant $O(1)$ auxiliary memory, as it accumulates radiance and transmittance in scalar variables.
*   **Strengths:** Simple implementation; uses Russian Roulette to handle deep transmittance efficiently; decouples volume data from rendering logic via the `Volume` interface.
*   **Limitations:** Limited to single volume and single light scenarios in this implementation; relies on pre-computed or approximated `tau` values which may be expensive or inaccurate if not cached; no support for multiple scattering (single scattering only, implied by the direct lighting calculation).
*   **When to Use:** Suitable for real-time or interactive rendering where single-scattering approximation is acceptable and scene complexity is limited to single volume regions.
*   **When Not to Use:** Scenes with multiple overlapping volumes, complex lighting setups, or high-fidelity requirements needing multiple scattering.

## Background
Mr-Voxel is a volume rendering library and accelerated voxel data structure developed by C. Gibson as part of his graphics graduate thesis [S1, unofficial]. The volume rendering components, including the `VolumeIntegrator::Li` method, are derived or loosely based on the volume rendering sections of "Physically Based Rendering, 2nd edition" by Matt Pharr [S1, unofficial]. The project aims to integrate importance-sampled light caches for polygonal, implicit, and volume objects.

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
