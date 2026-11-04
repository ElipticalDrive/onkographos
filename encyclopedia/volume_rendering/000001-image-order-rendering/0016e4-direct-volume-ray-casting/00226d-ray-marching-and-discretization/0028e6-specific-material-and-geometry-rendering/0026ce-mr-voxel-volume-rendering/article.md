# Mr-Voxel Volume Rendering

Also called: VolumeIntegrator::Li

## Overview
Mr-Voxel Volume Rendering is a direct volume ray casting technique implemented in the `VolumeIntegrator::Li` function that performs ray marching through a single volume region, accumulating emitted light and computing transmittance and scattering contributions from a single light source [S1, unofficial]. It is distinguished from its siblings by its specific implementation within the Mr-Voxel library's integrator, which combines an accelerated voxel data structure with a ray marching loop that handles single-volume, single-light scenarios with Russian Roulette termination [S1, unofficial]. Unlike [Procedural Volumetric Cloud Rendering](3480-procedural-volumetric-cloud-rendering.md) which often relies on procedural noise for density, or [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md) which may use simplified scattering models, this method explicitly computes optical depth and direct lighting transmittance per step for a defined volume object [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray `ray` (origin, direction, $t_0$, $t_1$), a scene object `mScene` providing access to volumes and lights, and a transmittance pointer `T`.
*   **Output:** A `Spectrum` representing the accumulated radiance $L_v$, and the updated transmittance `*T`.

**Data Structures:**
*   `VolumeIntegrator`: Contains member `mStepSize` (float/double) defining the base step size for ray marching [S1, unofficial].
*   `Scene`: Provides `getVolumeCount()`, `getVolumes()` (array of volume objects), and `getLightSources()` (array of light objects) [S1, unofficial].
*   `Volume`: Provides methods `tau(ray, length, ...)` for optical depth, `Lve(p)` for emitted radiance, and `sigma_s(p)` for scattering coefficient [S1, unofficial].
*   `Ray`: Has method `intersect(shadRay, &surface)` to check for occlusions [S1, unofficial].

**Parameters:**
*   `mStepSize`: The base step size for ray marching. Typical value not specified in sources, but used to calculate `numSamples` [S1, unofficial].
*   Russian Roulette probability: Fixed at 0.5 [S1, unofficial].
*   Transmittance threshold: $10^{-3}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Calculate Samples:** Compute the number of steps `numSamples` based on the ray segment length $(t_1 - t_0)$ and `mStepSize`. If `numSamples` is 0, set it to 1 [S1, unofficial].
    $$ \text{numSamples} = \max\left(1, \left\lfloor \frac{t_1 - t_0}{\text{mStepSize}} \right\rfloor \right) $$
2.  **Calculate Step Length:** Compute the actual step length `step` to ensure the ray covers exactly from $t_0$ to $t_1$ [S1, unofficial].
    $$ \text{step} = \frac{t_1 - t_0}{\text{numSamples}} $$
3.  **Random Offset:** Apply a random offset to the starting position $t_0$ to reduce banding artifacts [S1, unofficial].
    $$ t_0 \leftarrow t_0 + \text{step} \times \frac{\text{rand()}}{\text{RAND\_MAX}} $$
