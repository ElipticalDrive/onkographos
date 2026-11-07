# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric integrators in Tomographic Volumetric Additive Manufacturing (TVAM) that manages projector ray sampling, time sampling, and switching between regular and stochastic sampling modes [S1, unofficial]. It serves as the foundation for specialized integrators like the [CornerIntegrator](7445-cornerintegrator.md) and [Radon Integrator](7470-radon-integrator.md) by providing the infrastructure for sampling rays from projector emitters and handling the temporal sequence of light patterns [S1, unofficial]. Unlike standard volume rendering integrators, it explicitly accounts for the discrete nature of Digital Micromirror Device (DMD) pixels and the temporal multiplexing of projection patterns [S1, unofficial].

## How it works
The integrator prepares rays for emission from projector emitters, samples their positions and times, and computes initial weights based on the sampling strategy. It relies on a scene containing a target shape and a single interior medium [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Scene configuration (shapes, emitters), sampling parameters (`regular_sampling`, `sample_time`), and physical model parameters (e.g., `K`, `M`, `tp`, `top`, `tinhibitor`, weights).
*   **Outputs:** Ray origins, directions, weights, and time samples for the current pixel.

**Data Structures:**
*   **Scene:** Contains a target shape identified by `target_id` and exactly one interior medium.
*   **Emitter:** Defines the projector geometry and pattern sequence.
*   **Sampler:** Provides random numbers for stochastic sampling (`next_2d`, `next_1d`).

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample [S1, unofficial].
*   `print_time` (float, default `1.0`): Used to scale the ray weight [S1, unofficial].
*   `target_id` (str, default `'target'`): The ID of the target shape [S1, unofficial].
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces [S1, unofficial].
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset [S1, unofficial].
*   `K` (int, default `2`): Exponent for physical model evaluation [S1, unofficial].
*   `M` (int, default `4`): Exponent for sparsity evaluation [S1, unofficial].
*   `tp` (float, default `0.1`): Threshold parameter for polymerization [S1, unofficial].
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization [S1, unofficial].
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibition [S1, unofficial].
*   `weight_object` (float, default `1`): Weight for object region [S1, unofficial].
*   `weight_void` (float, default `1`): Weight for void region [S1, unofficial].
*   `weight_limit` (float, default `1`): Weight for limit region [S1, unofficial].
*   `weight_sparsity` (float, default `0`): Weight for sparsity term [S1, unofficial].

**Steps:**

1.  **Prepare:**
    *   If `regular_sampling` is true, force samples per pixel (`spp`) to 1, as shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].

2.  **Parse Scene:**
    *   Iterate over scene shapes to find the target shape by `target_id` and the interior medium [S1, unofficial].
    *   Raise an error if more than one interior medium is found [S1, unofficial].

3.  **Sample Rays:**
    *   Compute discrete sample position `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$
        \text{row\_idx} = \lfloor \text{pixel\_idx} / w \rfloor
        $$
        $$
        \text{pos}.y = \text{row\_idx}
        $$
        $$
        \text{pos}.x = \text{pixel\_idx} - \text{row\_idx} \times w
        $$
        where $w$ is the pixel width [S1, unofficial].
    *   Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is true, add 0.5 offset to both coordinates.
        *   Else, add `sampler.next_2d()` to both coordinates.
        *   Scale by $1/w$ and $1/h$ respectively [S1, unofficial].
    *   Compute time sample:
        $$
        \text{time} = \frac{\text{emitter\_idx}}{n\_patterns}
        $$
        *   If `sample_time` is true, add `sampler.next_1d()` to the numerator before dividing [S1, unofficial].
    *   Sample ray from emitter and scale weight by `print_time` (specifically, multiply by $1/\text{pdf of time sample}$, which simplifies to scaling by `print_time` in the context of uniform time sampling) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare(self, active_size, spp):
    if self.regular_sampling:
        spp = 1
    if active_size * spp > 2**32:
        raise Exception("Wavefront size exceeds 2^32")
    return spp

def parse_scene(self, scene):
    target = None
    medium = None
    for shape in scene.shapes:
        if shape.id == self.target_id:
            target = shape
        if shape.medium is not None:
            if medium is not None:
                raise Exception("More than one medium found")
            medium = shape.medium
    if target is None or medium is None:
        raise Exception("Target or medium not found")
    return target, medium

def sample_rays(self, active_pixels, w, h, emitter, sampler, n_patterns):
    # idx repeats active_pixels spp times
    # For each sample i in range(active_pixels * spp):
    pixel_idx = i % active_pixels
    row_idx = pixel_idx // w
    pos_y = row_idx
    pos_x = pixel_idx - row_idx * w
    
    if self.regular_sampling:
        pos_x += 0.5
        pos_y += 0.5
    else:
        pos_x += sampler.next_2d()[0]
        pos_y += sampler.next_2d()[1]
        
    pos_x /= w
    pos_y /= h
    
    time = emitter.idx / n_patterns
    if self.sample_time:
        time += sampler.next_1d()
    time /= n_patterns
    
    ray = emitter.sample(pos_x, pos_y, time)
    ray.weight *= self.print_time # Simplified from 1/pdf
    return ray
```

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized integrator that filters out rays intersecting corners of the target build volume to solve discretization artifacts.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform (projection) of a volume by tracing rays through a purely absorptive medium.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot \text{spp})$) for ray generation.
*   **Memory Complexity:** Linear with respect to the wavefront size ($O(N \cdot \text{spp})$) for storing ray states.
*   **Strengths:** Provides a unified framework for TVAM simulation, handling both regular and stochastic sampling, and integrating with physical models for polymerization and inhibition.
*   **Limitations:** The base class does not implement the specific physical interaction models (e.g., polymerization kinetics) itself; these are handled by derived classes or external physical models. The constraint on `wavefront_size` limits the maximum resolution and samples per pixel that can be processed in a single batch.
*   **When to use:** When implementing a TVAM-specific integrator that requires projector ray sampling and time multiplexing.
*   **When not to use:** For standard volume rendering without TVAM-specific requirements like DMD pixel sampling or temporal pattern sequencing.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It was developed to simulate the light propagation and polymerization processes in TVAM, enabling the optimization of projection patterns for 3D printing [S1, unofficial]. The framework integrates with Mitsuba 2 for rendering and Dr.Jit for differentiable programming [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
