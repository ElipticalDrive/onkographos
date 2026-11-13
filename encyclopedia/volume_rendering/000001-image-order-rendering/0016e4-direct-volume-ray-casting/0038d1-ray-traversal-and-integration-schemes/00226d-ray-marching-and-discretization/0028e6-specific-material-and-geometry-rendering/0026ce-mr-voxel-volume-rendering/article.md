# Mr-Voxel Volume Rendering

Also called: VolumeIntegrator::Li

## Overview
Mr-Voxel Volume Rendering is a direct volume ray casting method implemented within the `VolumeIntegrator` class of the Mr-Voxel library [S1, unofficial]. It performs ray marching through a single volume region to accumulate emitted light and compute scattering contributions from a single light source [S1, unofficial]. It is distinguished from sibling methods by its specific implementation of Russian Roulette termination and its reliance on a pre-computed optical depth (`tau`) query for transmittance, rather than explicit density integration per step [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A camera ray $r(t) = o + t \cdot d$ with origin $o$ and direction $d$; the scene graph containing volumes and lights.
*   **Output:** A `Spectrum` representing the accumulated radiance $L_v$ and an updated transmittance value $T$ [S1, unofficial].

**Data Structures:**
*   `VolumeIntegrator`: Contains member `mStepSize` (float) defining the nominal step length [S1, unofficial].
*   `Scene`: Provides access to `getVolumeCount()`, `getVolumes()` (array of volume objects), and `getLightSources()` [S1, unofficial].
*   `Volume`: Provides methods `tau(Ray, float, int)` to compute optical depth, `Lve(Point)` for emitted radiance, and `sigma_s(Point)` for scattering coefficient [S1, unofficial].

**Parameters:**
*   `mStepSize`: Nominal step size for ray marching. Typical value: Not given in the sources.
*   `Russian Roulette Probability`: Fixed at 0.5 for termination checks [S1, unofficial].
*   `Transmittance Threshold`: 1e-3 for triggering Russian Roulette [S1, unofficial].

**Algorithm Steps:**
1.  **Interval Calculation:** Determine the intersection interval $[t_0, t_1]$ of the ray with the volume domain.
2.  **Sample Count:** Calculate the number of steps:
    $$ N = \left\lceil \frac{t_1 - t_0}{\text{mStepSize}} \right\rceil $$
    If $N = 0$, set $N = 1$ [S1, unofficial].
3.  **Step Length:** Compute the actual step size:
    $$ \Delta t = \frac{t_1 - t_0}{N} $$
4.  **Jittering:** Apply a random offset to the starting position to reduce aliasing:
    $$ t_{start} = t_0 + U[0,1) \cdot \Delta t $$
    where $U[0,1)$ is a uniform random number [S1, unofficial].
5.  **Initialization:** Set accumulated radiance $L_v = 0$ and transmittance $T_r = 1$ [S1, unofficial].
6.  **Marching Loop:** For $i = 0$ to $N-1$:
    a.  Define current position $p = r(t_{start} + i \cdot \Delta t)$ and previous position $p_{prev} = r(t_{start} + (i-1) \cdot \Delta t)$ (for $i=0$, $p_{prev}$ is the ray origin or $t_0$) [S1, unofficial].
    b.  Create a short ray segment `tRay` from $p_{prev}$ to $p$ with length $\Delta t$ [S1, unofficial].
    c.  **Optical Depth:** Compute optical depth for the segment:
        $$ \tau_{step} = \text{volumes}[0].\text{tau}(\text{tRay}, 0.5 \cdot \Delta t, 0) $$
        [S1, unofficial].
    d.  **Transmittance Update:**
        $$ T_r \leftarrow T_r \cdot e^{-\tau_{step}} $$
        [S1, unofficial].
    e.  **Emission Accumulation:**
        $$ L_v \leftarrow L_v + \text{volumes}[0].\text{Lve}(p) $$
        [S1, unofficial].
    f.  **Scattering Check:** Get scattering coefficient $\sigma_s = \text{volumes}[0].\sigma_s(p)$ [S1, unofficial].
    g.  **Lighting:** If $\sigma_s$ is non-zero:
        i.  Compute light direction $\omega_L$ from $p$ to light position, normalized.
        ii. Create shadow ray from $p$ in direction $\omega_L$.
        iii. Check occlusion: If the shadow ray intersects a surface at distance $t_{surf}$ such that $t_{surf} < \|p - \text{light\_pos}\|$, the point is shadowed. Otherwise, it is unshadowed [S1, unofficial].
        iv. If unshadowed, compute light transmittance through volume:
            $$ T_{light} = e^{-\text{volumes}[0].\text{tau}(\text{shadowRay}, 2 \cdot \text{mStepSize}, 0)} $$
            [S1, unofficial].
        v.  Compute direct lighting contribution:
            $$ L_d = T_{light} \cdot \text{lights}[0].\text{color} $$
            [S1, unofficial].
        vi. Add to accumulated radiance:
            $$ L_v \leftarrow L_v + L_d \cdot \sigma_s \cdot T_r $$
            [S1, unofficial].
    h.  **Termination (Russian Roulette):** If $T_r < 10^{-3}$:
        i.  Generate random number $r \in [0,1)$.
        ii. If $r < 0.5$, break the loop [S1, unofficial].
        iii. Else, scale transmittance: $T_r \leftarrow T_r / 0.5$ and continue [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mr_voxel_volume_rendering(ray, scene, mStepSize):
    # 1. Intersection
    t0, t1 = scene.ray_volume_intersection(ray)
    if t0 > t1:
        return Spectrum(0.0), 1.0

    # 2. Sample Count and Step Size
    num_samples = int((t1 - t0) / mStepSize)
    if num_samples == 0:
        num_samples = 1
    
    step = (t1 - t0) / num_samples
    
    # 3. Jittering
    t_start = t0 + random.uniform(0.0, 1.0) * step
    
    # 4. Initialization
    Lv = Spectrum(0.0)
    Tr = 1.0
    
    # Access single volume and light as per implementation notes
    volume = scene.get_volumes()[0]
    light = scene.get_light_sources()[0]
    
    for i in range(num_samples):
        # Current and previous positions
        t_curr = t_start + i * step
        t_prev = t_start + (i - 1) * step if i > 0 else t0
        
        p_curr = ray(t_curr)
        p_prev = ray(t_prev)
        
        # Create short ray for optical depth calculation
        # Note: Mr-Voxel's tau function takes a ray and a length
        t_ray = Ray(p_prev, p_curr - p_prev)
        t_ray.length = step
        
        # 5. Optical Depth and Transmittance
        step_tau = volume.tau(t_ray, 0.5 * step, 0)
        Tr = Tr * exp(-step_tau)
        
        # 6. Emission
        Lv = Lv + volume.Lve(p_curr)
        
        # 7. Scattering
        sigma_s = volume.sigma_s(p_curr)
        
        # Check if scattering coefficient is non-black/non-zero
        if not is_black(sigma_s):
            # Light direction
            light_pos = light.position()
            w_L = normalize(light_pos - p_curr)
            
            # Shadow Ray
            shad_ray = Ray(p_curr, w_L)
            surface_hit = scene.intersect(shad_ray)
            
            is_shadowed = False
            if surface_hit is not None:
                dist_to_light = length(light_pos - p_curr)
                if surface_hit.t < dist_to_light:
                    is_shadowed = True
            
            if not is_shadowed:
                # Light transmittance
                # Note: Mr-Voxel uses 2 * mStepSize for light tau query
                l_tau = volume.tau(shad_ray, 2 * mStepSize, 0)
                l_tr = exp(-l_tau)
                
                L_d = l_tr * light.color()
                Lv = Lv + L_d * sigma_s * Tr
        
        # 8. Russian Roulette Termination
        if Tr < 1e-3:
            r = random.uniform(0.0, 1.0)
            if r < 0.5:
                break
            else:
                Tr = Tr / 0.5
                
    return Lv, Tr
```

**Edge Cases and Pitfalls:**
*   **Single Volume/Light Assumption:** The implementation assumes `volumes[0]` and `lights[0]` exist. If the scene has no volumes or lights, accessing index 0 will crash. A TODO comment indicates multiple volumes require aggregators [S1, unofficial].
*   **Zero Step Count:** If the ray segment is shorter than `mStepSize`, `num_samples` is forced to 1 to prevent division by zero or infinite loops [S1, unofficial].
*   **Precision:** The use of `exp(-tau)` requires `tau` to be computed accurately. The `tau` function is a black-box query to the volume structure [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of steps along the ray. Each step involves a constant number of volume queries and potentially one shadow ray intersection [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, excluding the storage of the volume data structure itself [S1, unofficial].
*   **Strengths:** Simple implementation; jittering reduces aliasing; Russian Roulette handles deep volumes efficiently by terminating rays with low transmittance [S1, unofficial].
*   **Limitations:** Limited to single volume and single light source in the current implementation; performance depends heavily on the efficiency of the `tau` query [S1, unofficial].
*   **When to use:** Suitable for simple volumetric scenes with a single light source where performance is prioritized over multi-light accuracy [S1, unofficial].

## Background
Mr-Voxel is a combination of an accelerated voxel data-structure and volume rendering library [S1, unofficial]. The volume rendering component, specifically the `VolumeIntegrator::Li` function, is part of the author's graphics graduate thesis [S1, unofficial]. The lighting equations are derived or loosely based on the volume rendering sections of "Physically Based Rendering, 2nd edition" by Matt Pharr [S1, unofficial]. The project aims to include importance-sampled light caches for polygonal, implicit, and volume objects [S1, unofficial].

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