4.  **Initialize Accumulators:** Set accumulated radiance $L_v = 0$ and transmittance $T_r = 1$ [S1, unofficial].
5.  **Ray Marching Loop:** Iterate $i$ from 0 to `numSamples`:
    a.  **Update Positions:** Let $p_{prev}$ be the current position. Update current position $p = \text{ray}(t_0 + i \times \text{step})$. Create a short ray `tRay` from $p_{prev}$ to $p$ with length `step` [S1, unofficial].
    b.  **Optical Depth:** Compute the optical depth (tau) for the step. The source specifies calling `volumes[0]->tau(tRay, 0.5 * step, 0)` [S1, unofficial].
    c.  **Update Transmittance:** Multiply current transmittance by the exponential of the negative optical depth [S1, unofficial].
        $$ T_r \leftarrow T_r \times e^{-\tau} $$
    d.  **Accumulate Emission:** Add the emitted radiance at point $p$ to $L_v$ [S1, unofficial].
        $$ L_v \leftarrow L_v + L_{ve}(p) $$
    e.  **Scattering Check:** Get the scattering coefficient $\sigma_s = \text{volumes}[0].\sigma_s(p)$. If $\sigma_s$ is not black (zero), proceed to compute lighting [S1, unofficial].
    f.  **Light Direction:** Compute the normalized direction $\omega_L$ from point $p$ to the light source position [S1, unofficial].
    g.  **Shadow Ray:** Create a shadow ray `shadRay` from $p$ in direction $\omega_L$. Check if it intersects a surface closer than the light distance. The condition for being unshadowed is: no intersection, or intersection distance $t_{surf}$ is greater than the distance to the light [S1, unofficial].
        $$ \text{unshadowed} = (\neg \text{mScene}.\text{intersect}(\text{shadRay}, \text{surface})) \lor (\text{surface}.t > \|p - \omega_L\|) $$
    h.  **Light Transmittance:** If unshadowed, compute the light transmittance $l_T$ through the volume. The source specifies calling `volumes[0]->tau(shadRay, mStepSize * 2, 0.0)` [S1, unofficial].
        $$ l_T = e^{-\tau_{light}} $$
    i.  **Direct Lighting:** Compute the direct lighting contribution $L_d$ using the light's color and the light transmittance. Add to accumulated radiance, scaled by scattering coefficient and current transmittance [S1, unofficial].
        $$ L_d = l_T \times \text{lights}[0].\text{color} $$
        $$ L_v \leftarrow L_v + L_d \times \sigma_s \times T_r $$
    j.  **Russian Roulette:** If $T_r < 10^{-3}$, perform Russian Roulette. With probability 0.5, break the loop. Otherwise, continue and scale $T_r$ by $1/0.5$ (i.e., $T_r \leftarrow T_r / 0.5$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def VolumeIntegrator_Li(ray, scene, T_out):
    # Access scene data
    volumes = scene.getVolumes()
    lights = scene.getLightSources()
    mStepSize = scene.getIntegrator().mStepSize
    
    # Assume single volume and single light as per implementation notes
    volume = volumes[0]
    light = lights[0]
    
    # 1. Calculate number of samples
    t0 = ray.t0
    t1 = ray.t1
    numSamples = int((t1 - t0) / mStepSize)
    if numSamples == 0:
        numSamples = 1
        
    # 2. Calculate actual step length
    step = (t1 - t0) / numSamples
    
    # 3. Apply random offset to starting position
    random_val = rand() / RAND_MAX
    t0 += random_val * step
    
    # 4. Initialize accumulators
    Lv = Spectrum(0, 0, 0)
    Tr = 1.0
    
    # 5. Ray marching loop
    for i in range(numSamples):
        # Update positions
        p_prev = ray(t0 + (i - 1) * step) if i > 0 else ray(t0) # Note: logic in source implies p_prev updates to current p, then p updates. 
        # Source says: "update pPrev to current p, update p to ray(t0)". 
        # This phrasing is ambiguous. Standard ray marching: p_prev is start of step, p is end.
        # Let's interpret: p_prev is position at start of step i. p is position at end of step i.
        # The source says: "update pPrev to current p, update p to ray(t0)". 
        # This likely means: p_prev = p; p = ray(t0 + i*step). 
        # But t0 was modified by offset. Let's assume t_current = t0 + i*step.
        
        t_current = t0 + i * step
        p_prev = ray(t_current - step) if i > 0 else ray(t0) # Approximation for first step
        p = ray(t_current)
        
        # Create short ray for optical depth calculation
        # Source: tRay from pPrev to p with length step
        tRay = Ray(p_prev, p - p_prev)
        
        # Compute optical depth (tau)
        # Source: volumes[0]->tau(tRay, 0.5 * step, 0)
        tau = volume.tau(tRay, 0.5 * step, 0)
        
        # Update transmittance
        Tr = Tr * exp(-tau)
        
        # Accumulate emitted light
        Lve = volume.Lve(p)
        Lv = Lv + Lve
        
        # Get scattering coefficient
        ss = volume.sigma_s(p)
        
        # Check if scattering is non-zero
        if not ss.is_black():
            # Compute light direction
            wL = normalize(light.position - p)
            
            # Create shadow ray
            shadRay = Ray(p, wL)
            
            # Check intersection
            surface = Surface()
            intersected = scene.intersect(shadRay, surface)
            
            # Distance to light
            dist_to_light = length(light.position - p)
            
            # Check if unshadowed
            # Source: !mScene->intersect(shadRay, &surface) || (surface.t > (p-wL).length())
            # Note: (p-wL).length() is effectively dist_to_light if wL is normalized and points to light
            unshadowed = (not intersected) or (surface.t > dist_to_light)
            
            if unshadowed:
                # Compute light transmittance
                # Source: volumes[0]->tau(shadRay, mStepSize * 2, 0.0)
                tau_light = volume.tau(shadRay, mStepSize * 2, 0.0)
                lTr = exp(-tau_light)
                
                # Compute direct lighting contribution
                Ld = lTr * light.color
                Lv = Lv + Ld * ss * Tr
        
        # Russian Roulette
        if Tr < 1e-3:
            if rand() < 0.5:
                break
            else:
                Tr = Tr / 0.5
                
    T_out = Tr
    return Lv
```

**Edge Cases and Pitfalls:**
*   **Zero Samples:** If the ray segment is shorter than `mStepSize`, `numSamples` is forced to 1 to prevent division by zero or empty loops [S1, unofficial].
*   **Single Volume/Light Assumption:** The implementation explicitly accesses `volumes[0]` and `lights[0]`. It does not support multiple volumes or lights in the current form, with a TODO noting that aggregators are needed for multiple volumes [S1, unofficial].
*   **Precision:** Transmittance can drop below $10^{-3}$, triggering Russian Roulette to prevent unnecessary computation [S1, unofficial].
*   **Optical Depth Calculation:** The source uses `0.5 * step` for the volume tau calculation in the main loop and `mStepSize * 2` for the shadow ray. These specific values are implementation details that may affect accuracy [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples `numSamples`, which is proportional to the ray length divided by `mStepSize` [S1, unofficial].
*   **Memory Complexity:** Constant, as it only stores a few accumulators and temporary rays per step [S1, unofficial].
*   **Strengths:** Simple implementation for single-volume, single-light scenarios; uses Russian Roulette for efficiency; combines voxel acceleration with ray marching [S1, unofficial].
*   **Limitations:** Limited to single volume and single light; does not handle multiple scattering; specific tau calculation parameters (0.5*step, mStepSize*2) are hardcoded and may not be optimal for all scenes [S1, unofficial].
*   **When to Use:** Suitable for simple volumetric effects in a controlled environment where performance is prioritized over physical accuracy for complex lighting [S1, unofficial].
*   **When Not to Use:** For scenes with multiple light sources, multiple volumes, or requiring high-fidelity multi-scattering [S1, unofficial].

## Background
Mr-Voxel is a combination of an accelerated voxel data-structure and volume rendering library, developed by C. Gibson [S1, unofficial]. The volume rendering component is part of the author's graphics graduate thesis, which culminated in research and design surrounding voxel data structures [S1, unofficial]. The lighting equations used are derived or loosely based on the volume rendering sections of "Physically Based Rendering, 2nd edition" by Matt Pharr [S1, unofficial]. The long-term goal involves including importance-sampled light caches for all polygonal, implicit, and volume objects [S1, unofficial].

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
