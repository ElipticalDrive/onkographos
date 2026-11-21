# 3DGS Radiance Caching

Also called: GSCache Radiance Caching, Radiance caching with 3DGS

## Overview
3DGS Radiance Caching is a real-time variance reduction technique for volume path tracing that uses 3D Gaussian Splatting (3DGS) primitives to store and query cached radiance values [S1, S2, unofficial]. It reduces noise by probabilistically accepting cached samples based on throughput luminance and a scaling parameter, optimizing exponentially subsampled Gaussians from noisy path samples to achieve real-time frame rates [S1, S2, unofficial]. Unlike neural radiance caching which requires large memory footprints, this method leverages the compact representation of 3DGS to store attenuated radiance at multiple path lengths, offering a significantly smaller memory footprint [S2, unofficial].

## How it works
The method integrates a caching mechanism into the path tracing loop, using 3DGS primitives to approximate the radiance field. It operates by querying a cache of radiance samples, accepting or rejecting them based on a probability derived from the current path's throughput, and updating state variables to maintain unbiased expectations (with specific bias trade-offs noted in sources).

### Inputs and Outputs
**Inputs:**
- Current path throughput vector $L$ (radiance).
- Current path throughput scalar (weight) $w$ (often implicit in the throughput vector scaling).
- Pixel coordinates $(u, v)$.
- Cache state: `cache_levels`, `betas`, `radiance_samples`.
- Configuration parameter `sample_factor_c`.

**Outputs:**
- Updated throughput $L$ and weight.
- Boolean flag `cache_hit` indicating if a cached value was used.
- Updated cache state (`betas`, potentially `radiance_samples`).

### Data Structures
The implementation uses specific structures to manage the cache and query state [S1, unofficial].

**`gscache_sample`**: Records ground truth radiance samples.
- `u`: int, pixel x-coordinate.
- `v`: int, pixel y-coordinate.
- `scatter_index`: int, index into the Gaussian splat list or cache level.
- `L`: `gscache_vec3f`, the radiance vector.

**`gscache_query`**: Used to query the cache with current path state.
- `u`: int, pixel x-coordinate.
- `v`: int, pixel y-coordinate.
- `scatter_index`: int, index into the Gaussian splat list or cache level.
- `rand01`: float, a random number in $[0, 1]$ for probabilistic acceptance.
- `L`: `gscache_vec3f`, current incoming radiance/throughput.
- `throughput`: `gscache_vec3f`, current path throughput vector.

**`gscache_query_result`**: Returned by the cache query.
- `cache_hit`: bool, true if a cache hit occurred and was accepted.
- `throughput`: `gscache_vec3f`, the throughput to use for the next path segment.
- `L`: `gscache_vec3f`, the radiance value to use (either cached or original).

**`gscache_device_context`**: Manages the cache state on the device.
- `n_levels`: int, number of cache levels (max 16).
- `height`: size_t, image height.
- `width`: size_t, image width.
- `sample_factor_c`: float, scaling factor for acceptance probability.
- `cache_levels[MAX_NUM_CACHE_LEVELS]`: Array of pointers to `gscache_vec3f` arrays, storing cached radiance values.
- `radiance_samples[MAX_NUM_CACHE_LEVELS]`: Array of pointers to `gscache_vec3f` arrays, storing raw samples.
- `betas`: float array, per-pixel beta values for each cache level, used for importance weighting.

**Constants:**
- `MAX_NUM_CACHE_LEVELS`: 16 [S1, unofficial].

### Algorithm Steps
The caching logic is applied during the path tracing loop, typically after computing the contribution of a scattering event or at the end of a path segment [S1, unofficial].

1.  **Compute Luminance**: Calculate the luminance of the incoming throughput vector $L$ using standard RGB luminance coefficients [S1, unofficial]:
    $$ \text{luminance} = 0.212671 \cdot L.x + 0.715160 \cdot L.y + 0.072169 \cdot L.z $$

2.  **Calculate Acceptance Probability**: Compute the acceptance probability $q$ by scaling the luminance with `sample_factor_c` and clamping to $[0, 1]$ [S1, unofficial]:
    $$ q = \text{clamp}(\text{sample\_factor\_c} \cdot \text{luminance}, 0.0, 1.0) $$

3.  **Calculate Rejection Probability**: Compute $p$ as [S1, unofficial]:
    $$ p = 1.0 - q $$

