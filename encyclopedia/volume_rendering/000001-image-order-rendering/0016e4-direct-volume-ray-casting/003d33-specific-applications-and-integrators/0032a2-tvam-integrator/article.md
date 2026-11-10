# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric integrators in Tomographic Volumetric Additive Manufacturing (TVAM) that handles projector ray sampling, time sampling, and regular vs. stochastic sampling modes [S1, unofficial]. It solves the problem of correctly mapping discrete projector pixels and temporal patterns to continuous ray origins and weights for inverse rendering optimization. It sets itself apart from sibling integrators by focusing on the specific temporal and geometric constraints of DMD-based projection (e.g., `regular_sampling` for deterministic pixel centers) rather than complex lighting models or occlusion shading.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene configuration containing a target shape (identified by `target_id`) and an interior medium; projector parameters; sampling mode flags (`regular_sampling`, `sample_time`).
*   **Outputs:** Ray packets (origin, direction, weight, time) for the integrator loop.

**Data Structures:**
*   **Scene:** Contains shapes. The integrator iterates to find the shape matching `target_id` and the single interior medium.
*   **Ray Packet:** Contains `origin`, `direction`, `weight`, `time`, and `pixel_idx`.

**Parameters:**
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample.
*   `print_time` (float, default `1.0`): Used to scale the ray weight by `1/print_time`.
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization.
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces.
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset.
*   `K` (int, default `2`): Exponent for physical model evaluation (not used in base sampling, but part of the integrator's context).
*   `M` (int, default `4`): Exponent for sparsity evaluation.
*   `tp` (float, default `0.1`): Threshold parameter for polymerization.
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization.
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibition.
*   `weight_object` (float, default `1`): Weight for object region loss.
*   `weight_void` (float, default `1`): Weight for void region loss.
*   `weight_limit` (float, default `1`): Weight for limit region loss.
*   `weight_sparsity` (float, default `0`): Weight for sparsity loss.

**Steps:**
1.  **Prepare:**
    *   If `regular_sampling` is true, force `spp` (samples per pixel) to 1, because shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (`active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].
2.  **Parse Scene:**
    *   Iterate over scene shapes to find the target shape (by `target_id`) and the interior medium [S1, unofficial].
    *   Raise an error if more than one medium is found [S1, unofficial].
3.  **Sample Rays:**
    *   Compute discrete sample position `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    *   Compute pixel coordinates:
        $$ pos.y = \lfloor pixel\_idx / w \rfloor $$
        $$ pos.x = pixel\_idx - pos.y \times w $$
        where $w$ is the image width [S1, unofficial].
    *   Convert pixel position to floating point $[0, 1]$ range:
        *   If `regular_sampling` is true: $u = pos.x + 0.5$, $v = pos.y + 0.5$.
        *   Else: $u = pos.x + sampler.next\_2d().x$, $v = pos.y + sampler.next\_2d().y$.
        *   Scale by resolution: $u = u / w$, $v = v / h$ [S1, unofficial].
    *   Compute time sample:
        $$ t_{base} = emitter\_idx / n\_patterns $$
        *   If `sample_time` is true: $t = t_{base} + sampler.next\_1d()$.
        *   Else: $t = t_{base}$.
        *   Normalize $t$ to $[0, 1]$ if necessary (source implies division by $n\_patterns$ handles the pattern index, but stochastic offset requires normalization to the pattern duration) [S1, unofficial].
    *   Sample ray from emitter using $(u, v, t)$ [S1, unofficial].
    *   Scale ray weight by $1 / print\_time$ (or $1 / pdf$ of time sample if stochastic) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare(self):
    if self.regular_sampling:
        self.spp = 1
    if self.active_size * self.spp > 2**32:
        raise Exception("Wavefront size exceeds 2^32")

def parse_scene(self, scene):
    target_shape = None
    medium = None
    for shape in scene.shapes:
        if shape.id == self.target_id:
            target_shape = shape
        if shape.medium is not None:
            if medium is not None:
                raise Exception("More than one medium found")
            medium = shape.medium
    if target_shape is None or medium is None:
        raise Exception("Target shape or medium not found")
    return target_shape, medium

def sample_rays(self, active_pixels, w, h, emitter, sampler, n_patterns):
    rays = []
    # idx repeats active_pixels spp times
    for i in range(len(active_pixels) * self.spp):
        pixel_idx = i % len(active_pixels)
        
        # Compute pixel coordinates
        row_idx = pixel_idx // w
        col_idx = pixel_idx - row_idx * w
        
        # Convert to [0, 1]
        if self.regular_sampling:
            u = (col_idx + 0.5) / w
            v = (row_idx + 0.5) / h
        else:
            u = (col_idx + sampler.next_2d().x) / w
            v = (row_idx + sampler.next_2d().y) / h
            
        # Compute time
        t_base = emitter.idx / n_patterns
        if self.sample_time:
            t = t_base + sampler.next_1d()
        else:
            t = t_base
            
        # Sample ray from emitter
        ray = emitter.sample(u, v, t)
        
        # Scale weight
        ray.weight *= 1.0 / self.print_time
        
        rays.append(ray)
    return rays
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point errors in pixel coordinate conversion can cause off-by-one errors at boundaries.
*   **Boundaries:** `regular_sampling` forces `spp=1`; attempting to use `spp > 1` with `regular_sampling` is invalid [S1, unofficial].
*   **Degenerate Input:** If no medium or target shape is found, the integrator fails [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized for filtering rays intersecting build volume corners to avoid DMD discretization artifacts.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform by tracing rays through a purely absorptive medium.

## Operating characteristics
*   **Time Complexity:** Linear in the number of active pixels and samples per pixel ($O(N \cdot SPP)$).
*   **Memory Complexity:** Linear in the number of rays generated for the wavefront.
*   **Strengths:** Deterministic sampling (`regular_sampling`) is efficient for inverse rendering optimization where noise reduction is critical.
*   **Limitations:** Base class does not implement the physical loss function evaluation; it only handles ray generation. The physical model evaluation (`eval`, `eval_in`, `eval_out`) is defined in the source but not explicitly part of the "integrator" ray sampling loop in the provided notes, though it is part of the `drtvam` package context [S1, unofficial].
*   **When to use:** When performing inverse rendering for TVAM where precise control over projector pixel sampling and temporal patterns is required.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It is part of the Mitsuba 3 renderer ecosystem, leveraging DrJit for differentiable programming.

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
