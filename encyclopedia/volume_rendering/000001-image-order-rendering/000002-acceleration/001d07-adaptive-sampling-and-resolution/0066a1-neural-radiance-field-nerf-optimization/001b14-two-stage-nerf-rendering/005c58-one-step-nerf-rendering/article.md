# One-Step NeRF Rendering

Also called: NeRF++ setup, OneStepNeRFRenderer

## Overview
One-Step NeRF Rendering is a volume rendering method that eliminates the multi-stage sampling pipeline by using only stratified sampling for both foreground and background components [S1, unofficial]. It solves the computational overhead of importance sampling and proposal networks by treating the background as an outer volume, similar to the NeRF++ approach [S1, unofficial]. It differs from [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by removing the fine-stage refinement and from [Coarse-Fine Sampling](8172-coarse-fine-sampling.md) by using a single uniform depth distribution for all samples [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of rays, foreground NeRF model, foreground spatial volume, optional background NeRF model, optional background spatial volume, and configuration options (sample counts, depth modes, channel scale).
*   **Outputs:** Rendered channels (RGB), distances, transmittance, ray start/end points ($t_0, t_1$), intersection flags, and auxiliary losses.

**Data Structures:**
The method constructs a list of `RayVolumeIntegral` parts. Each part contains:
*   `model`: A callable (wrapped in `partial`) representing the NeRF model (foreground or background).
*   `volume`: The spatial volume object defining the domain.
*   `sampler`: A `StratifiedRaySampler` instance configured with a specific depth mode.
*   `n_samples`: Integer count of samples per ray for this part.

**Numbered Steps:**
1.  **Initialize Options:** Retrieve rendering settings from instance variables or provided options. Key flags include `render_background` (bool), `render_with_direction` (bool), `n_samples` (int), and sampling modes for foreground and background.
2.  **Create Foreground Part:** Instantiate a `RayVolumeIntegral` object for the foreground.
    *   Set `model` to the foreground NeRF model wrapped in a partial function with necessary parameters.
    *   Set `volume` to the foreground spatial volume.
    *   Set `sampler` to a `StratifiedRaySampler` initialized with `depth_mode` from `options.foreground_stratified_depth_sampling_mode` (default: `'linear'`).
    *   Set `n_samples` to `options.n_samples`.
3.  **Create Background Part (Conditional):** If `render_background` is true and an `outer_volume` is provided:
    *   Instantiate a second `RayVolumeIntegral` object for the background.
    *   Set `model` to the background NeRF model wrapped in a partial function.
    *   Set `volume` to the `outer_volume`.
    *   Set `sampler` to a `StratifiedRaySampler` initialized with `depth_mode` from `options.background_stratified_depth_sampling_mode` (default: `'linear'`).
    *   Set `n_samples` to `options.n_samples`.
    *   Append this part to the list of parts.
4.  **Render Rays:** Call the `render_rays` function with the batch of rays, the list of parts, a `void_model` (for rays that do not intersect any volume), and the `render_with_direction` flag.
5.  **Process Results:** Multiply the output channels by `channel_scale` (default: 255.0) to convert from normalized float to standard integer range if required. Return the channels, distances, transmittance, $t_0$, $t_1$, intersection status, and auxiliary losses.

**Formulas:**
The rendering equation for a ray $r$ is computed by summing the contributions of all parts. For a single part with $N$ samples at depths $t_i$ within interval $[t_0, t_1]$:
$$ C(r) = \sum_{i=1}^{N} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) c_i $$
Where:
*   $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance.
*   $\sigma_i$ is the volume density predicted by the model at sample $i$.
*   $c_i$ is the color predicted by the model at sample $i$.
*   $\delta_i = t_{i} - t_{i-1}$ is the distance between samples.
*   The background part is rendered after the foreground, typically handling rays that have passed through the foreground volume or are outside it, depending on the specific composition logic in `render_rays` [S1, unofficial].

**Parameters:**
*   `n_samples`: Number of stratified samples per ray per part. Typical values depend on desired quality vs. speed; not explicitly given in sources, but implied to be a single integer for both stages [S1, unofficial].
*   `void_model`: NeRFModel used for rays that do not intersect the foreground or background volumes.
*   `foreground_model`: NeRFModel for the primary scene.
*   `background_model`: Optional NeRFModel for the background.
*   `volume`: Foreground spatial volume.
*   `outer_volume`: Optional background spatial volume.
*   `foreground_stratified_depth_sampling_mode`: String, default `'linear'`.
*   `background_stratified_depth_sampling_mode`: String, default `'linear'`.
*   `channel_scale`: Float, multiplier for output channels, default 255.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Linear sampling may miss thin structures compared to importance sampling, leading to aliasing or transparency artifacts [S1, unofficial].
*   **Boundaries:** If `outer_volume` is None but `render_background` is True, the background part is skipped, potentially leaving rays unrendered if the foreground volume does not cover the entire ray path [S1, unofficial].
*   **Degenerate Input:** If `n_samples` is 0, no samples are taken, resulting in empty or black renders [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_one_step(rays, options, foreground_model, foreground_volume, 
                    background_model, outer_volume, void_model):
    """
    Renders rays using a single stratified sampling stage for foreground and background.
    """
    # 1. Initialize options
    render_background = options.render_background
    render_with_direction = options.render_with_direction
    n_samples = options.n_samples
    fg_depth_mode = options.foreground_stratified_depth_sampling_mode
    bg_depth_mode = options.background_stratified_depth_sampling_mode
    channel_scale = options.channel_scale

    parts = []

    # 2. Create foreground part
    fg_sampler = StratifiedRaySampler(depth_mode=fg_depth_mode)
    fg_part = RayVolumeIntegral(
        model=partial(foreground_model),
        volume=foreground_volume,
        sampler=fg_sampler,
        n_samples=n_samples
    )
    parts.append(fg_part)

    # 3. Create background part if applicable
    if render_background and outer_volume is not None:
        bg_sampler = StratifiedRaySampler(depth_mode=bg_depth_mode)
        bg_part = RayVolumeIntegral(
            model=partial(background_model),
            volume=outer_volume,
            sampler=bg_sampler,
            n_samples=n_samples
        )
        parts.append(bg_part)

    # 4. Render rays
    results = render_rays(
        rays=rays,
        parts=parts,
        void_model=void_model,
        render_with_direction=render_with_direction
    )

    # 5. Process and return results
    channels = results.output.channels * channel_scale
    return {
        'channels': channels,
        'distances': results.distances,
        'transmittance': results.transmittance,
        't0': results.t0,
        't1': results.t1,
        'intersected': results.intersected,
        'aux_losses': results.aux_losses
    }
```

## Subtopics

### Combined Sampling (Occupancy Grid + Proposal Network)
Uses learned occupancy grids and proposal networks to concentrate samples in occupied space. Choose when high fidelity is needed and computational budget allows for two-stage sampling. [Link](1626-combined-sampling-occupancy-grid-proposal-network.md)

### VF-NeRF Rendering Pipeline
A specific pipeline implementation for View-Frontier NeRF. Choose when working with the VF-NeRF architecture. [Link](3302-vf-nerf-rendering-pipeline.md)

### Coarse-Fine Sampling
The standard two-stage approach using a coarse network for importance sampling and a fine network for refinement. Choose for general high-quality NeRF rendering. [Link](8172-coarse-fine-sampling.md)

### Semantic Loss Computation
Computes losses based on semantic segmentation maps. Choose when training with semantic supervision. [Link](15616-semantic-loss-computation.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of samples per ray and $R$ is the number of rays. Since only one stage is used, it is faster than two-stage methods which require $2N$ samples or adaptive sampling.
*   **Memory Complexity:** $O(N \cdot R)$ for storing sample positions, densities, and colors. Lower than two-stage methods if the fine stage is skipped.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of rays and samples. Does not benefit from adaptive sampling reductions.
*   **Strengths:** Simpler implementation, lower latency per frame, no need for proposal network training or occupancy grid maintenance.
*   **Limitations:** Lower image quality for complex scenes with thin structures or high-frequency details due to uniform/linear sampling. Prone to noise and aliasing compared to importance sampling.
*   **When to use:** Real-time applications, simple scenes, or when computational resources are severely constrained.
*   **When not to use:** High-fidelity offline rendering, scenes with complex geometry or transparency.

## Background
Originated in the context of the gsgen3d project, which integrates NeRF-like rendering with Gaussian Splatting [S1, unofficial]. The method is described as a "NeRF++ setup" because it handles background rendering via an outer volume, a technique popularized by NeRF++ [S1, unofficial]. It is implemented in the `gsgen3d/gsgen` repository [S1, unofficial].

## Sources
[S1] gsgen3d/gsgen, https://github.com/gsgen3d/gsgen, fetched 2026-10-27 (unofficial)
