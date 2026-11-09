# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric additive manufacturing (TVAM) integrators that handles projector ray sampling, time sampling, and regular vs. stochastic sampling modes [S1, unofficial]. It serves as the architectural foundation for specialized integrators like the [Radon Integrator](7470-radon-integrator.md) and [CornerIntegrator](7445-cornerintegrator.md), providing the core loop for tracing rays from projector emitters through the build volume. It differs from general-purpose volume rendering integrators by explicitly managing the temporal sequence of DMD patterns and the geometric constraints of the projector array.

## How it works
The integrator manages the sampling of rays originating from projector emitters, accounting for both spatial pixel locations and temporal pattern sequences. It supports both regular (deterministic) and stochastic sampling strategies.

**Inputs and Outputs:**
*   **Inputs:** Scene description containing a target shape (identified by `target_id`) and an interior medium; projector emitter definitions; sampling parameters (`regular_sampling`, `sample_time`).
*   **Outputs:** A set of rays, each defined by origin, direction, and weight, ready for traversal through the volume.

**Data Structures:**
*   **Ray Structure:** Contains origin ($O$), direction ($D$), and weight ($W$).
*   **Pixel Indexing:** Discrete indices mapped to floating-point coordinates.

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample.
*   `print_time` (float, default `1.0`): Used to scale the ray weight by $1/\text{print\_time}$.
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization.
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces.
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset.

**Steps:**

1.  **Prepare:**
    *   If `regular_sampling` is true, force samples per pixel (spp) to 1, because shooting from the pixel center makes spp > 1 nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].

2.  **Parse Scene:**
    *   Iterate over scene shapes to find the target shape (by `target_id`) and the interior medium [S1, unofficial].
    *   Raise an error if more than one medium is found [S1, unofficial].

3.  **Sample Rays:**
    *   Compute discrete sample position `idx` by repeating `active_pixels` spp times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ \text{pos.y} = \text{pixel\_idx} // w $$
        $$ \text{pos.x} = \text{pixel\_idx} - \text{row\_idx} \times w $$
        where $w$ is the image width [S1, unofficial].
    *   Convert pixel position to floating point $[0, 1]$ range:
        *   If `regular_sampling`, add 0.5 offset.
        *   Else, add `sampler.next_2d()` [S1, unofficial].
        *   Scale by $1/(w, h)$ [S1, unofficial].
    *   Compute time sample:
        $$ t = \text{emitter\_idx} / n\_patterns $$
        *   If `sample_time` is true, add `sampler.next_1d()` before dividing [S1, unofficial].
    *   Sample ray from emitter and scale weight by `print_time` ($1/\text{pdf of time sample}$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_rays(self, active_pixels, w, h, emitter_idx, n_patterns, sampler):
    # 1. Prepare checks
    if self.regular_sampling:
        spp = 1
    else:
        spp = self.spp # Assumed from parent context
    
    wavefront_size = len(active_pixels) * spp
    if wavefront_size > 2**32:
        raise ValueError("Wavefront size exceeds 2^32")

    rays = []
    
    # 2. Parse scene (handled in separate method, assumed to set self.target and self.medium)
    # Error if multiple mediums found
    
    for pixel_idx in active_pixels:
        for s in range(spp):
            # Compute pixel coordinates
            row_idx = pixel_idx // w
            pos_y = row_idx
            pos_x = pixel_idx - row_idx * w
            
            # Convert to [0, 1]
            if self.regular_sampling:
                u = (pos_x + 0.5) / w
                v = (pos_y + 0.5) / h
            else:
                u = (pos_x + sampler.next_2d()[0]) / w
                v = (pos_y + sampler.next_2d()[1]) / h
            
            # Compute time sample
            time = emitter_idx / n_patterns
            if self.sample_time:
                time += sampler.next_1d()
                time /= n_patterns # Or scale appropriately based on pdf
            
            # Sample ray from emitter
            # Note: Emitter sampling logic depends on specific projector model
            # Assuming emitter provides origin and direction based on u, v, time
            origin, direction = self.emitter.sample(u, v, time)
            
            # Scale weight
            weight = self.print_time # 1 / print_time if print_time is duration? 
                                     # Source says "scale weight by print_time (1/pdf of time sample)"
                                     # Interpreting source literally: weight *= print_time
            
            rays.append(Ray(origin, direction, weight))
            
    return rays
```

**Edge Cases and Pitfalls:**
*   **Wavefront Overflow:** If `active_size * spp` exceeds $2^{32}$, the method must raise an exception [S1, unofficial].
*   **Regular Sampling Conflict:** Setting `regular_sampling` to true with spp > 1 is logically inconsistent and forces spp to 1 [S1, unofficial].
*   **Multiple Mediums:** The scene must contain exactly one interior medium; otherwise, an error is raised [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
A specialized integrator that filters out rays intersecting the corners of the target build volume to solve discretization artifacts from DMD pixels.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform (projection) of a volume by tracing rays through a purely absorptive medium, calculating absorbed dose or optical density.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot \text{spp})$).
*   **Memory Complexity:** Linear with respect to the number of rays generated for the wavefront.
*   **Strengths:** Provides a unified framework for handling temporal pattern sequencing and spatial sampling in TVAM.
*   **Limitations:** Base class does not perform volume integration itself; it only generates rays. Specialized integrators are required for actual dose calculation or surface interaction.
*   **When to use:** When implementing custom TVAM integrators that require specific ray sampling strategies (regular vs. stochastic) or temporal modulation.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It was developed to address the specific challenges of projecting light patterns through a rotating vial of photopolymer resin, requiring precise control over ray origins, directions, and temporal synchronization with DMD patterns.

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
