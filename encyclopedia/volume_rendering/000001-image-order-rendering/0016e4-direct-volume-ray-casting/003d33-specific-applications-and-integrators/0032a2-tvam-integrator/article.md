# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a specialized volumetric integrator for Tomographic Volumetric Additive Manufacturing (TVAM) that serves as the base class for domain-specific ray casting methods. It solves the problem of accurately simulating light transport and polymerization dose accumulation in a resin vat by handling projector ray sampling, time sampling, and the distinction between regular and stochastic sampling modes. Unlike standard volume rendering integrators that compute color, this method computes physical quantities (such as polymerization dose) and supports complex loss functions for inverse design. It is distinguished from its siblings, such as the [Radon Integrator](7470-radon-integrator.md) and [CornerIntegrator](7445-cornerintegrator.md), by providing the foundational infrastructure for ray generation and scene parsing, which these subclasses extend to handle specific boundary conditions or projection transforms.

## How it works
The integrator operates within the direct volume ray casting framework [Direct volume ray casting](19-direct-volume-ray-casting.md). It manages the lifecycle of rays from the projector (emitter) through the resin medium to the sensor or build volume.

### Inputs and Outputs
*   **Inputs**: Scene description containing a target shape (by ID), an interior medium (resin), and projector emitters. Configuration parameters for sampling modes and physical weights.
*   **Outputs**: Accumulated physical quantities (e.g., polymerization dose, inhibitor concentration) and sparsity metrics per ray.

### Data Structures
The integrator maintains state for active pixels and rays.
*   `active_pixels`: List of pixel indices to be sampled.
*   `spp`: Samples per pixel.
*   `wavefront_size`: Total number of rays to process, calculated as `active_size * spp`.
*   `pos`: Floating-point pixel coordinates in $[0, 1]$ range.
*   `time`: Normalized time sample in $[0, 1]$.

### Parameters
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample.
*   `print_time` (float, default `1.0`): Used to scale the ray weight by $1/\text{print\_time}$.
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization.
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces.
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset.
*   `K` (int, default `2`): Exponent for the physical model evaluation.
*   `M` (int, default `4`): Exponent for sparsity evaluation.
*   `tp` (float, default `0.1`): Threshold parameter for polymerization.
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization.
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibitor.
*   `weight_object` (float, default `1`): Weight for object region evaluation.
*   `weight_void` (float, default `1`): Weight for void region evaluation.
*   `weight_limit` (float, default `1`): Weight for limit evaluation.
*   `weight_sparsity` (float, default `0`): Weight for sparsity evaluation.

### Algorithm Steps

1.  **Prepare**:
    *   If `regular_sampling` is true, force `spp` to 1. Shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    *   Calculate `wavefront_size = active_size * spp`.
    *   If `wavefront_size > 2^{32}`, raise an exception [S1, unofficial].

2.  **Parse Scene**:
    *   Iterate over scene shapes to find the target shape identified by `target_id`.
    *   Identify the interior medium (resin).
    *   Raise an error if more than one medium is found [S1, unofficial].

3.  **Sample Rays**:
    *   Compute discrete sample position `idx` by repeating `active_pixels` `spp` times.
    *   Compute pixel coordinates:
        $$ \text{pos}.y = \lfloor \text{pixel\_idx} / w \rfloor $$
        $$ \text{pos}.x = \text{pixel\_idx} - \text{row\_idx} \times w $$
        where $w$ is the image width [S1, unofficial].
    *   Convert pixel position to floating point $[0, 1]$ range:
        *   If `regular_sampling`: $\text{pos} += 0.5$.
        *   Else: $\text{pos} += \text{sampler.next\_2d()}$.
        *   Scale by $1/w$ and $1/h$.
    *   Compute time sample:
        $$ t = \text{emitter\_idx} / n\_patterns $$
        *   If `sample_time` is true: $t += \text{sampler.next\_1d()}$.
        *   Normalize $t$ to $[0, 1]$.
    *   Sample ray from emitter.
    *   Scale ray weight by $1/\text{print\_time}$ (which is $1/\text{pdf of time sample}$ if stochastic) [S1, unofficial].

