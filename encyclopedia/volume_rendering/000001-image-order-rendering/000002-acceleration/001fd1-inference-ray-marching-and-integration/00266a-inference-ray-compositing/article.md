# Inference Ray Compositing

Also called: Ray Compositing for Inference, Ray Compositing with Ambient Light (Inference), Volume Rendering Composite (Inference)

## Overview
Inference Ray Compositing is the final rendering stage of the Inference Ray Marching and Integration pipeline, responsible for aggregating per-step volumetric properties (density, color, depth, ambient light, uncertainty) into final pixel buffers [S1, S2, unofficial]. It differs from training-time compositing by operating on pre-computed ray samples with strict early-termination logic to maximize throughput during real-time inference [S2, unofficial]. It serves as the bridge between the sparse ray marching results and the final image output, handling multiple output channels (RGB, depth, ambient, uncertainty) in parallel [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `n_alive`: Integer, number of rays currently active (not terminated) [S3, unofficial].
*   `n_step`: Integer, maximum number of sampling steps per ray [S3, unofficial].
*   `rays_alive`: Boolean array `[N]`, mask of active rays [S1, unofficial].
*   `rays_t`: Integer array `[N]`, current step index for each ray [S1, unofficial].
*   `sigmas`: Float array `[M]`, volume densities at sampled points [S1, unofficial].
*   `rgbs`: Float array `[M, 3]`, RGB colors at sampled points [S1, unofficial].
*   `deltas`: Float array `[M, 2]`, step sizes (distances) between samples [S1, unofficial].
*   `weights_sum`: Float array `[N]`, accumulated opacity (alpha) for each ray [S1, unofficial].
*   `depth`: Float array `[N]`, accumulated depth for each ray [S1, unofficial].
*   `image`: Float array `[N, 3]`, accumulated RGB color for each ray [S1, unofficial].
*   `T_thresh`: Float, transparency threshold for early termination (default $10^{-2}$) [S1, S3, unofficial].
*   `ambients`: Float array `[M, 3]`, ambient lighting contributions (optional, for ambient variant) [S1, unofficial].
*   `uncertainties`: Float array `[M]`, uncertainty values (optional, for uncertainty variant) [S1, unofficial].
*   `uncertainty_sum`: Float array `[N]`, accumulated uncertainty (optional) [S1, unofficial].
*   `ambient_sum`: Float array `[N]`, accumulated ambient light (optional) [S1, unofficial].

**Outputs:**
*   `weights_sum`: Updated accumulated opacity [S1, unofficial].
*   `depth`: Updated accumulated depth [S1, unofficial].
*   `image`: Updated final RGB image [S1, unofficial].
*   `rays_t`: Updated step indices; set to $-1$ if terminated early, or final step count if completed [S2, unofficial].
*   `uncertainty_sum`: Updated accumulated uncertainty (optional) [S1, unofficial].
*   `ambient_sum`: Updated accumulated ambient light (optional) [S1, unofficial].

### Data Structures
The method operates on CUDA tensors. The layout is contiguous in memory.
*   `sigmas`, `deltas`: Shape `[M]` and `[M, 2]` respectively, where $M$ is the total number of samples across all alive rays.
*   `rays_alive`, `rays_t`, `weights_sum`, `depth`, `image`: Shape `[N]` or `[N, 3]`, where $N$ is the number of alive rays.
*   `T_thresh`: Scalar float.

### Formulas
The core compositing formula for each step $i$ along a ray is [S2, unofficial]:

$$
\alpha_i = 1 - \exp(-\sigma_i \cdot \delta_i)
$$

$$
T_i = 1 - \sum_{j=0}^{i-1} w_j
$$

$$
w_i = \alpha_i \cdot T_i
$$

$$
\text{weights\_sum} \leftarrow \text{weights\_sum} + w_i
$$

$$
\text{depth} \leftarrow \text{depth} + w_i \cdot t_i
$$

$$
\text{image} \leftarrow \text{image} + w_i \cdot \text{rgb}_i
$$

Where:
*   $\sigma_i$ is the density (`sigmas[i]`).
*   $\delta_i$ is the step size (`deltas[i]`).
*   $t_i$ is the distance of the current sample from the camera origin.
*   $T_i$ is the transmittance up to step $i$, calculated as $1 - \text{weights\_sum}$ before adding the current weight.
*   $w_i$ is the weight contributed by the current sample.

For the **ambient** variant, the color equation includes an ambient term $A_i$:
$$
\text{image} \leftarrow \text{image} + w_i \cdot (\text{rgb}_i + A_i)
$$
And `ambient_sum` accumulates $w_i \cdot A_i$ [S1, unofficial].

For the **uncertainty** variant, uncertainty $U_i$ is accumulated similarly:
$$
\text{uncertainty\_sum} \leftarrow \text{uncertainty\_sum} + w_i \cdot U_i
$$
[S1, unofficial].

### Parameters
*   `T_thresh`: Float, default $10^{-2}$. Rays with transmittance $T < T_{thresh}$ are terminated early [S1, S3, unofficial].

### Steps
1.  **Initialization**: Initialize `weights_sum`, `depth`, and `image` to zero for all rays [S2, unofficial]. Initialize `rays_t` to 0 [S2, unofficial].
2.  **Loop**: For each step $s$ from 0 to `n_step` - 1:
    a.  Identify active rays using `rays_alive` [S2, unofficial].
    b.  For each active ray $n$:
        i.  Retrieve $\sigma$, $\text{rgb}$, $\delta$ for the current step $s$ [S2, unofficial].
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
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def composite_rays(n_alive, n_step, rays_alive, rays_t, sigmas, rgbs, deltas, weights_sum, depth, image, T_thresh=1e-2):
    """
    Standard ray compositing for inference.
    """
    # Initialize accumulators if not already done by caller
    # Note: In CUDA implementation, these are passed in and updated in-place
    
    for step in range(n_step):
        # Check if any rays are still alive
        if not any(rays_alive):
            break
            
        for n in range(n_alive):
            if not rays_alive[n]:
                continue
                
            # Get current step data for ray n
            # Assuming sigmas, rgbs, deltas are flattened or indexed appropriately
            # In practice, this is done in parallel on GPU
            sigma = sigmas[n * n_step + step]
            rgb = rgbs[n * n_step + step]
            delta = deltas[n * n_step + step]
            
            # Early termination if delta is zero
            if delta == 0:
                rays_t[n] = -1
                rays_alive[n] = False
                continue
                
            # Compute alpha
            alpha = 1.0 - torch.exp(-sigma * delta)
            
            # Compute transmittance
            T = 1.0 - weights_sum[n]
            
            # Compute weight
            w = alpha * T
            
            # Update accumulators
            weights_sum[n] += w
            depth[n] += w * (step + 1) # Assuming t is step index or pre-computed distance
            image[n] += w * rgb
            
            # Early termination based on transparency
            if T < T_thresh:
                rays_t[n] = -1
                rays_alive[n] = False
                continue
                
        # Update rays_t for completed rays
        # This is typically handled after the loop or in a separate pass
```

### Edge Cases and Pitfalls
*   **Delta Zero**: If `deltas` contains zero, the ray should be terminated immediately to avoid division by zero or invalid alpha calculations [S2, unofficial].
*   **Precision**: Use `float32` for compositing to balance speed and accuracy. Mixed precision (`float16`) may be used but requires careful handling of small weights [S3, unofficial].
*   **Boundary Conditions**: Ensure `rays_t` is correctly updated to $-1$ for terminated rays to distinguish them from rays that have completed all steps [S2, unofficial].
*   **Memory Layout**: The input arrays `sigmas`, `rgbs`, `deltas` are typically flattened or structured to allow efficient GPU access. Misalignment can cause performance degradation [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of steps. Early termination reduces the effective $S$ [S2, unofficial].
*   **Memory Complexity**: $O(N \cdot S)$ for storing intermediate ray samples, plus $O(N)$ for accumulators [S1, unofficial].
*   **Performance**: Highly optimized for GPU execution using CUDA kernels. Throughput scales with the number of alive rays and available GPU memory [S3, unofficial].
*   **Strengths**: Real-time performance, supports multiple output channels (RGB, depth, ambient, uncertainty), efficient early termination [S1, S2, unofficial].
*   **Limitations**: Requires pre-computed ray samples, sensitive to `T_thresh` parameter [S1, S3, unofficial].
*   **When to Use**: Real-time volume rendering inference where latency is critical [S1, unofficial].
*   **When Not to Use**: Training phase where gradients are needed through the compositing step (use differentiable compositing instead) [S3, unofficial].

## Background
Inference Ray Compositing is part of the ER-NeRF (Efficient Real-time Neural Radiance Fields) framework, which focuses on real-time novel view synthesis [S1, unofficial]. It builds upon the Inference Ray Marching and Integration method, which provides the per-ray samples [Parent]. The method was developed to enable real-time rendering by optimizing the compositing step for GPU execution, leveraging early termination and parallel processing [S1, S2, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
