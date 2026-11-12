# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric integrators in Tomographic Volumetric Additive Manufacturing (TVAM) that handles projector ray sampling, time sampling, and regular versus stochastic sampling modes [S1, unofficial]. It solves the problem of generating correct ray samples for the projector emitter in a TVAM simulation, distinguishing between deterministic pixel-center sampling for regular modes and stochastic sampling for Monte Carlo integration [S1, unofficial]. It sets itself apart from sibling integrators like the [Radon Integrator](7470-radon-integrator.md) by explicitly managing the temporal dimension of the DMD pattern sequence and the specific geometry of the projector emitter, rather than just performing a static projection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene configuration containing a target shape (identified by `target_id`) and an interior medium; projector emitter parameters; sampling mode flags (`regular_sampling`, `sample_time`).
*   **Outputs:** A set of rays with positions, directions, weights, and time samples suitable for tracing through the TVAM volume.

**Data Structures:**
The integrator maintains state for the current pixel and time samples. Key parameters include:
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample.
*   `print_time` (float, default `1.0`): Used to scale the ray weight by $1/\text{print\_time}$.
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization.
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces.
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset.
*   `K` (int, default `2`): Exponent for physical model evaluation [S1, unofficial].
*   `M` (int, default `4`): Exponent for sparsity evaluation [S1, unofficial].
*   `tp` (float, default `0.1`): Threshold parameter for polymerization [S1, unofficial].
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization [S1, unofficial].
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibitor consumption [S1, unofficial].
*   `weight_object` (float, default `1`): Weight for object region losses [S1, unofficial].
*   `weight_void` (float, default `1`): Weight for void region losses [S1, unofficial].
*   `weight_limit` (float, default `1`): Weight for limit region losses [S1, unofficial].
*   `weight_sparsity` (float, default `0`): Weight for sparsity losses [S1, unofficial].

**Numbered Steps:**

1.  **Prepare:**
    *   If `regular_sampling` is true, force the number of samples per pixel (spp) to 1, because shooting from the pixel center makes spp > 1 nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].

2.  **Parse Scene:**
    *   Iterate over scene shapes to find the target shape (identified by `target_id`) and the interior medium [S1, unofficial].
    *   Raise an error if more than one medium is found [S1, unofficial].

3.  **Sample Rays:**
    *   Compute the discrete sample position index `idx` by repeating `active_pixels` spp times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ \text{pos.y} = \lfloor \text{pixel\_idx} / w \rfloor $$
        $$ \text{pos.x} = \text{pixel\_idx} - \text{row\_idx} \times w $$
        where $w$ is the width of the projector resolution and $\text{row\_idx} = \text{pos.y}$ [S1, unofficial].
    *   Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is true, add a 0.5 offset to both coordinates.
        *   Else, add a stochastic offset from `sampler.next_2d()` to both coordinates.
        *   Scale by $1/w$ and $1/h$ respectively [S1, unofficial].
    *   Compute the time sample:
        $$ \text{time} = \text{emitter\_idx} / n\_patterns $$
        If `sample_time` is true, add a stochastic offset from `sampler.next_1d()` before dividing by $n\_patterns$ [S1, unofficial].
    *   Sample the ray from the emitter using the computed position and time.
    *   Scale the ray weight by `print_time` (specifically, multiply by $1/\text{print\_time}$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare(self):
    if self.regular_sampling:
        self.spp = 1
    if self.active_size * self.spp > 2**32:
        raise ValueError("Wavefront size exceeds 2^32")

def parse_scene(self, scene):
    target = None
    medium = None
    for shape in scene.shapes:
        if shape.id == self.target_id:
            target = shape
        if shape.has_medium:
            if medium is not None:
                raise ValueError("More than one medium found")
            medium = shape.medium
    if target is None or medium is None:
        raise ValueError("Target or medium not found")
    return target, medium

def sample_rays(self, sampler, emitter, active_pixels, w, h, n_patterns):
    # idx is computed by repeating active_pixels spp times
    # This is a conceptual representation of the wavefront generation
    rays = []
    for pixel_idx in range(len(active_pixels) * self.spp):
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
        emitter_idx = pixel_idx % n_patterns # Assuming emitter_idx cycles or is mapped
        time = emitter_idx / n_patterns
        if self.sample_time:
            time = (emitter_idx + sampler.next_1d()) / n_patterns
            
        # Sample ray from emitter
        ray = emitter.sample(u, v, time)
        
        # Scale weight
        ray.weight *= 1.0 / self.print_time
        
        rays.append(ray)
    return rays
```

**Edge Cases and Pitfalls:**
*   **Wavefront Overflow:** If `active_size * spp` exceeds $2^{32}$, the method raises an exception. Implementers must ensure spp is reduced or active pixels are subsampled in such cases [S1, unofficial].
*   **Regular Sampling vs. SPP:** Setting `regular_sampling` to true forces `spp` to 1. Attempting to use `spp > 1` with regular sampling is nonsensical and should be handled by the `prepare` step [S1, unofficial].
*   **Multiple Mediums:** The scene parser raises an error if more than one medium is found. The scene must be strictly defined with a single interior medium for the TVAM volume [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized for filtering rays intersecting build volume corners to avoid discretization artifacts.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform by tracing rays through a purely absorptive medium.

### [HoloVAM](13314-holovam.md)
Uses holographic phase modulation instead of amplitude modulation for higher light efficiency.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot \text{spp})$), where $N$ is the number of active pixels. The time sampling adds a constant overhead per ray.
*   **Memory Complexity:** Linear with respect to the wavefront size ($O(\text{active\_size} \cdot \text{spp})$). The $2^{32}$ limit constrains the maximum memory footprint for the ray batch.
*   **Strengths:** Provides a flexible base for TVAM simulations, supporting both deterministic and stochastic sampling. Explicit handling of time and projector geometry allows for accurate simulation of DMD pattern sequences.
*   **Limitations:** The base class does not include the physical polymerization model itself; it only handles ray sampling. The physical model parameters (K, M, tp, etc.) are defined but not integrated into the ray tracing loop in this base class [S1, unofficial].
*   **When to Use:** When implementing a TVAM integrator that requires precise control over projector ray sampling and time sequencing.
*   **When Not to Use:** When a simpler static projection (like Radon) or a specialized corner-filtering approach is sufficient.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It was developed to simulate the light propagation in TVAM processes, where a target 3D shape is polymerized by rotating DMD patterns around a build volume. The integrator serves as the foundation for more specialized integrators like the CornerIntegrator and Radon Integrator, which build upon its sampling capabilities [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
