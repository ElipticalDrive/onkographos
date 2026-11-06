# Inference Ray Compositing

Also called: Ray Compositing for Inference, Ray Compositing with Ambient Light (Inference), Volume Rendering Composite (Inference)

## Overview
Inference Ray Compositing aggregates per-step volumetric properties (density, color, depth, ambient light, uncertainty) into final pixel buffers [S1, S2, unofficial]. It differs from training-time compositing by operating on pre-computed ray samples with strict early-termination logic to maximize throughput during real-time inference [S2, unofficial]. It serves as the bridge between sparse ray marching results and the final image output, handling multiple output channels in parallel [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `n_alive`: Integer, number of rays currently active [S3, unofficial].
*   `n_step`: Integer, maximum number of sampling steps per ray [S3, unofficial].
*   `rays_alive`: Boolean array `[N]`, mask of active rays [S1, unofficial].
*   `rays_t`: Integer array `[N]`, current step index for each ray [S1, unofficial].
*   `sigmas`: Float array `[M]`, volume densities at sampled points [S3, unofficial].
*   `rgbs`: Float array `[M, 3]`, RGB colors at sampled points [S3, unofficial].
*   `deltas`: Float array `[M, 2]`, step sizes between samples [S3, unofficial].
*   `weights_sum`: Float array `[N]`, accumulated opacity for each ray [S1, unofficial].
*   `depth`: Float array `[N]`, accumulated depth for each ray [S1, unofficial].
*   `image`: Float array `[N, 3]`, accumulated RGB color for each ray [S1, unofficial].
*   `T_thresh`: Float, transparency threshold for early termination (default $10^{-2}$) [S1, S3, unofficial].
*   `ambients`: Optional float array for ambient lighting contributions [S1, unofficial].
*   `uncertainties`: Optional float array for uncertainty values [S1, unofficial].
*   `uncertainty_sum`: Optional float array `[N]`, accumulated uncertainty [S1, unofficial].
*   `ambient_sum`: Optional float array `[N]`, accumulated ambient light [S1, unofficial].

**Outputs:**
*   `weights_sum`: Updated accumulated opacity [S1, unofficial].
*   `depth`: Updated accumulated depth [S1, unofficial].
*   `image`: Updated final RGB image [S1, unofficial].
*   `rays_t`: Updated step indices; set to $-1$ if terminated early, or final step count if completed [S2, unofficial].
*   `uncertainty_sum`: Updated accumulated uncertainty (optional) [S1, unofficial].
*   `ambient_sum`: Updated accumulated ambient light (optional) [S1, unofficial].

### Data Structures
The method operates on contiguous CUDA tensors [G]. The layout follows a flattened structure where $M$ is the total number of samples across all alive rays, and $N$ is the number of alive rays [S3, unofficial]. Exact memory packing details beyond contiguous layout are not given in the sources.

### Formulas
The core compositing formula for each step along a ray is [S2, unofficial]:

$$
\alpha = 1 - \exp(-\sigma \cdot \delta)
$$

$$
T = 1 - \text{weights\_sum}
$$

$$
w = \alpha \cdot T
$$

$$
\text{weights\_sum} \leftarrow \text{weights\_sum} + w
$$

$$
\text{depth} \leftarrow \text{depth} + w \cdot t
$$

$$
\text{image} \leftarrow \text{image} + w \cdot \text{rgb}
$$

Where:
*   $\sigma$ is the density (`sigmas[i]`) [S2, unofficial].
*   $\delta$ is the step size (`deltas[i]`) [S2, unofficial].
*   $t$ is the distance of the current sample from the camera origin [S2, unofficial].
*   $T$ is the transmittance up to the current step, calculated as $1 - \text{weights\_sum}$ before adding the current weight [S2, unofficial].
*   $w$ is the weight contributed by the current sample [S2, unofficial].

For the **ambient** variant, the color equation includes an ambient term $A$:
$$
\text{image} \leftarrow \text{image} + w \cdot (\text{rgb} + A)
$$
And `ambient_sum` accumulates $w \cdot A$ [S1, unofficial].

For the **uncertainty** variant, uncertainty $U$ is accumulated similarly:
$$
\text{uncertainty\_sum} \leftarrow \text{uncertainty\_sum} + w \cdot U
$$
[S1, unofficial].

### Parameters
*   `T_thresh`: Float, default $10^{-2}$. Rays with transmittance $T < T_{thresh}$ are terminated early [S1, S3, unofficial].

### Steps
1.  **Initialization**: Initialize `weights_sum`, `depth`, and `image` to zero for all rays [S2, unofficial]. Initialize `rays_t` to 0 [S2, unofficial].
2.  **Loop**: While `step < n_step`:
    a.  Identify active rays using `rays_alive` [S2, unofficial].
    b.  For each active ray $n$:
        i.  Retrieve $\sigma$, $\text{rgb}$, $\delta$ for the current step [S2, unofficial].
        ii. If $\delta = 0$, mark ray $n$ as terminated (set `rays_t[n] = -1`) and skip further processing for this ray [S2, unofficial].
        iii. Compute $\alpha = 1 - \exp(-\sigma \cdot \delta)$ [S2, unofficial].
        iv. Compute $T = 1 - \text{weights\_sum}[n]$ [S2, unofficial].
        v. Compute $w = \alpha \cdot T$ [S2, unofficial].
        vi. Update $\text{weights\_sum}[n] \leftarrow \text{weights\_sum}[n] + w$ [S2, unofficial].
        vii. Update $\text{depth}[n] \leftarrow \text{depth}[n] + w \cdot t$ [S2, unofficial].
        viii. Update $\text{image}[n] \leftarrow \text{image}[n] + w \cdot \text{rgb}$ [S2, unofficial].
        ix. If using ambient: Update $\text{ambient\_sum}[n] \leftarrow \text{ambient\_sum}[n] + w \cdot \text{ambient}$ and add ambient to color [S1, unofficial].
        x. If using uncertainty: Update $\text{uncertainty\_sum}[n] \leftarrow \text{uncertainty\_sum}[n] + w \cdot \text{uncertainty}$ [S1, unofficial].
        xi. If $T < T_{thresh}$, mark ray $n$ as terminated (set `rays_t[n] = -1`) and skip further steps for this ray [S2, unofficial].
    c.  If all rays are terminated, break loop [S2, unofficial].
3.  **Finalization**: For rays that completed all `n_step` steps without early termination, set `rays_t[n]` to the final step count [S2, unofficial]. Write final `weights_sum`, `depth`, and `image` to output buffers [S2, unofficial].

### Pseudocode
```python
import math

def composite_rays(n_alive, n_step, rays_alive, rays_t, sigmas, rgbs, deltas, weights_sum, depth, image, T_thresh=1e-2):
    """
    Standard ray compositing for inference.
    Matches CUDA kernel _backend.composite_rays signature.
    """
    # Initialize accumulators if not already done by caller
    for n in range(n_alive):
        weights_sum[n] = 0.0
        depth[n] = 0.0
        image[n] = [0.0, 0.0, 0.0]
        rays_t[n] = 0

    # Track distance from camera origin per ray
    ray_dist = [0.0] * n_alive

    for step in range(n_step):
        if not any(rays_alive):
            break
            
        for n in range(n_alive):
            if not rays_alive[n]:
                continue
                
            idx = n * n_step + step
            sigma = sigmas[idx]
            rgb = rgbs[idx]
            delta = deltas[idx]
            
            # Early termination if delta is zero
            if delta == 0:
                rays_t[n] = -1
                rays_alive[n] = False
                continue
                
            # Accumulate distance from camera origin
            ray_dist[n] += delta
            t = ray_dist[n]
            
            # Compute alpha
            alpha = 1.0 - math.exp(-sigma * delta)
            
            # Compute transmittance
            T = 1.0 - weights_sum[n]
            
            # Compute weight
            w = alpha * T
            
            # Update accumulators
            weights_sum[n] += w
            depth[n] += w * t
            image[n][0] += w * rgb[0]
            image[n][1] += w * rgb[1]
            image[n][2] += w * rgb[2]
            
            # Early termination based on transparency
            if T < T_thresh:
                rays_t[n] = -1
                rays_alive[n] = False
                continue
                
    # Finalize rays_t for completed rays
    for n in range(n_alive):
        if rays_alive[n]:
            rays_t[n] = n_step
```

### Edge Cases and Pitfalls
*   **Delta Zero**: If `deltas` contains zero, the ray should be terminated immediately to avoid invalid alpha calculations [S2, unofficial].
*   **Precision**: Use `float32` for compositing to balance speed and accuracy [S3, unofficial].
*   **Boundary Conditions**: Ensure `rays_t` is correctly updated to $-1$ for terminated rays to distinguish them from rays that have completed all steps [S2, unofficial].
*   **Early Termination Logic**: The transmittance check $T < T_{thresh}$ must occur after weight accumulation to prevent premature termination of visible surfaces [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Highly optimized for GPU execution using CUDA kernels [S3, unofficial].
*   **Strengths**: Real-time performance, supports multiple output channels (RGB, depth, ambient, uncertainty), efficient early termination [S1, S2, unofficial].
*   **Limitations**: Requires pre-computed ray samples, sensitive to `T_thresh` parameter [S1, S3, unofficial].
*   **When to Use**: Real-time volume rendering inference where latency is critical [S1, unofficial].
*   **When Not to Use**: Training phase where gradients are needed through the compositing step (use differentiable compositing instead) [S3, unofficial].

## Background
Inference Ray Compositing is part of the ER-NeRF (Efficient Real-time Neural Radiance Fields) framework, which focuses on real-time novel view synthesis [S1, unofficial]. It builds on [Inference Ray Marching and Integration](2148-inference-ray-marching-and-integration.md), which provides the per-ray samples. The method was developed to enable real-time rendering by optimizing the compositing step for GPU execution, leveraging early termination and parallel processing [S1, S2, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
