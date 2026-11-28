# One-Step NeRF Rendering

Also called: NeRF++ setup, OneStepNeRFRenderer

## Overview
One-Step NeRF Rendering is a volume rendering method that uses only stratified sampling without a fine-stage or importance sampling refinement [S1, unofficial]. It supports background rendering via an outer volume, similar to the NeRF++ setup [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of rays, foreground NeRF model, foreground spatial volume, optional background NeRF model, optional background spatial volume, and configuration options (sample counts, depth modes, channel scale) [S1, unofficial].
*   **Outputs:** Rendered channels (RGB), distances, transmittance, ray start/end points ($t_0, t_1$), intersection flags, and auxiliary losses [S1, unofficial].

**Data Structures:**
The method constructs a list of `RayVolumeIntegral` parts [S1, unofficial]. Each part contains:
*   `model`: A callable representing the NeRF model (foreground or background) [S1, unofficial].
*   `volume`: The spatial volume object defining the domain [S1, unofficial].
*   `sampler`: A `StratifiedRaySampler` instance configured with a specific depth mode [S1, unofficial].
*   `n_samples`: Integer count of samples per ray for this part [S1, unofficial].

**Numbered Steps:**
1.  **Initialize Options:** Retrieve rendering settings from instance variables or provided options [S1, unofficial]. Key flags include `render_background` (bool), `render_with_direction` (bool), `n_samples` (int), and sampling modes for foreground and background [S1, unofficial].
2.  **Create Foreground Part:** Instantiate a `RayVolumeIntegral` object for the foreground [S1, unofficial]. Set `model` to the foreground NeRF model wrapped in a partial function with necessary parameters [S1, unofficial]. Set `volume` to the foreground spatial volume [S1, unofficial]. Set `sampler` to a `StratifiedRaySampler` initialized with `depth_mode` from `options.foreground_stratified_depth_sampling_mode` (default: `'linear'`) [S1, unofficial]. Set `n_samples` to `options.n_samples` [S1, unofficial].
3.  **Create Background Part (Conditional):** If `render_background` is true and an `outer_volume` is provided, instantiate a second `RayVolumeIntegral` object for the background [S1, unofficial]. Set `model` to the background NeRF model wrapped in a partial function [S1, unofficial]. Set `volume` to the `outer_volume` [S1, unofficial]. Set `sampler` to a `StratifiedRaySampler` initialized with `depth_mode` from `options.background_stratified_depth_sampling_mode` (default: `'linear'`) [S1, unofficial]. Set `n_samples` to `options.n_samples` [S1, unofficial]. Append this part to the list of parts [S1, unofficial].
4.  **Render Rays:** Call the `render_rays` function with the batch of rays, the list of parts, a `void_model` (for rays that do not intersect any volume), and the `render_with_direction` flag [S1, unofficial].
5.  **Process Results:** Multiply the output channels by `channel_scale` (default: 255.0) to convert from normalized float to standard integer range if required [S1, unofficial]. Return the channels, distances, transmittance, $t_0$, $t_1$, intersection status, and auxiliary losses [S1, unofficial].

**Formulas:**
The rendering equation for a ray $r$ is computed by summing the contributions of all parts [G]. For a single part with $N$ samples at depths $t_i$ within interval $[t_0, t_1]$:
$$ C(r) = \sum_{i=1}^{N} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) c_i $$
Where:
*   $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance [G].
*   $\sigma_i$ is the volume density predicted by the model at sample $i$ [G].
*   $c_i$ is the color predicted by the model at sample $i$ [G].
*   $\delta_i = t_{i} - t_{i-1}$ is the distance between samples [G].

**Parameters:**
*   `n_samples`: Number of stratified samples per ray per part [S1, unofficial].
*   `void_model`: NeRFModel used for rays that do not intersect the foreground or background volumes [S1, unofficial].
*   `foreground_model`: NeRFModel for the primary scene [S1, unofficial].
*   `background_model`: Optional NeRFModel for the background [S1, unofficial].
*   `volume`: Foreground spatial volume [S1, unofficial].
*   `outer_volume`: Optional background spatial volume [S1, unofficial].
*   `foreground_stratified_depth_sampling_mode`: String, default `'linear'` [S1, unofficial].
*   `background_stratified_depth_sampling_mode`: String, default `'linear'` [S1, unofficial].
*   `channel_scale`: Float, multiplier for output channels, default 255.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Dict, Tuple, Callable, Any

