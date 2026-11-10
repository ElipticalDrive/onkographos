# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric integrators in Tomographic Volumetric Additive Manufacturing (TVAM) that handles projector ray sampling, time sampling, and regular versus stochastic sampling modes [S1, unofficial]. It solves the problem of correctly sampling the temporal sequence of projector patterns and mapping them to spatial pixel coordinates for inverse rendering optimization [S1, unofficial]. It differs from sibling integrators by focusing on the specific temporal and geometric constraints of DMD-based projection systems, such as enforcing single-sample-per-pixel for regular sampling and managing the weight scaling based on print time [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene configuration containing a target shape (identified by `target_id`) and an interior medium; projector parameters (`K`, `M`, `tp`, `top`, `tinhibitor`, weights); sampling flags (`sample_time`, `regular_sampling`).
*   **Outputs:** Ray directions, weights, and time samples for each pixel in the active set.

**Data Structures:**
*   `active_pixels`: List of pixel indices to be sampled.
*   `spp`: Samples per pixel (integer).
*   `wavefront_size`: Total number of samples (`active_size * spp`).
*   `emitter_idx`: Index of the current projector pattern/time step.
*   `n_patterns`: Total number of patterns in the sequence.

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample.
*   `print_time` (float, default `1.0`): Used to scale the ray weight by `1/print_time`.
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization.
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces.
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset.
*   `K` (float, default `2`): Exponent for the physical model evaluation.
*   `M` (float, default `4`): Exponent for sparsity evaluation.
*   `tp` (float, default `0.1`): Threshold parameter for polymerization.
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization.
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibition.
*   `weight_object` (float, default `1`): Weight for object regions.
*   `weight_void` (float, default `1`): Weight for void regions.
*   `weight_limit` (float, default `1`): Weight for limit regions.
*   `weight_sparsity` (float, default `0`): Weight for sparsity penalty.

**Numbered Steps:**

1.  **Prepare Phase:**
    1.  If `regular_sampling` is `True`, force `spp` to `1`. Shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    2.  Calculate `wavefront_size` as `active_size * spp`. If `wavefront_size` exceeds $2^{32}$, raise an exception [S1, unofficial].

2.  **Sample Rays Phase:**
    1.  Compute discrete sample position `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    2.  Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ pos.y = \lfloor pixel\_idx / w \rfloor $$
        $$ pos.x = pixel\_idx - row\_idx \times w $$
        where $w$ is the image width and $row\_idx$ is the row index [S1, unofficial].
    3.  Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is `True`, add $0.5$ offset.
        *   Else, add `sampler.next_2d()` (stochastic offset).
        *   Scale by $1/w$ for x and $1/h$ for y [S1, unofficial].
    4.  Compute time sample:
        $$ time = emitter\_idx / n\_patterns $$
        If `sample_time` is `True`, add `sampler.next_1d()` before dividing by `n_patterns` [S1, unofficial].
    5.  Sample ray from emitter and scale weight by `print_time` (which is $1/pdf$ of time sample) [S1, unofficial].

3.  **Parse Scene Phase:**
    1.  Iterate over scene shapes to find the target shape (by `target_id`) and the interior medium [S1, unofficial].
    2.  Raise an error if more than one medium is found [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare(self, active_pixels, spp):
    # Step 1.1: Enforce single sample for regular sampling
    if self.regular_sampling:
        spp = 1
    
    # Step 1.2: Check wavefront size limit
    active_size = len(active_pixels)
    wavefront_size = active_size * spp
    if wavefront_size > 2**32:
        raise Exception("Wavefront size exceeds 2^32")
    
    return spp

def sample_rays(self, active_pixels, spp, emitter_idx, n_patterns, sampler):
    # Step 2.1: Repeat active pixels spp times
    indices = []
    for pixel in active_pixels:
        for _ in range(spp):
            indices.append(pixel)
            
    w = self.width
    h = self.height
    
    rays = []
    weights = []
    
    for pixel_idx in indices:
        # Step 2.2: Compute pixel coordinates
        row_idx = pixel_idx // w
        pos_y = row_idx
        pos_x = pixel_idx - row_idx * w
        
        # Step 2.3: Convert to [0,1] range
        if self.regular_sampling:
            u = (pos_x + 0.5) / w
            v = (pos_y + 0.5) / h
        else:
            u = (pos_x + sampler.next_2d()[0]) / w
            v = (pos_y + sampler.next_2d()[1]) / h
            
        # Step 2.4: Compute time sample
        if self.sample_time:
            time = (emitter_idx + sampler.next_1d()) / n_patterns
        else:
            time = emitter_idx / n_patterns
            
        # Step 2.5: Sample ray and scale weight
        # Note: Exact ray construction depends on emitter implementation [G]
        ray = self.emitter.sample_ray(u, v, time)
        
        # Scale weight by print_time (1/pdf of time sample)
        # Assuming uniform time distribution for simplicity if pdf not provided [G]
        weight = self.print_time 
        
        rays.append(ray)
        weights.append(weight)
        
    return rays, weights

def parse_scene(self, scene):
    # Step 3.1: Find target and medium
    target_shape = None
    medium = None
    medium_count = 0
    
    for shape in scene.shapes:
        if shape.id == self.target_id:
            target_shape = shape
        if shape.medium is not None:
            medium = shape.medium
            medium_count += 1
            
    # Step 3.2: Error if multiple mediums
    if medium_count > 1:
        raise Exception("More than one medium found in scene")
        
    return target_shape, medium
```

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized integrator that filters out rays intersecting corners of the build volume to solve discretization artifacts.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform by tracing rays through a purely absorptive medium.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot spp)$).
*   **Memory Complexity:** Linear with respect to the wavefront size ($O(N \cdot spp)$) to store ray and weight data.
*   **Strengths:** Handles both regular and stochastic sampling modes; manages temporal sampling for projector patterns; enforces constraints specific to TVAM (e.g., single sample for regular sampling).
*   **Limitations:** Limited to the specific sampling and weighting logic defined in the base class; requires careful configuration of physical model parameters (`K`, `M`, thresholds).
*   **When to use:** When implementing inverse rendering for TVAM where projector temporal patterns and spatial sampling need to be handled correctly.
*   **When not to use:** When global illumination effects or complex scattering models are required beyond the base transmission sampling.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It is part of the Mitsuba 2 renderer ecosystem, adapted for the specific needs of volumetric 3D printing optimization [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
