# Ray-Centric Adaptive Step and Termination

Also called: Adaptive Step Size, Adaptive Ray Marching, Adaptive Integration

## Overview
Ray-Centric Adaptive Step and Termination is a family of volume rendering acceleration techniques that dynamically adjust the discretization step size $\Delta t$ along a ray and determine when to terminate the ray march based on local scene properties or accumulated opacity. It solves the inefficiency of uniform sampling, which wastes computation in empty or homogeneous regions and may alias high-frequency details in complex areas [S1, S4, unofficial][S6]. What sets this family apart from its siblings, such as [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) (which separates coarse and fine networks) or [Random Pixel Subsampling](1668-random-pixel-subsampling.md) (which selects pixels stochastically), is its focus on the *spatial* resolution of the integration path itself rather than the network architecture or pixel selection strategy. It differs from [Adaptive Refinement for Volume Rendering](1892-adaptive-refinement-for-volume-rendering.md) by typically operating as a single-pass or primary-pass optimization of the ray march, whereas refinement often implies a secondary pass or hierarchical structure.

## Shared principles
All methods in this family share the core mechanism of varying the integration step size $\Delta t$ and the termination condition based on local metrics. The general ray marching loop is modified to compute a metric $m(\mathbf{x})$ at the current position $\mathbf{x} = \mathbf{o} + t\mathbf{d}$, derive $\Delta t$ from $m$, and check for termination based on accumulated opacity $\alpha$.

**1. Metric Calculation**
A local metric $m(\mathbf{x})$ is computed to estimate the complexity or emptiness of the volume at the current ray position. Common metrics include:
*   **Density/Opacity:** The scalar field value $f(\mathbf{x})$ or the segment opacity $1 - e^{-f(\mathbf{x})\Delta t}$ [S6][S10, unofficial].
*   **Gradient:** The magnitude of the scalar field gradient $|\nabla f(\mathbf{x})|$, indicating rapid changes [S6, S7].
*   **Variance:** The local variance $\sigma^2$ of the scalar field within a cell, normalized by the global variance [S8].
*   **Error:** An estimate of the rendering error, such as the relative standard error of luminance [S2, S4545].

**2. Step Size Determination**
The step size $\Delta t$ is calculated based on the metric $m$. The goal is to increase $\Delta t$ in smooth/empty regions and decrease it in complex/opaque regions.
*   **Inverse Density/Opacity:** $\Delta t \propto 1/m$ or $\Delta t \propto 1/(1-\alpha)$, where $\alpha$ is the accumulated opacity [S6].
*   **Gradient-Based:** $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, k / |\nabla f|))$, where $k$ is a constant [S6].
*   **Variance-Based:** An integer multiplier $M$ is derived from the metric, e.g., $M = 2m - 1 + 1$, and $\Delta t_{new} = M \Delta t_{base}$ [S8].
*   **Null-Region Skip:** If the change in accumulated opacity between steps is negligible ($|\alpha_i - \alpha_{i-1}| < \epsilon$), $\Delta t$ is increased significantly to skip empty space [S10, unofficial].

**3. Termination Check**
The ray march terminates when the accumulated opacity $\alpha$ exceeds a threshold $1 - \epsilon_{term}$ (typically $\epsilon_{term} = 0.01 \dots 0.05$) or when the ray exits the bounding box ($t > t_{far}$) [S6][S10, unofficial].

**4. Compositing**
Samples are taken at the determined positions, and front-to-back compositing is used to update the accumulated color $C$ and opacity $\alpha$:
$$ \alpha_{seg} = 1 - e^{-f(\mathbf{x})\Delta t} $$
$$ C_{seg} = T(f(\mathbf{x})) \cdot \alpha_{seg} $$
$$ \alpha \leftarrow \alpha + \alpha_{seg}(1 - \alpha) $$
$$ C \leftarrow C + C_{seg}(1 - \alpha) $$

**5. GPU Coherence and Quantization**
Fully adaptive per-pixel step sizes cause warp divergence and texture cache thrashing on GPUs. A common shared principle is to quantize step sizes to a small set of discrete values (e.g., base and max) to maintain coherence [S7, S8].

## Subtopics

### [Adaptive Sampling](1385-adaptive-sampling.md)
Broadly encompasses strategies that adjust sampling density based on local data metrics (density, gradient, variance) or image-space feedback (error, variance). It differs from this family's specific focus on step size $\Delta t$ and termination by including broader density adjustments.

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Adjusts sampling based on user input state (active/inactive) to maintain interactive frame rates, rather than local geometric metrics.

### [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md)
Focuses on termination criteria specifically within the context of preintegration techniques.

### Adaptive Sampling with Uniform Fallback
*   **Definition:** A variant that uses adaptive sampling (e.g., via nerfacc) but falls back to uniform sampling for rays that receive too few samples, ensuring robust coverage [S1, unofficial].
*   **Steps:**
    1.  Compute $t_{near}$ and $t_{far}$ using ray-AABB intersection. Filter valid rays where $t_{far} > t_{near}$ [S1, unofficial].
    2.  If no valid rays, return empty tensors [S1, unofficial].
    3.  Call the adaptive estimator (e.g., `self.nerfacc_estimator.sampling`) with valid rays to get sample positions $t_s, t_e$ and ray indices [S1, unofficial].
    4.  Count samples per ray using `torch.bincount`. Identify rays with counts below a threshold `fallback_when_samples` as `low_ids` [S1, unofficial].
    5.  For `low_ids`, compute uniform samples: $t_{s,fb} = t_0 + k \cdot dt$, $t_{e,fb} = t_{s,fb} + dt$, where $dt = (t_1 - t_0)/K$ and $k \in [0, K-1]$. Clamp final $t_{e,fb}$ to $t_1$ [S1, unofficial].
    6.  Concatenate the adaptive samples with the fallback uniform samples [S1, unofficial].

### Loss-based Adaptive Sampling
*   **Definition:** A strategy that allocates more samples to image regions with higher rendering loss, identified by partitioning the image into an 8x8 grid [S2].
*   **Steps:**
    1.  Partition each input image into 64 regions $R_j$ ($j=1..64$) [S2].
    2.  Randomly sample a uniform distribution of pixels $s_i$ from source views [S2].
    3.  Compute mean loss per region $R_j$ using gradient loss $e^g$ and pixel loss $e^p$ [S2].
    4.  Normalize region losses into a probability distribution $f_i[j]$ [S2].
    5.  Resample pixels for the next iteration by allocating $n_i \cdot f_i[j]$ samples to each region $R_j$, where $n_i$ is the total sample count per image [S2].
*   **Formulas:**
    *   Region Loss: $\mathcal{H}_i[j] = \frac{1}{|r_j|} \sum_{(u,v) \in r_j} e_i^g[u, v] + e_i^p[u, v]$, where $r_j = s_i \cap R_j$ [S2].
    *   Probability Distribution: $f_i[j] = \frac{\mathcal{H}_i[j]}{\sum_{m=1}^{64} \mathcal{H}_i[m]}$ [S2].
*   **Parameters:** Grid size is 8x8 (64 regions). $n_i$ is the total sample count per image [S2].

## Background
Adaptive step sizing in volume rendering originated from the need to optimize ray marching in medical imaging and scientific visualization, where datasets are often sparse or contain large homogeneous regions. Early work focused on gradient-based step sizing to capture isosurfaces accurately [S7]. The concept has been widely adopted in modern neural rendering (NeRF) to reduce the number of samples required for convergence, often combined with hierarchical sampling or loss-based redistribution [S1, unofficial][S2].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
