# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric additive manufacturing (TVAM) simulators that handles projector ray sampling, time sampling, and regular versus stochastic sampling modes [S1, unofficial]. It serves as the foundation for specialized integrators like the Radon Integrator and CornerIntegrator, which compute optical density or surface-aware discretization for tomographic printing [S1, unofficial]. Unlike general-purpose volume rendering methods that focus on visual fidelity, this method focuses on the physical accumulation of polymerization dose and inhibitor concentration, often incorporating specific loss functions for inverse design [S1, unofficial].

## How it works
The method simulates the light transport from a projector through a photosensitive resin vat to a rotating target object. It operates in two main phases: scene preparation and ray sampling.

### Inputs and Outputs
**Inputs:**
*   Scene configuration containing a target shape (identified by `target_id`) and an interior medium (resin) [S1, unofficial].
*   Emitter parameters defining the projector's position and intensity [S1, unofficial].
*   Sampling parameters: `regular_sampling` (bool), `sample_time` (bool), `spp` (samples per pixel) [S1, unofficial].
*   Physical model parameters: `K`, `M`, `tp`, `top`, `tinhibitor`, `weight_object`, `weight_void`, `weight_limit`, `weight_sparsity` [S1, unofficial].

**Outputs:**
*   Ray origins, directions, and weights for each sample [S1, unofficial].
*   Accumulated polymerization and inhibitor values at target locations (computed in derived classes or post-processing) [S1, unofficial].

### Data Structures
The integrator maintains state for active pixels and their corresponding ray samples.
*   `active_pixels`: List of pixel indices to be sampled.
*   `pixel_idx`: Integer index mapping to pixel coordinates.
*   `pos`: Floating-point pixel coordinates in the range $[0, 1]$ [S1, unofficial].
*   `time`: Normalized time sample in $[0, 1]$ [S1, unofficial].

### Parameters
*   `sample_time` (bool, default `False`): If true, adds a stochastic offset to the time sample [S1, unofficial].
*   `print_time` (float, default `1.0`): Used to scale the ray weight by $1/\text{print\_time}$ [S1, unofficial].
*   `target_id` (str, default `'target'`): The ID of the target shape used for surface-aware discretization [S1, unofficial].
*   `transmission_only` (bool, default `True`): If true, only samples the transmission component of the BSDF at interfaces [S1, unofficial].
*   `regular_sampling` (bool, default `False`): If true, samples from the center of the pixel (0.5 offset) instead of using a random offset [S1, unofficial].
*   `K` (int, default `2`): Exponent for the polymerization/inhibitor loss functions [S1, unofficial].
*   `M` (int, default `4`): Exponent for sparsity calculation [S1, unofficial].
*   `tp` (float, default `0.1`): Threshold parameter for polymerization [S1, unofficial].
*   `top` (float, default `0.2`): Threshold parameter for over-polymerization [S1, unofficial].
*   `tinhibitor` (float, default `0.2`): Threshold parameter for inhibitor [S1, unofficial].
*   `weight_object` (float, default `1`): Weight for object region losses [S1, unofficial].
*   `weight_void` (float, default `1`): Weight for void region losses [S1, unofficial].
*   `weight_limit` (float, default `1`): Weight for limit region losses [S1, unofficial].
*   `weight_sparsity` (float, default `0`): Weight for sparsity penalty [S1, unofficial].

### Steps

1.  **Prepare**:
    *   If `regular_sampling` is true, force `spp` to 1, as shooting from the pixel center makes `spp > 1` nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].

2.  **Sample Rays**:
    *   Compute discrete sample position `idx` by repeating `active_pixels` `spp` times [S1, unofficial].
    *   Compute pixel coordinates `pos.y` and `pos.x` from `pixel_idx`:
        $$ \text{row\_idx} = \lfloor \text{pixel\_idx} / w \rfloor $$
        $$ \text{pos}.y = \text{row\_idx} $$
        $$ \text{pos}.x = \text{pixel\_idx} - \text{row\_idx} \times w $$
        where $w$ is the image width [S1, unofficial].
    *   Convert pixel position to floating point $[0, 1]$ range:
        *   If `regular_sampling` is true, add 0.5 offset to `pos`.
        *   Else, add `sampler.next_2d()` (uniform random in $[0, 1]$) to `pos` [S1, unofficial].
        *   Scale by $1/w$ for x and $1/h$ for y [S1, unofficial].
    *   Compute time sample:
        $$ \text{time} = \text{emitter\_idx} / n\_patterns $$
        *   If `sample_time` is true, add `sampler.next_1d()` (uniform random in $[0, 1]$) to the numerator before dividing [S1, unofficial].
    *   Sample ray from emitter using `pos` and `time` [S1, unofficial].
    *   Scale ray weight by `print_time` (specifically, multiply by $1/\text{print\_time}$) [S1, unofficial].

3.  **Parse Scene**:
    *   Iterate over scene shapes to find the target shape (by `target_id`) and the interior medium [S1, unofficial].
    *   Raise an error if more than one medium is found [S1, unofficial].

### Physical Evaluation (Derived/Associated)
While the base class handles sampling, the associated physical model evaluates the dose using the parameters defined above. The evaluation function `eval` selects between `eval_in` and `eval_out` based on a binary target mask [S1, unofficial]:

$$
\text{eval}(x_{poly}, x_{inh}) = \begin{cases} 
\text{eval\_in}(x_{poly}, x_{inh}) & \text{if inside target} \\
\text{eval\_out}(x_{poly}, x_{inh}) & \text{if outside target}
\end{cases}
$$

Where:
$$ \text{eval\_in}(x_{poly}, x_{inh}) = w_{obj} \cdot \text{relu}(t_p - x_{poly} + x_{inh})^K + w_{lim} \cdot \text{relu}(x_{poly} - t_{op})^K $$
$$ \text{eval\_out}(x_{poly}, x_{inh}) = w_{void} \cdot \text{relu}(t_{inh} - x_{inh} + x_{poly})^K $$
$$ \text{eval\_sparsity}(\text{patterns}) = |\text{patterns}|^M \cdot w_{sparsity} $$

Here, $\text{relu}(x) = \max(0, x)$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Wavefront Overflow**: If the number of active pixels multiplied by samples per pixel exceeds $2^{32}$, the simulation must fail or reduce resolution [S1, unofficial].
*   **Regular Sampling Conflict**: Using `spp > 1` with `regular_sampling=True` is logically invalid and must be corrected to `spp=1` [S1, unofficial].
*   **Multiple Mediums**: The scene parser strictly enforces a single interior medium; multiple mediums cause an error [S1, unofficial].

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized for filtering rays intersecting build volume corners to avoid discretization artifacts from DMD pixels.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform (projection) of the volume by tracing rays through a purely absorptive medium.

## Operating characteristics
*   **Complexity**: Linear with respect to the number of active pixels and samples per pixel ($O(N \cdot \text{spp})$) [S1, unofficial].
*   **Memory**: Depends on the wavefront size; constrained by $2^{32}$ total samples to avoid overflow [S1, unofficial].
*   **Performance**: Efficient for inverse design workflows due to differentiable rendering capabilities (using Dr.Jit) [S1, unofficial].
*   **Limitations**: Base class does not perform volume integration itself; it provides the ray sampling infrastructure for derived classes [S1, unofficial].

## Background
The TVAM Integrator originates from the Dr. TVAM project, which focuses on inverse rendering for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It is implemented within the Mitsuba 2 renderer using the Dr.Jit differentiable programming library, allowing for gradient-based optimization of projector patterns [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
