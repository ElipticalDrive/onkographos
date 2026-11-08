# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric additive manufacturing (TVAM) simulations that handles projector ray sampling, time sampling, and regular versus stochastic sampling modes [S1, unofficial]. It serves as the foundation for specialized integrators like the Radon Integrator and CornerIntegrator [S1, unofficial]. Unlike general-purpose volume renderers, it is designed to simulate the optical projection of Digital Micromirror Devices (DMDs) onto a photosensitive resin vat, managing the specific constraints of time-multiplexed tomographic exposure [S1, unofficial].

## How it works
The integrator manages the generation of rays from projector emitters, accounting for both spatial pixel coordinates and temporal exposure patterns. It supports both regular (deterministic) and stochastic sampling strategies.

**Inputs and Outputs:**
*   **Inputs:** Scene configuration containing a target shape (by ID), an interior medium (resin), and projector emitters. Parameters include `sample_time`, `print_time`, `target_id`, `transmission_only`, `regular_sampling`, `K`, `M`, `tp`, `top`, `tinhibitor`, `weight_object`, `weight_void`, `weight_limit`, and `weight_sparsity` [S1, unofficial].
*   **Outputs:** A set of rays with associated weights, ready for traversal through the volume medium [S1, unofficial].

**Data Structures:**
*   **Ray Buffer:** Stores ray origins, directions, and weights.
*   **Pixel Index Mapping:** Maps active pixels to ray indices.

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample [S1, unofficial].
*   `print_time` (float, default `1.0`): Used to scale the ray weight by $1/\text{print\_time}$ [S1, unofficial].
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization [S1, unofficial].
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces [S1, unofficial].
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset [S1, unofficial].
*   `K` (int, default `2`): Exponent for physical model evaluation [S1, unofficial].
*   `M` (int, default `4`): Exponent for sparsity evaluation [S1, unofficial].
*   `tp` (float, default `0.1`): Threshold parameter for polymerization [S1, unofficial].
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization [S1, unofficial].
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibition [S1, unofficial].
*   `weight_object` (float, default `1`): Weight for object regions [S1, unofficial].
*   `weight_void` (float, default `1`): Weight for void regions [S1, unofficial].
*   `weight_limit` (float, default `1`): Weight for limit conditions [S1, unofficial].
*   `weight_sparsity` (float, default `0`): Weight for sparsity penalty [S1, unofficial].

**Steps:**

1.  **Parse Scene:** Iterate over scene shapes to identify the target shape (matching `target_id`) and the interior medium. Raise an error if more than one medium is found [S1, unofficial].
2.  **Prepare:**
    *   If `regular_sampling` is true, force samples per pixel (spp) to 1, as shooting from the pixel center makes spp > 1 nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].
3.  **Sample Rays:**
    *   Compute the discrete sample position index `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ \text{pos.y} = \text{pixel\_idx} // w $$
        $$ \text{pos.x} = \text{pixel\_idx} - \text{row\_idx} \times w $$
        where $w$ is the width of the projector resolution [S1, unofficial].
    *   Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is true, add a 0.5 offset.
        *   Else, add `sampler.next_2d()` (a random 2D value).
        *   Scale by $1/w$ and $1/h$ respectively [S1, unofficial].
    *   Compute time sample:
        $$ \text{time} = \text{emitter\_idx} / n\_patterns $$
        If `sample_time` is true, add `sampler.next_1d()` (a random 1D value) before dividing by $n\_patterns$ [S1, unofficial].
    *   Sample the ray from the emitter using the computed spatial and temporal coordinates.
    *   Scale the ray weight by $1/\text{print\_time}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_scene(scene):
    target_shape = None
    interior_medium = None
    for shape in scene.shapes:
        if shape.id == target_id:
            target_shape = shape
        if shape.is_medium():
            if interior_medium is not None:
                raise ValueError("More than one medium found in scene")
            interior_medium = shape
    if target_shape is None or interior_medium is None:
        raise ValueError("Target shape or interior medium not found")
    return target_shape, interior_medium

def prepare(active_size, spp):
    if regular_sampling:
        spp = 1
    wavefront_size = active_size * spp
    if wavefront_size > 2**32:
        raise ValueError("Wavefront size exceeds 2^32")
    return spp

def sample_rays(active_pixels, w, h, emitter_idx, n_patterns, sampler):
    spp = prepare(len(active_pixels), 1 if regular_sampling else 1) # spp is forced to 1 if regular
    # In regular mode, spp is 1. In stochastic, spp can be > 1.
    # The source implies spp is determined by regular_sampling flag logic in prepare.
    # Let's assume spp is passed or derived.
    
    # Repeat active_pixels spp times
    indices = []
    for _ in range(spp):
        indices.extend(active_pixels)
        
    rays = []
    for pixel_idx in indices:
        row_idx = pixel_idx // w
        pos_y = pixel_idx // w
        pos_x = pixel_idx - row_idx * w
        
        # Convert to [0,1]
        if regular_sampling:
            u = (pos_x + 0.5) / w
            v = (pos_y + 0.5) / h
        else:
            u = (pos_x + sampler.next_2d()[0]) / w
            v = (pos_y + sampler.next_2d()[1]) / h
            
        # Time sample
        time = emitter_idx / n_patterns
        if sample_time:
            time = (emitter_idx + sampler.next_1d()) / n_patterns
            
        # Sample ray from emitter
        ray = emitter.sample(u, v, time)
        
        # Scale weight
        ray.weight *= 1.0 / print_time
        rays.append(ray)
    return rays
```

**Edge Cases and Pitfalls:**
*   **Wavefront Overflow:** If the number of active pixels multiplied by samples per pixel exceeds $2^{32}$, the method raises an exception [S1, unofficial].
*   **Multiple Mediums:** The parser strictly enforces a single interior medium; multiple mediums cause an error [S1, unofficial].
*   **Regular Sampling:** Forcing spp to 1 in regular sampling mode prevents nonsensical multi-sample accumulation at pixel centers [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Filters out rays intersecting corners of the build volume to solve discretization artifacts from DMD pixels.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform by tracing rays through a purely absorptive medium, accumulating attenuation.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot \text{spp})$) for ray generation.
*   **Memory Complexity:** Linear with respect to the wavefront size ($O(N \cdot \text{spp})$) to store ray data.
*   **Strengths:** Provides a flexible base for TVAM simulation, supporting both deterministic and stochastic sampling, and integrating time-multiplexed exposure patterns.
*   **Limitations:** Base class does not perform volume traversal or compositing itself; it only generates rays. Specialized integrators are required for actual dose calculation.
*   **When to use:** When implementing custom TVAM simulation methods that require precise control over projector ray sampling and time exposure.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It was developed to simulate the complex optical interactions in DMD-based 3D printing, where light patterns are projected from multiple angles over time to polymerize resin [S1, unofficial]. The class structure allows for the extension of specific physical models and boundary conditions, such as corner masking or pure absorption, through derived classes [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
