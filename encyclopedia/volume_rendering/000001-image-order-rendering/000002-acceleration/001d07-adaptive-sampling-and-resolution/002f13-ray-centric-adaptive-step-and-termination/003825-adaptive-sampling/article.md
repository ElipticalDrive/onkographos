# Adaptive Sampling

Also called: Adaptive Sampling Rate, Adaptive Sampling Strategy, Adaptive Sampling with Bricking, Adaptive Volume Sampling, Adaptive density, Opacity-based Adaptation

## Overview
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays to reduce computational cost while preserving visual quality [S1, unofficial][S6]. Unlike uniform sampling, which treats all regions equally, adaptive sampling concentrates samples in high-detail or high-opacity areas and skips empty space [S4, unofficial][S10, unofficial]. It sets itself apart from its parent method, [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md), by explicitly addressing GPU hardware constraints (warp divergence and texture cache coherence) through quantization and block-level metrics, rather than relying solely on continuous per-pixel step adjustments [S7][S8].

## How it works
Adaptive sampling modifies the standard ray marching loop by dynamically determining the step size $\Delta t$ based on a local metric $m(\mathbf{x})$ or accumulated state.

### Inputs and Outputs
*   **Inputs:** Scalar field $f(\mathbf{x})$, ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, transfer function $T(f)$, isovalue $\rho$ (for isosurface rendering), and parameters controlling adaptation (thresholds, multipliers, discrete rates) [S6][S7].
*   **Outputs:** Final pixel color $C$ and accumulated opacity $\alpha$ [S6].

### Data Structures
*   **Metric Storage:** For per-macrocell or per-block adaptation, a scalar metric $m$ is computed for each region [S8].
*   **State Variables:** Accumulated opacity $\alpha$, accumulated color $C$, current position $t$, step size $\Delta t$, and previous accumulated opacity $\alpha_{prev}$ for null-region detection [S6][S10, unofficial].

### Algorithm Steps

1.  **Metric Calculation:** Compute a local metric $m(\mathbf{x})$ to estimate complexity or emptiness. Common metrics include:
    *   **Opacity/Density:** Local scalar value $f(\mathbf{x})$ or segment opacity [S6][S10, unofficial].
    *   **Gradient:** Magnitude $|\nabla f(\mathbf{x})|$ [S6, S7].
    *   **Variance:** Ratio of local cell variance to global variance [S8].
    *   **Null-Region Check:** Difference in accumulated opacity between steps [S10, unofficial].

2.  **Step Size Determination:** Calculate $\Delta t$ based on $m$.
    *   **Inverse Opacity:** $\Delta t \propto 1/f(\mathbf{x})$ or $\Delta t \propto 1/(1-\alpha)$ [S6].
    *   **Gradient-Based:** $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, k / |\nabla f|))$ [S6].
    *   **Variance-Based:** Compute multiplier $M = 2m - 1 + 1$ (where $m = \sqrt{dVar(f_{cell})} / \sqrt{dVar(f_{vol})}$), then $\Delta t_{new} = M \Delta t_{base}$ [S8].
    *   **Null-Region Skip:** If $|\alpha_i - \alpha_{i-1}| < \epsilon$, increase $\Delta t$ significantly [S10, unofficial].

3.  **Quantization for Coherence:** To prevent warp divergence on GPUs, quantize $\Delta t$ to a small set of discrete values (e.g., base rate $r_0$ and max rate $r_1 = n r_0$) [S7, S8].

4.  **Termination Check:** Terminate if accumulated opacity $\alpha > 1 - \epsilon_{term}$ (typically $\epsilon_{term} = 0.01 \dots 0.05$) or ray exits bounding box [S6][S10, unofficial].