4.  **Conditional Caching Logic**:
    -   If $q < 0.9$:
        -   Retrieve the cached radiance $L_{cached}$ from `cache_levels[scatter_index]` at index $(v \cdot \text{width} + u)$ [S1, unofficial].
        -   Retrieve the current beta value $\beta$ from `betas` at the same index [S1, unofficial].
        -   Generate a random number $r \in [0, 1]$ (denoted as `rand01` in sources) [S1, unofficial].
        -   **If** $r < p$:
            -   Accept the cache hit.
            -   Set `result.cache_hit` = true.
            -   Set `result.L` = $L_{cached} / \beta$.
            -   Keep `result.throughput` unchanged (or as passed) [S1, unofficial].
        -   **Else** ($r \ge p$):
            -   Reject the cache hit.
            -   Update the beta value for the pixel: $\beta_{new} = \beta \cdot q$ [S1, unofficial].
            -   Scale the throughput: $L_{new} = L / q$ [S1, unofficial].
            -   Set `result.cache_hit` = false.
            -   Set `result.L` = $L_{new}$.
            -   Set `result.throughput` = $L_{new}$ (or appropriate throughput update) [S1, unofficial].
    -   If $q \ge 0.9$:
        -   Skip caching logic.
        -   Set `result.cache_hit` = false.
        -   Set `result.L` = $L$.
        -   Set `result.throughput` = $L$ [S1, unofficial].

5.  **Cache Update (Implicit)**: The sources mention `radiance_samples` and `cache_levels`. While the query logic is detailed, the update logic for writing to `cache_levels` is implied to occur when samples are accepted or stored, likely involving the `gscache_sample` structure. The exact write-back mechanism (e.g., averaging, exponential moving average) is not explicitly detailed in the provided snippets, but the use of `betas` suggests an importance-weighted accumulation [S1, unofficial].

### Parameters
-   `sample_factor_c`: A float parameter that scales the luminance to determine the acceptance probability $q$. Typical values are not explicitly given, but the threshold $q < 0.9$ implies it controls the aggressiveness of caching [S1, unofficial].
-   `MAX_NUM_CACHE_LEVELS`: 16 [S1, unofficial].
-   `n_levels`: Number of active cache levels, default 3 in the config [S1, unofficial].

### Edge Cases and Pitfalls
-   **Division by Zero**: The code divides $L_{cached}$ by $\beta$. If $\beta$ is zero, this results in infinity. The implementation relies on the update rule $\beta \cdot q$ and the condition $q < 0.9$ to keep $\beta$ non-zero if initialized correctly [S1, unofficial].
-   **Bias**: The method introduces bias with no guaranteed error bounds [S2, unofficial].
-   **High Throughput Rejection**: If $q$ is close to 1.0 (high luminance), $p$ is close to 0, making cache hits unlikely unless `rand01` is extremely small. The threshold $q < 0.9$ prevents caching in these high-variance scenarios [S1, unofficial].
-   **Transfer Function Changes**: The cache requires re-initialization when transfer functions change, as it stores attenuated radiance specific to the current transfer function [S2, unofficial].
-   **Zero-Opacity Regions**: The caching mechanism may not reconstruct zero-opacity regions effectively if not handled by the underlying 3DGS representation [S2, unofficial].

## Operating characteristics
-   **Performance**: Achieves real-time frame rates on scientific datasets at 720p, with total times ranging from 38ms to 186ms [S2, unofficial].
-   **Memory Footprint**: 28 MB for the cache, compared to 272 MB for neural radiance caching [S2, unofficial].
-   **Scalability**: The method is designed for real-time rendering, leveraging the efficiency of 3DGS rasterization. It does not explicitly address scaling to very large scenes without additional techniques like distributed training (which is a separate extension) [S2, unofficial].
-   **Strengths**: Real-time performance, low memory footprint, variance reduction for path-traced volume rendering.
-   **Limitations**: Introduces bias, requires re-initialization for transfer function changes, no guaranteed error bounds, potential issues with zero-opacity regions [S2, unofficial].

## Background
3DGS Radiance Caching, also known as GSCache, was introduced by David Bauer, Qi Wu, Hamid Gadirov, and Kwan-Liu Ma in the paper "GSCache: Real-Time Radiance Caching for Volume Path Tracing using 3D Gaussian Splatting," accepted at IEEE VIS 2025 [S1, unofficial]. The work builds on the principles of 3D Gaussian Splatting for volume rendering, adapting it to serve as a compact, differentiable radiance cache for path tracing. It addresses the high variance and computational cost of path-traced volume rendering by leveraging the spatial coherence and compact representation of 3DGS primitives [S1, S2, unofficial].

## Sources
[S1] GSCache: Real-Time Radiance Caching for Volume Path Tracing using 3D Gaussian Splatting, https://github.com/DBauer15/gscache, fetched 2026-10-26 (unofficial)
[S2] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
