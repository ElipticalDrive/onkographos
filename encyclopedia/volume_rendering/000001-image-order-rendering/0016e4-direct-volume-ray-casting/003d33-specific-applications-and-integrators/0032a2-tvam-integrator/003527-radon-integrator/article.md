# Radon Integrator
## Overview
The Radon Integrator computes the Radon transform (projection) of a volume by tracing rays through a purely absorptive medium and accumulating attenuation along the path [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Scene geometry (target mesh, vial, projector, sensor), volume density/absorption coefficients (via medium interaction), configuration parameters (spp, max_depth, transmission_only) [S1, unofficial].
- Output: A tensor representing the projected attenuation (Radon transform) accumulated per pixel [S1, unofficial].

**Data Structures:**
- `active`: Boolean flag indicating if the ray is still valid and within bounds [S1, unofficial].
- `inside_target`: Boolean flag tracking if the ray is currently inside the target geometry [S1, unofficial].
- `active_medium`: Boolean flag tracking if the ray is currently inside a medium that contributes to absorption [S1, unofficial].
- `throughput`: Scalar float, initial value 1.0, accumulates the product of BSDF weights [S1, unofficial].
- `L`: Scalar float, initial value 0.0, accumulates the total contribution [S1, unofficial].
- `t`: Scalar float, initial value 0.0, accumulates distance traveled within the medium [S1, unofficial].
- `d`: Integer, current ray depth [S1, unofficial].

**Numbered Steps:**
1. **Render Initialization**: In the main render loop, suspend automatic differentiation gradients [S1, unofficial]. Prepare the sampler and generate the initial ray [S1, unofficial]. Call the `sample()` function with `ADMode.Primal` to ensure primal values are computed without gradient tracking during the path traversal [S1, unofficial].
2. **Weighting and Accumulation**: Multiply the result returned by `sample()` by the ray weight [S1, unofficial]. Scatter-reduce this value into the output image tensor using an additive operation (`Add`) [S1, unofficial].
3. **Sample Initialization**: Inside `sample()`, initialize `active_medium = False`, `inside_target = False`, `throughput = 1.0`, `L = 0.0`, `t = 0.0`, and `d = 0` [S1, unofficial].
4. **Medium Retrieval**: Get the scattering coefficient $\sigma_s$ and extinction coefficient $\sigma_t$ from the current medium context [S1, unofficial]. The method assumes a purely absorptive medium, implying $\sigma_s$ is ignored in the final contribution formula [S1, unofficial].
5. **Ray Tracing Loop**: Loop while `active` is true:
    a. Intersect the ray with the scene geometry [S1, unofficial].
    b. Update `active &= si.is_valid()`, where `si` is the scene intersection [S1, unofficial].
    c. **Contribution Calculation**: Compute the contribution for the current segment:
        $$ \text{contrib} = \text{throughput} \times \exp(-\sigma_t \times t) \times (1 - \exp(-\sigma_t \times \text{si}.t)) $$
        This formula assumes the medium is purely absorptive, where $\text{si}.t$ is the distance to the next intersection [S1, unofficial].
    d. **Accumulation**: If `hit_target` and `active_medium` are true, add `contrib` to `L` [S1, unofficial]. Update `t += si.t` [S1, unofficial].
    e. **State Update (Inside Target)**: Update `inside_target` state using the XOR-like logic:
        $$ \text{inside\_target} = (\neg \text{inside\_target} \land \text{hit\_target}) \lor (\text{inside\_target} \land \neg \text{hit\_target}) $$
        This logic may cause some rays to leak out of the target volume if intersection validity is ambiguous [S1, unofficial].
    f. **BSDF Sampling**: Sample the BSDF at the intersection [S1, unofficial]. If `transmission_only` is true, set the context type mask to `Transmission` [S1, unofficial]. Else, if `depth == 0` and `active`, force refraction sampling to avoid generating useless rays that do not penetrate the volume [S1, unofficial].
    g. **Throughput and Depth**: Update `throughput *= bs_w` (BSDF weight) [S1, unofficial]. Spawn a new ray [S1, unofficial]. Increment `d` for non-target hits [S1, unofficial]. Update `active &= (d < max_depth)` [S1, unofficial].
    h. **Medium Transition**: Update `active_medium` state:
        $$ \text{active\_medium} = (\text{active\_medium} \land \neg \text{si.is\_medium\_transition}()) \lor (\text{si.is\_medium\_transition}() \land (\text{si.target\_medium}(\text{ray}.d) \neq \text{None})) $$
        This checks if the ray is transitioning between media and if the new target medium is valid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def radon_integrator_render(scene, sampler, config):
    spp = config.get('spp', 4)
    max_depth = config.get('max_depth', 10)
    transmission_only = config.get('transmission_only', False)
    
    output_tensor = np.zeros(scene.sensor_shape)
    
    for _ in range(spp):
        ray = sampler.next_ray()
        L = sample_radon(ray, scene, max_depth, transmission_only)
        weight = 1.0
        output_tensor[ray.index] += L * weight
            
    return output_tensor

def sample_radon(ray, scene, max_depth, transmission_only):
    active = True
    inside_target = False
    active_medium = False
    throughput = 1.0
    L = 0.0
    t = 0.0
    d = 0
    
    while active:
        si = scene.intersect(ray)
        active = active and si.is_valid()
        
        if not active:
            break
            
        sigma_s, sigma_t = scene.get_medium_coefficients(si)
        
        hit_target = si.is_target_hit()
        if hit_target and active_medium:
            contrib = throughput * math.exp(-sigma_t * t) * (1.0 - math.exp(-sigma_t * si.t))
            L += contrib
            
        t += si.t
        
        inside_target = (not inside_target and hit_target) or (inside_target and not hit_target)
        
        if transmission_only:
            ctx_type_mask = 'Transmission'
        else:
            if d == 0 and active:
                ctx_type_mask = 'Refraction'
            else:
                ctx_type_mask = 'All'
                
        bs_w = scene.sample_bsdf(si, ctx_type_mask)
        throughput *= bs_w
        
        ray = scene.spawn_new_ray(ray, si, bs_w)
        if not hit_target:
            d += 1
            
        active = active and (d < max_depth)
        
        is_transition = si.is_medium_transition()
        new_medium = si.target_medium(ray.d)
        active_medium = (active_medium and not is_transition) or (is_transition and new_medium is not None)
        
    return L
```

**Parameters:**
- `spp`: Samples per pixel. Typical value: 4 [S1, unofficial].
- `max_depth`: Maximum ray depth. Not given in the sources.
- `transmission_only`: Boolean flag to restrict sampling to transmission. Not given in the sources.

**Edge Cases and Pitfalls:**
- The `inside_target` state update logic may cause rays to leak out of the target volume if intersection validity is ambiguous [S1, unofficial].

## Subtopics

### Half-Precision Radon Transform
An optimization technique that stores input and output data in half precision (16 bits) while performing computations in single precision (32 bits) [S2, unofficial]. Radon transforms are memory bound operations where arithmetic takes negligible time, making cache hit rates critical for performance [S2, unofficial]. The method stores inputs in CUDA Textures, which are read-only data structures with opaque memory layouts optimized for texture fetching, improving cache hit rates for line integrals [S2, unofficial]. This implementation achieves up to 125× faster execution than the Astra Toolbox [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of samples (spp) and ray depth, as each ray traverses the scene geometry and medium [G].
- **Memory Complexity**: Constant per ray, storing state variables (`active`, `throughput`, `L`, `t`, `d`) [G].
- **Strengths**: Efficient for computing pure absorption projections (Radon transform) without complex scattering calculations [G].
- **Limitations**: Assumes purely absorptive medium; may suffer from ray leakage due to `inside_target` logic [G].
- **When to use**: When the goal is to compute the tomographic projection (absorbed dose) for TVAM optimization [G].
- **When not to use**: When scattering or emission is significant, or when corner-based boundary handling is required [G].

## Background
The Radon Integrator is part of the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] TorchRadon: Fast Differentiable Routines for Computed Tomography, https://doi.org/10.48550/arxiv.2009.14788, fetched 2026-10-26 (unofficial)