4.  **Evaluate Physical Model** (Extension Point):
    *   The base class defines the interface for evaluating physical quantities. Subclasses implement `eval()` which typically calls:
        $$ \text{eval\_in}(x_{\text{poly}}, x_{\text{inh}}) = w_{\text{obj}} \cdot \text{ReLU}(t_p - x_{\text{poly}} + x_{\text{inh}})^K + w_{\text{limit}} \cdot \text{ReLU}(x_{\text{poly}} - t_{\text{op}})^K $$
        $$ \text{eval\_out}(x_{\text{poly}}, x_{\text{inh}}) = w_{\text{void}} \cdot \text{ReLU}(t_{\text{inh}} - x_{\text{inh}} + x_{\text{poly}})^K $$
        $$ \text{eval\_sparsity}(\text{patterns}) = |\text{patterns}|^M \cdot w_{\text{sparsity}} $$
        where $x_{\text{poly}}$ is polymerization dose, $x_{\text{inh}}$ is inhibitor concentration, and $\text{ReLU}(x) = \max(0, x)$ [S1, unofficial].
    *   The final output selects between `eval_in` and `eval_out` based on whether the ray is inside the target mask.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare(self):
    if self.regular_sampling:
        self.spp = 1
    self.wavefront_size = self.active_size * self.spp
    if self.wavefront_size > 2**32:
        raise Exception("Wavefront size exceeds 2^32")

def parse_scene(self, scene):
    target_shape = None
    medium = None
    for shape in scene.shapes:
        if shape.id == self.target_id:
            target_shape = shape
        if shape.has_medium():
            if medium is not None:
                raise Exception("More than one medium found")
            medium = shape.medium
    if target_shape is None:
        raise Exception(f"Target shape {self.target_id} not found")
    return target_shape, medium

def sample_rays(self, sampler, active_pixels, w, h, n_patterns, emitter_idx):
    rays = []
    for _ in range(self.spp):
        for pixel_idx in active_pixels:
            # Compute pixel coordinates
            row_idx = pixel_idx // w
            pos_y = row_idx
            pos_x = pixel_idx - row_idx * w
            
            # Convert to [0, 1]
            if self.regular_sampling:
                pos_x += 0.5
                pos_y += 0.5
            else:
                offset = sampler.next_2d()
                pos_x += offset[0]
                pos_y += offset[1]
            
            pos_x /= w
            pos_y /= h
            
            # Compute time sample
            time = emitter_idx / n_patterns
            if self.sample_time:
                time += sampler.next_1d()
            time /= n_patterns # Normalize if needed, or assume emitter_idx handles distribution
            
            # Sample ray from emitter
            ray = self.emitter.sample_ray(pos_x, pos_y, time)
            
            # Scale weight
            ray.weight *= 1.0 / self.print_time
            
            rays.append(ray)
    return rays
```

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
A specialized integrator that filters out rays intersecting the corners of the target build volume to solve discretization artifacts from DMD pixels.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform (projection) of a volume by tracing rays through a purely absorptive medium, serving as a specialized TVAM integrator for absorbed dose calculation.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot \text{spp})$).
*   **Memory Complexity**: Linear with respect to the wavefront size ($O(N \cdot \text{spp})$) for storing ray states.
*   **Strengths**: Provides a robust foundation for inverse rendering in TVAM by handling complex sampling strategies (regular vs. stochastic) and integrating physical models for polymerization.
*   **Limitations**: The base class does not implement specific physical interactions; it relies on subclasses for domain-specific logic. Precision issues may arise if `wavefront_size` approaches $2^{32}$.
*   **When to use**: When implementing a new TVAM-specific integrator that requires custom ray sampling or physical evaluation logic.
*   **When not to use**: For standard color volume rendering or when no specific TVAM physics are required.

## Background
The TVAM Integrator originates from the Dr. TVAM project, an inverse rendering framework for Tomographic Volumetric Additive Manufacturing. It was developed to enable the optimization of projector patterns for 3D printing in resin. The method is part of the broader field of computational manufacturing, where optical simulation is used to predict and control the curing of photopolymer resins.

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
