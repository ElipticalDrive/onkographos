# Adaptive Sampling

Also called: Adaptive Sampling Rate, Adaptive Sampling Strategy, Adaptive Sampling with Bricking, Adaptive density, Opacity-based Adaptation

## Overview
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays or across the image plane to reduce computational cost while maintaining visual fidelity [S1, S2, S3, unofficial]. Unlike uniform sampling, which casts a constant number of samples per ray regardless of local data complexity, adaptive sampling concentrates samples in regions of high detail (e.g., near surfaces or high gradients) and reduces them in homogeneous or empty regions [S1, S4, unofficial][S6]. This method builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by explicitly modulating the step size $\Delta t$ or sample count based on local metrics, often using a two-pass or hierarchical approach to balance parallel execution efficiency with accuracy [S2, unofficial][S8].

## How it works
The core mechanism involves evaluating a local metric $M$ (such as gradient magnitude, opacity, or variance) and using it to adjust the sampling parameter $P$ (step size, rate, or probability) [S6, S8]. The specific implementation varies by variant, but general principles include:

1.  **Metric Evaluation:** Compute a scalar value representing local complexity or error. Common metrics include the magnitude of the scalar field gradient $|\nabla s|$ [S6], the ratio of local cell variance to global variance $m$ [S8], or the difference in accumulated opacity [S10, unofficial].
2.  **Adaptation:** Adjust the sampling rate. For step size, $\Delta t$ is often inversely proportional to the metric: $\Delta t \propto 1/M$ or bounded by $\min(\Delta t_{max}, \max(\Delta t_{min}, k/M))$ [S6, S8]. For discrete rates, a base rate $r_0$ is increased to a maximum rate $r_1$ when the metric exceeds a threshold [S7].
3.  **Rendering:** Execute the volume integral with the adjusted parameters. This may involve a single pass with dynamic step sizes or a multi-pass strategy where initial sparse samples determine where additional rays are needed [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Volumetric dataset (scalar field $f$), transfer function, camera parameters (ray origin $O$, direction $D$), and adaptation parameters (thresholds, min/max step sizes, base rates).
*   **Outputs:** Rendered image pixel values (color and opacity).

### Data Structures
*   **Macrocells/Blocks:** The volume is often divided into blocks (e.g., bricks of size $32^3$) to compute local metrics efficiently [S2, unofficial][S7, S8].
*   **Variance/Gradient Maps:** Precomputed or on-the-fly maps storing local statistics (e.g., max standard deviation per cell) to guide sampling decisions [S8].

### Numbered Steps (General Adaptive Strategy)
1.  **Initialize:** Set ray parameters ($O, D$) and initial sampling state (e.g., `density = 0.0`, `color = 0.0`) [S9, unofficial].
2.  **Evaluate Metric:** Compute the local metric $M$ at the current ray position or block.
    *   *Opacity-based:* Check accumulated opacity $A_i$ [S6].
    *   *Gradient-based:* Compute $|\nabla s|$ [S6].
    *   *Variance-based:* Compute $m = \sqrt{dVar(f_{cell})} / \sqrt{dVar(f_{vol})}$ [S8].
    *   *Null-region:* Check if $|A_i^\Delta - A_i^{\Delta-1}| < \epsilon$ [S10, unofficial].
3.  **Adjust Step Size/Rate:**
    *   If using step size: $\Delta t = \text{clamp}(k/M, \Delta t_{min}, \Delta t_{max})$ [S6].
    *   If using discrete rates: If metric indicates high detail (e.g., $|f - \rho| < \delta$), switch from base rate $r_0$ to max rate $r_1 = n \cdot r_0$ [S7].
    *   If using variance: Multiplier $M = 2m - 1 + 1$; new step $\Delta t_{new} = M \cdot \Delta t_{uniform}$ [S8].
4.  **Sample and Composite:**
    *   Advance ray by $\Delta t$.
    *   Sample scalar field $f$ and compute color/opacity contribution.
    *   Update accumulated opacity and color using front-to-back compositing [S6].
5.  **Termination:**
    *   Stop if accumulated opacity $> 1 - \epsilon$ (typically $\epsilon = 0.01-0.05$) [S6].
    *   Stop if ray exits volume or enters a null-value region with increased step size [S10, unofficial].
    *   Stop if density threshold is reached (e.g., `density >= 1.0`) [S9, unofficial].

### Pseudocode (Opacity and Gradient Driven)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sample_ray(ray_origin, ray_direction, volume, transfer_func, params):
    # Parameters
    epsilon = params.epsilon  # e.g., 0.01
    min_step = params.min_step
    max_step = params.max_step
    base_step = params.base_step
    
    color = vec3(0.0)
    accumulated_opacity = 0.0
    t = 0.0
    
    while t < volume.max_dist:
        # 1. Evaluate Metric
        pos = ray_origin + t * ray_direction
        scalar_val = volume.sample(pos)
        gradient_mag = volume.gradient_magnitude(pos)
        
        # 2. Adjust Step Size
        # Inverse relationship with gradient magnitude for detail preservation
        # Avoid division by zero
        if gradient_mag > 0:
            step = base_step / gradient_mag
        else:
            step = base_step
            
        # Clamp step size
        step = max(min_step, min(max_step, step))
        
        # 3. Sample and Composite
        sample_val = volume.sample(pos)
        color_contrib, opacity_contrib = transfer_func(sample_val)
        
        # Front-to-back compositing
        alpha = opacity_contrib * step
        color += color_contrib * alpha * (1.0 - accumulated_opacity)
        accumulated_opacity += alpha * (1.0 - accumulated_opacity)
        
        # 4. Termination Check
        if accumulated_opacity > 1.0 - epsilon:
            break
            
        # Null-region optimization: if opacity change is negligible, skip
        if opacity_contrib < epsilon:
            step = max_step
            
        t += step
        
    return color
```

### Parameters with Typical Values
*   **Epsilon ($\epsilon$):** Termination threshold for accumulated opacity. Typical values: $0.01$ to $0.05$ [S6].
*   **Base Sampling Rate ($r_0$):** Corresponds to 1 voxel spacing. User-specified [S7].
*   **Multiplier ($n$):** Ratio of max to base rate ($r_1 = n \cdot r_0$). Typical value: $n=8$ [S7].
*   **Threshold ($\delta$):** Distance to isovalue for switching rates in silhouette detection. Constant for distance fields, data-dependent for CT scans [S7].
*   **Target Frame Time:** Used in dynamic adaptation. Typical value: $32$ ms [S5, unofficial].

### Edge Cases and Pitfalls
*   **Parallel Divergence:** Completely adaptive sampling causes neighboring pixels in a warp/quad to diverge, under-utilizing texture caches and SIMD units. Solution: Use discrete sampling rates (e.g., only $r_0$ and $r_1$) rather than continuous step sizes [S7, S8].
*   **Missed Intersections:** In isosurface rendering, large step sizes may skip thin structures if the segment crosses the isosurface twice. Guaranteed intersections require bounded gradient length $L$ and step size $h < |f - \rho|/L$ [S7].
*   **Grazing Angles:** In slicing methods, slant can go to infinity at grazing view angles. Handled by limiting slant $k_M = 2h$ [S13].
*   **Bricking Overhead:** Adaptive sampling with bricking (texture indirection) can reduce performance to 80-85% of non-bricked rendering, but still offers significant speedups over uniform sampling with bricking [S7].

## Subtopics

### [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md)
Exploits coherence by casting a sparse grid of rays and adaptively increasing density in high-complexity regions. Suitable for scan-conversion methods.

### [Adaptive Sampling with Relative Standard Error](4545-adaptive-sampling-with-relative-standard-error.md)
Tracks per-pixel luminance squares to estimate error, stopping converged pixels.

### Adaptive Sampling for Silhouettes
A strategy using two discrete rates (base $r_0$ and max $r_1$) to improve intersection detection near silhouettes without diverging parallel pipelines [S7]. It increases the rate from $r_0$ to $r_1$ when the sample value is within a threshold $\delta$ of the isovalue $\rho$. This avoids the cache under-utilization of fully adaptive rates [S7].

### Adaptive Sampling with Opacity Correction
An approach tailored for AMR (Adaptive Mesh Refinement) data that guarantees sampling frequency adapts to the finest cell size while keeping sample counts tractable, combined with opacity correction to prevent under/over-sampling errors common in generic AMR renderers [S11].

### Adaptive Sampling in Volume Rendering
A technique where the fragment shader enumerates non-empty leaves in front-to-back order and chooses a step size matching local voxel density, adapting to brick resolution. It uses a compact 2D transfer function with octree levels for opacity weighting [S12].

### Adaptive Slicing with Slant Limiting
A method that calculates slant $dh$ and limits it to $k_M$ to prevent infinity at grazing angles. It adjusts the sampling rate close to voxel size to improve time, handling transparent fragments from empty regions via clip registers or accepting rasterization cost [S13].

## Operating characteristics
*   **Time Complexity:** Reduces the number of volume queries significantly. Speed-ups of 2.6-2.8x over full density sampling are reported [S3, unofficial]. In some cases, rendering time scales with the inverse of the sampling distance (e.g., 24 fps at distance 1.0 vs 66 fps at distance 5.0) [S3, unofficial].
*   **Memory Complexity:** Requires storage for local metrics (variance/gradient maps) or additional ray buffers for multi-pass strategies [S2, unofficial][S8].
*   **Performance:** On the Asian Dragon dataset (512x256x256), adaptive sampling with bricking achieves 19.2 fps at rate 0.25, compared to 6.9 fps at rate 2 (uniform) [S7]. It achieves 80-85% of the performance of rendering without bricking [S7].
*   **Strengths:** Significant speedup in datasets with large empty or homogeneous regions; preserves detail in high-frequency areas.
*   **Limitations:** Potential for missed intersections if step sizes are too large; parallel divergence if not constrained to discrete rates; overhead of computing metrics.

## Background
Adaptive sampling originated as a response to the inefficiency of uniform sampling in volumetric datasets, which often consist mostly of empty space or homogeneous regions [S4, unofficial]. Early work by [S1, unofficial] highlighted that uniform sampling assumes homogeneity, leading to artifacts or wasted computation. Subsequent developments focused on GPU efficiency, noting that purely adaptive methods suffer from poor thread coherence on GPUs, leading to the adoption of per-macrocell or discrete-rate strategies [S8]. The method is widely used in real-time volume rendering, medical imaging, and atmospheric visualization [S3, S9, S10, unofficial].

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
