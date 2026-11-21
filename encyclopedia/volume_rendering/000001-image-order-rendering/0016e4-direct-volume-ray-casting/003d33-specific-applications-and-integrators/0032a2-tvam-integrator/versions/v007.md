# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric integrators designed for Tomographic Volumetric Additive Manufacturing (TVAM) [S1, unofficial]. It solves the problem of simulating light transport from projector emitters through a photosensitive resin volume to predict polymerization outcomes [S1, unofficial]. It sets itself apart from general-purpose volume renderers by explicitly handling projector ray sampling, time sampling for patterned light, and specific physical models for polymerization and inhibition [S1, unofficial]. Unlike the [Radon Integrator](7470-radon-integrator.md) which computes simple attenuation, or the [CornerIntegrator](7445-cornerintegrator.md) which masks boundary artifacts, the base TVAM Integrator provides the foundational ray generation and sampling infrastructure for these specialized variants [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene description containing a target shape (identified by `target_id`), an interior medium (resin), and one or more projector emitters. Configuration parameters including `sample_time`, `print_time`, `target_id`, `transmission_only`, `regular_sampling`, and physical model weights (`K`, `M`, `tp`, `top`, `tinhibitor`, `weight_object`, `weight_void`, `weight_limit`, `weight_sparsity`) [S1, unofficial].
*   **Outputs:** A set of rays sampled from the projector emitters, ready for traversal through the volume [S1, unofficial].

**Data Structures:**
*   **Scene:** Contains shapes. The integrator iterates to find the shape with ID matching `target_id` and the single interior medium [S1, unofficial].
*   **Ray Buffer:** Stores active pixels, pixel indices, and computed ray origins/directions [S1, unofficial].

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample [S1, unofficial].
*   `print_time` (float, default `1.0`): Used to scale the ray weight by `1/print_time` [S1, unofficial].
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization [S1, unofficial].
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces [S1, unofficial].
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset [S1, unofficial].
*   `K` (int, default `2`): Exponent for the physical model [S1, unofficial].
*   `M` (int, default `4`): Exponent for sparsity calculation [S1, unofficial].
*   `tp` (float, default `0.1`): Polymerization threshold parameter [S1, unofficial].
*   `top` (float, default `0.2`): Over-polymerization threshold parameter [S1, unofficial].
*   `tinhibitor` (float, default `0.2`): Inhibitor threshold parameter [S1, unofficial].
*   `weight_object` (float, default `1`): Weight for object region evaluation [S1, unofficial].
*   `weight_void` (float, default `1`): Weight for void region evaluation [S1, unofficial].
*   `weight_limit` (float, default `1`): Weight for limit evaluation [S1, unofficial].
*   `weight_sparsity` (float, default `0`): Weight for sparsity evaluation [S1, unofficial].

**Steps:**

1.  **Parse Scene (`parse_scene`):**
    *   Iterate over all shapes in the scene [S1, unofficial].
    *   Identify the shape whose ID matches `target_id` [S1, unofficial].
    *   Identify the interior medium [S1, unofficial].
    *   If more than one interior medium is found, raise an error [S1, unofficial].

2.  **Prepare (`prepare`):**
    *   If `regular_sampling` is `True`, force `spp` (samples per pixel) to 1, because shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].

3.  **Sample Rays (`sample_rays`):**
    *   Compute discrete sample position `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ pos.y = \lfloor pixel\_idx / w \rfloor $$
        $$ pos.x = pixel\_idx - row\_idx \times w $$
        where $w$ is the width of the emitter/sensor resolution [S1, unofficial].
    *   Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is `True`, add 0.5 offset to both coordinates [S1, unofficial].
        *   Else, add `sampler.next_2d()` (stochastic offset) to both coordinates [S1, unofficial].
        *   Scale by $1/w$ and $1/h$ respectively [S1, unofficial].
    *   Compute time sample:
        $$ time = emitter\_idx / n\_patterns $$
        *   If `sample_time` is `True`, add `sampler.next_1d()` to the result before dividing by `n_patterns` (or effectively adding a fractional offset to the time index) [S1, unofficial].
    *   Sample ray from the emitter using the computed position and time [S1, unofficial].
    *   Scale the ray weight by `print_time` (specifically, multiply by `1/print_time` as per the parameter description, though the code snippet implies scaling by `print_time` or its inverse depending on context; the note says "scale weight by print_time (1/pdf of time sample)" which implies the weight adjustment compensates for the time sampling PDF, often resulting in a factor of `1` if uniform, but explicitly scaled by `print_time` in the config context) [S1, unofficial].

**Physical Model Evaluation (Internal Helper):**
The integrator supports a physical model for evaluating polymerization states, defined by:
*   `eval_in(x_polymerization, x_inhibitor)`:
    $$ weight\_object \cdot \max(0, tp - x\_polymerization + x\_inhibitor)^K + weight\_limit \cdot \max(0, x\_polymerization - top)^K $$
*   `eval_out(x_polymerization, x_inhibitor)`:
    $$ weight\_void \cdot \max(0, tinhibitor - x\_inhibitor + x\_polymerization)^K $$
*   `eval_sparsity(patterns)`:
    $$ weight\_sparsity \cdot |patterns|^M $$
*   `eval(x_polymerization, target, patterns, x_inhibitor)`:
    Returns a tuple based on whether `target > 0`:
    $$ (\text{eval\_in if target} > 0 \text{ else eval\_out}, \text{eval\_sparsity}) $$
    [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **SPP Conflict:** Using `regular_sampling` with `spp > 1` is invalid and handled by forcing `spp=1` [S1, unofficial].
*   **Overflow:** Large wavefront sizes exceeding $2^{32}$ cause an exception [S1, unofficial].
*   **Multiple Mediums:** The scene must contain exactly one interior medium; otherwise, an error is raised [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
A specialized integrator that filters out rays intersecting the corners of the target build volume to solve discretization artifacts near DMD pixel boundaries. It masks out hits near object corners based on a distance threshold.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform (projection) of a volume by tracing rays through a purely absorptive medium, calculating absorbed dose or optical density. It builds on the TVAM Integrator base class.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot spp)$), plus the cost of ray traversal through the volume [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the wavefront size ($O(active\_size \cdot spp)$) for storing ray data [S1, unofficial].
*   **Strengths:** Provides a unified framework for TVAM simulation, handling projector geometry, time-varying patterns, and specific physical models for resin polymerization [S1, unofficial].
*   **Limitations:** The base class does not handle specific boundary artifacts (handled by CornerIntegrator) or simple attenuation-only projections (handled by Radon Integrator). It requires careful configuration of physical model parameters for accurate simulation [S1, unofficial].
*   **When to use:** When simulating the light distribution in TVAM processes where projector patterns, time sampling, and resin physics are relevant.
*   **When not to use:** For simple projection visualization (use Radon Integrator) or when corner artifacts are the primary concern without needing the full physical model (use CornerIntegrator).

## Background
The TVAM Integrator originates from the Dr. TVAM project, which focuses on inverse rendering for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It was developed to provide a robust simulation environment for optimizing TVAM processes, integrating with the Mitsuba renderer and DrJIT for differentiable rendering capabilities [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