def stratified_sample(t0: float, t1: float, n_samples: int, depth_mode: str) -> List[float]:
    """Generate n_samples stratified depths in [t0, t1]."""
    if depth_mode == 'linear':
        step = (t1 - t0) / n_samples
        return [t0 + (i + math.random()) * step for i in range(n_samples)]
    return [t0 + (t1 - t0) * (i + math.random()) / n_samples for i in range(n_samples)]

def query_model(model: Callable, rays_o: List[Tuple[float, float, float]], rays_d: List[Tuple[float, float, float]], depths: List[float]) -> Tuple[List[float], List[Tuple[float, float, float]]]:
    """Query NeRF model for density and color at given depths along rays."""
    return model(rays_o, rays_d, depths)

def render_one_step(rays_o: List[Tuple[float, float, float]], rays_d: List[Tuple[float, float, float]],
                    options: Dict, foreground_model: Callable, foreground_volume: Any,
                    background_model: Callable, outer_volume: Any, void_model: Callable) -> Dict:
    """
    Renders rays using a single stratified sampling stage for foreground and background.
    """
    # 1. Initialize options
    render_background = options.get('render_background', True)
    render_with_direction = options.get('render_with_direction', True)
    n_samples = options.get('n_samples', 64)
    fg_depth_mode = options.get('foreground_stratified_depth_sampling_mode', 'linear')
    bg_depth_mode = options.get('background_stratified_depth_sampling_mode', 'linear')
    channel_scale = options.get('channel_scale', 255.0)

    R = len(rays_o)
    channels = [[0.0, 0.0, 0.0] for _ in range(R)]
    distances = [0.0 for _ in range(R)]
    transmittance = [1.0 for _ in range(R)]
    t0_vals = [0.0 for _ in range(R)]
    t1_vals = [1.0 for _ in range(R)]
    intersected = [False for _ in range(R)]
    aux_losses = []

    # 2. Create foreground part
    fg_sampler = {'depth_mode': fg_depth_mode, 'n_samples': n_samples}
    fg_part = {'model': foreground_model, 'volume': foreground_volume, 'sampler': fg_sampler}
    parts = [fg_part]

    # 3. Create background part if applicable
    if render_background and outer_volume is not None:
        bg_sampler = {'depth_mode': bg_depth_mode, 'n_samples': n_samples}
        bg_part = {'model': background_model, 'volume': outer_volume, 'sampler': bg_sampler}
        parts.append(bg_part)

    # 4. Render rays
    for r in range(R):
        ro = rays_o[r]
        rd = rays_d[r]
        T = 1.0
        C = [0.0, 0.0, 0.0]
        t_prev = t0_vals[r]

        for part in parts:
            depths = stratified_sample(t_prev, t1_vals[r], n_samples, part['sampler']['depth_mode'])
            sigmas, colors = query_model(part['model'], [ro], [rd], depths)
            sigmas = sigmas[0]
            colors = colors[0]

            for i, t in enumerate(depths):
                delta = t - t_prev
                alpha = 1.0 - math.exp(-max(sigmas[i], 1e-10) * delta)
                weight = T * alpha
                C = [C[j] + weight * colors[j] for j in range(3)]
                T *= (1.0 - alpha)
                t_prev = t

            if T < 1e-4:
                break

        if T > 1e-4:
            void_c = void_model(ro, rd)
            C = [C[j] + T * void_c[j] for j in range(3)]
            intersected[r] = True

        channels[r] = C
        distances[r] = t_prev
        transmittance[r] = T

    # 5. Process and return results
    channels = [[c * channel_scale for c in ch] for ch in channels]
    return {
        'channels': channels,
        'distances': distances,
        'transmittance': transmittance,
        't0': t0_vals,
        't1': t1_vals,
        'intersected': intersected,
        'aux_losses': aux_losses
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
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of samples per ray and $R$ is the number of rays [G].
*   **Memory Complexity:** $O(N \cdot R)$ for storing sample positions, densities, and colors [G].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Originated in the context of the gsgen3d project [S1, unofficial]. The method is described as a "NeRF++ setup" because it handles background rendering via an outer volume [S1, unofficial]. It is implemented in the `gsgen3d/gsgen` repository [S1, unofficial].

## Sources
[S1] gsgen3d/gsgen, https://github.com/gsgen3d/gsgen, fetched 2026-10-27 (unofficial)
