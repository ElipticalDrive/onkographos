# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric additive manufacturing (TVAM) integrators that handles projector ray sampling, time sampling, and regular versus stochastic sampling modes [S1, unofficial]. It diverges from standard Direct Volume Ray Casting by focusing on the temporal and geometric sampling of projector emissions rather than material compositing [S1, unofficial]. It provides the generic sampling infrastructure (pixel, time, and emitter selection) upon which specific integrators like the Radon Integrator or CornerIntegrator build their accumulation logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene shapes, `active_pixels`, `spp`, `sampler`, `n_patterns`, `emitter_idx`, image dimensions `w` and `h`, emitter object, and parameters `sample_time`, `print_time`, `target_id`, `transmission_only`, `regular_sampling` [S1, unofficial].
*   **Outputs:** A list of rays defined by origin, direction, and weight [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample [S1, unofficial].
*   `print_time` (float, default `1.0`): Used to scale the ray weight by $1/\text{print\_time}$ [S1, unofficial].
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization [S1, unofficial].
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces [S1, unofficial].
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset [S1, unofficial].

**Steps:**

1.  **Parse Scene:**
    *   Iterate over scene shapes to find the target shape identified by `target_id` and the interior medium [S1, unofficial].
    *   Raise an error if more than one interior medium is found [S1, unofficial].

2.  **Prepare:**
    *   If `regular_sampling` is `True`, force `spp` to 1, because shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$ [S1, unofficial].
    *   If the check fails, raise an exception [S1, unofficial].

3.  **Sample Rays:**
    *   Compute discrete sample position `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ \text{pos.y} = \lfloor \text{pixel\_idx} / w \rfloor $$
        $$ \text{pos.x} = \text{pixel\_idx} - \text{row\_idx} \times w $$
        where $w$ is the image width and $\text{row\_idx} = \text{pos.y}$ [S1, unofficial].
    *   Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is `True`, add 0.5 offset to both coordinates [S1, unofficial].
        *   Else, add `sampler.next_2d()` (a random 2D value) to both coordinates [S1, unofficial].
        *   Scale by $1/w$ and $1/h$ respectively [S1, unofficial].
    *   Compute time sample:
        $$ \text{time} = \text{emitter\_idx} / n\_patterns $$
        *   If `sample_time` is `True`, add `sampler.next_1d()` (a random 1D value) to the numerator before dividing [S1, unofficial].
    *   Sample ray from emitter using the computed position and time [S1, unofficial].
    *   Scale ray weight by `print_time` (specifically, multiply by $1/\text{print\_time}$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_scene(self, scene_shapes):
    target = None
    medium_count = 0
    for shape in scene_shapes:
        if shape.id == self.target_id:
            target = shape
        if shape.is_medium:
            medium_count += 1
    if medium_count > 1:
        raise ValueError("More than one interior medium found.")
    return target

def prepare(self, active_pixels, spp):
    if self.regular_sampling:
        spp = 1
    active_size = len(active_pixels)
    if active_size * spp > 2**32:
        raise ValueError("Wavefront size exceeds 2^32.")
    return spp

def sample_rays(self, active_pixels, spp, sampler, n_patterns, emitter_idx, w, h, emitter):
    rays = []
    # Repeat active_pixels spp times
    indices = [idx for _ in range(spp) for idx in active_pixels]
    for pixel_idx in indices:
        row_idx = pixel_idx // w
        pos_y = row_idx
        pos_x = pixel_idx - row_idx * w

        if self.regular_sampling:
            u = (pos_x + 0.5) / w
            v = (pos_y + 0.5) / h
        else:
            offset = sampler.next_2d()
            u = (pos_x + offset[0]) / w
            v = (pos_y + offset[1]) / h

        if self.sample_time:
            time = (emitter_idx + sampler.next_1d()) / n_patterns
        else:
            time = emitter_idx / n_patterns

        ray = emitter.sample(u, v, time)
        ray.weight *= self.print_time
        rays.append(ray)
    return rays
```

**Edge Cases and Pitfalls:**
*   **Wavefront Overflow:** If `active_size * spp` exceeds $2^{32}$, the method raises an exception [S1, unofficial].
*   **Regular Sampling Conflict:** Setting `regular_sampling=True` with `spp > 1` is logically inconsistent and is forced to `spp=1` in the prepare step [S1, unofficial].
*   **Multiple Mediums:** The parser raises an error if more than one interior medium is detected in the scene [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized integrator that filters rays intersecting build volume corners to mitigate DMD discretization artifacts.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform by tracing rays through a purely absorptive medium.

### [HoloVAM](13314-holovam.md)
Uses holographic phase modulation instead of amplitude modulation for light projection.

## Operating characteristics
Not given in the sources.

## Background
The method originates from the `drtvam` repository, an inverse rendering framework for TVAM [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