5.  **Compositing:** Update color and opacity using front-to-back compositing:
    $$ \alpha_{seg} = 1 - e^{-f(\mathbf{x})\Delta t} $$
    $$ C_{seg} = T(f(\mathbf{x})) \cdot \alpha_{seg} $$
    $$ \alpha \leftarrow \alpha + \alpha_{seg}(1 - \alpha) $$
    $$ C \leftarrow C + C_{seg}(1 - \alpha) $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sample_ray(ray_origin, ray_direction, volume_data, transfer_func, params):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    volume_data: function sample(x) returning scalar f(x)
    transfer_func: function T(f) returning vec3 color
    params: dict containing:
        - base_step: float (initial delta_t)
        - min_step: float
        - max_step: float
        - opacity_threshold: float (1 - epsilon_term)
        - null_threshold: float (epsilon for null region check)
        - gradient_scale: float (k in gradient-based step)
        - discrete_rates: list of floats [r0, r1, ...] for quantization
    """
    t = 0.0
    accumulated_alpha = 0.0
    accumulated_color = vec3(0.0, 0.0, 0.0)
    prev_accumulated_alpha = 0.0
    
    # Determine entry and exit points (simplified, assuming unit sphere or known bounds)
    t_start, t_end = compute_ray_bounds(ray_origin, ray_direction, volume_bounds)
    
    # Initial step size
    step_size = params['base_step']
    
    while t < t_end and accumulated_alpha < params['opacity_threshold']:
        # 1. Sample scalar field
        pos = ray_origin + t * ray_direction
        f_val = volume_data.sample(pos)
        
        # 2. Compute Metric and Adjust Step Size
        # Example: Gradient-based or Opacity-based adaptation
        # Note: Gradient calculation requires neighboring samples or precomputed gradient field
        # Here we use a simplified opacity/null-region check for demonstration
        
        current_segment_opacity = 1.0 - exp(-f_val * step_size)
        
        # Null-region skip logic [S10, unofficial]
        if abs(accumulated_alpha - prev_accumulated_alpha) < params['null_threshold']:
            # Entered null region, increase step size
            step_size = min(step_size * 2.0, params['max_step'])
        else:
            # Reset or refine step size based on local density/gradient
            # Simplified inverse opacity adaptation [S6]
            if f_val > 0:
                step_size = params['gradient_scale'] / f_val
            else:
                step_size = params['max_step']
            
            # Clamp step size
            step_size = clamp(step_size, params['min_step'], params['max_step'])
            
            # Quantize to discrete rates for GPU coherence [S7]
            step_size = quantize_to_discrete_rate(step_size, params['discrete_rates'])
        
        # 3. Compositing
        alpha_seg = 1.0 - exp(-f_val * step_size)
        color_seg = transfer_func(f_val) * alpha_seg
        
        accumulated_alpha += alpha_seg * (1.0 - accumulated_alpha)
        accumulated_color += color_seg * (1.0 - accumulated_alpha)
        
        prev_accumulated_alpha = accumulated_alpha
        t += step_size
        
        # Early termination if fully opaque
        if accumulated_alpha >= params['opacity_threshold']:
            break
            
    return accumulated_color, accumulated_alpha

def quantize_to_discrete_rate(step, rates):
    """
    Selects the closest discrete rate from the list to maintain warp coherence.
    """
    if not rates:
        return step
    min_diff = float('inf')
    best_rate = rates[0]
    for r in rates:
        diff = abs(step - r)
        if diff < min_diff:
            min_diff = diff
            best_rate = r
    return best_rate
```

### Parameters and Typical Values
*   **Base Step Size:** Corresponds to 1 voxel or a fraction thereof [S7].
*   **Opacity Threshold ($\epsilon_{term}$):** $0.01 \dots 0.05$ [S6].
*   **Null Threshold ($\epsilon$):** Small positive value, e.g., $10^{-4}$ [S10, unofficial].
*   **Gradient Scale ($k$):** Tuned to dataset resolution [S6].
*   **Discrete Rates:** $r_0 = 1.0$ (1 voxel), $r_1 = 8 r_0$ [S7].

### Edge Cases and Pitfalls
*   **Warp Divergence:** Fully adaptive per-pixel step sizes cause divergence in GPU warps. Solution: Quantize step sizes to a small set of discrete values [S7, S8].
*   **Missed Intersections:** In isosurface rendering, large steps may skip thin structures. Guaranteed intersections require bounded gradient $L$ and step size $h \le |f - \rho|/L$ [S7].
*   **Texture Cache Thrashing:** Adaptive sampling can lead to poor texture cache utilization if sampling positions diverge significantly between neighboring pixels [S7].

## Subtopics

### [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md)
Adaptive Sampling (Sparse Grid) is a rendering optimization that exploits coherence in projections—often introduced by sensing device blurring—to reduce computational overhead by casting a sparse grid of rays (less than one per pixel) and adaptively increasing ray density in regions of high image complexity. It differs from uniform sampling by prioritizing high-frequency details and from pure brute-force methods by achieving savings of more than two orders of magnitude for many datasets. It is particularly well-suited for scan-conversion methods where polygons are filtered prior to ray tracing.

### [Adaptive Sampling with Relative Standard Error](4545-adaptive-sampling-with-relative-standard-error.md)
An adaptive sampling strategy that tracks per-pixel luminance squares to estimate relative standard error, stopping converged pixels and skipping noisy neighborhoods.

### Adaptive Sampling for Silhouettes
A sampling strategy using two discrete rates (base $r_0$ and max $r_1$) to improve intersection detection near silhouettes without diverging parallel execution pipelines [S7]. Intersection is detected when sample value $f < \text{isovalue } \rho$ (assuming front-to-back and isosurface defined by $f < \rho$). Exact intersection lies between samples. Missed intersections are possible if the segment crosses the isosurface twice. Guaranteed intersections require bounded gradient length $L$: $h = |f - \rho|/L$ [S7]. The base sampling rate $r_0$ is specified by the user (1.0 corresponds to a single voxel). The max sampling rate is $r_1 = n r_0$ (typical $n=8$) [S7]. The rate increases from $r_0$ to $r_1$ when $|sample\_value - \text{isovalue}| < \delta$ [S7]. For distance fields, $\delta$ is constant; for CT scans, it depends on data [S7]. Completely adaptive sampling causes neighboring pixels in parallel pipelines to diverge, under-utilizing the texture cache; the solution is to use only two discrete rates [S7].

### Adaptive Sampling with Opacity Correction
An adaptive sampling approach that guarantees the sampling frequency can adapt to the finest level cell size while keeping total sample counts tractable, combined with opacity correction [S11]. Unlike OSPRay's existing adaptive sampling code which is not aware of the underlying AMR hierarchy and may severely under- or over-sample, this method is tailored for AMR data [S11].

### Adaptive Sampling in Volume Rendering
A technique where the fragment shader enumerates non-empty leaves in front-to-back order and chooses a step size matching local voxel density. It adapts sampling density to brick resolution [S12]. For isosurface rendering, it looks for intervals that bracket the selected isovalue. For semitransparent direct volume rendering, it implements a Phong illumination model with boundary enhancement and view-dependent transparency, stepping by a discrete number of intervals directly associated to octree levels [S12]. The transfer function is a compact precomputed 2D function using the octree level for the second dimension to adjust opacity and color weighting [S12].

### Adaptive Slicing with Slant Limiting
Calculates slant $dh = k_{max} - k_{min}$. The sliced volume becomes a prism of height $(h+dh)$ in the $-z$ direction [S13]. The parameters are defined as:
$$ k_i = -\frac{\overline{GP_i} \cdot \overline{V}}{\overline{z} \cdot \overline{V}} $$
$$ k_{min} = \min(k_i) \text{ and } k_{max} = \max(k_i) $$
$$ P^0_i = P_i + (k_i - k_{max})\vec{-z} $$
$$ P^j_i = P^0_i + \frac{j}{n-1}(h+dh)\vec{-z}, \quad j = 0..n-1 $$
$$ d'_h = \min(dh, k_M) $$
$$ P^j_i = P_i + \frac{d'_h}{dh}(k_i - k_{max}) + \frac{j}{n-1}(h+d'_h)\vec{-z} $$
$$ \delta = \frac{1}{n-1}(h+d'_h)\vec{-z} \cdot \vec{V} $$
$$ n = 1 + \frac{L}{2^l} \frac{(h+d'_h)\vec{-z} \cdot \vec{V}}{1} $$
The maximum allowed slant $k_M$ prevents infinity at grazing view angles, with value $k_M = 2h$ [S13]. $L$ is the texture resolution (side length of $L^3$ volume) and $l$ is the MIP-map level [S13]. Grazing view angles cause slant to go to infinity, handled by limiting slant to $k_M$ [S13]. Transparent fragments from empty top/bottom regions are rasterized even if discarded by alpha test, handled by using clip registers feature (future) or accepting rasterization cost [S13]. Volume rendering applications often choose higher sampling rates than voxel size for quality; this method chooses rate close to voxel size to improve rendering time [S13].

## Operating characteristics
*   **Time Complexity:** Reduces the number of samples per ray in empty or homogeneous regions, leading to speed-ups of 2.6–2.8x compared to full density sampling [S3, unofficial]. Performance metrics show sampling distance 1.0 yields 24 fps, while 5.0 yields 66 fps [S3, unofficial].
*   **Memory Complexity:** Requires storage for metrics (e.g., variance per macrocell) or precomputed gradient fields [S8].
*   **Measured Performance:** On the Asian Dragon dataset (512x256x256), with brick size 32 and adaptive sampling ($r_1=8r_0$), the frame rate is 19.2 fps at sampling rate 0.25, 13.8 fps at 0.5, 10.2 fps at 1, and 6.9 fps at 2 [S7]. Adaptive sampling with bricking reduces the performance overhead of bricking, achieving 80-85% of the performance of rendering without bricking and without adaptive sampling [S7].
*   **Strengths:** Significant speed-up in datasets with large empty spaces or homogeneous regions [S4, unofficial]. Preserves visual quality by focusing samples on high-detail areas [S1, unofficial].
*   **Limitations:** Warp divergence if not quantized [S7]. Texture cache thrashing if sampling positions diverge [S7]. Potential for missed intersections in isosurface rendering if steps are too large [S7].
*   **When to Use:** Real-time rendering of large volumetric datasets with significant empty space or varying detail levels [S4, unofficial].
*   **When Not to Use:** When strict guaranteed intersections are required without additional refinement steps and the gradient is unbounded [S7].

## Background
Adaptive sampling originated as a technique to improve rendering quality over uniform sampling by sampling non-homogeneous regions more densely [S1, unofficial]. It was further developed to reduce computational cost by casting rays in two steps: an initial sparse cast followed by additional rays for blocks with high variance [S2, unofficial]. The technique varies sampling density across volume levels to improve performance while maintaining visual quality [S3, unofficial]. It subdivides volumetric datasets into regions of varying detail levels to adjust sampling rates, reducing computation in low-detail areas and increasing it in high-detail areas [S4, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S3] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S4] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S5] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S6] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S7] PDF Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S8] Volume Ray Casting with Peak Finding and Differential ..., https://www.sci.utah.edu/~knolla/avrc.pdf, fetched 2026-10-27
[S9] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S10] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
[S11] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S12] PDF A single-pass GPU ray casting framework for interactive out-of-core ..., https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
[S13] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
