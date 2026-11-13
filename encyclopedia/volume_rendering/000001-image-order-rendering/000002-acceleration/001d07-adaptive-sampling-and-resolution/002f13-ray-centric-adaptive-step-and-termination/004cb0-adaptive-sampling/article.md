# Adaptive Sampling

Also called: Adaptive Sampling Rate, Adaptive Sampling Strategy, Adaptive Sampling with Bricking

## Overview
Adaptive sampling is a volume rendering optimization that adjusts the sampling density along rays based on local data significance, sampling non-homogeneous regions more densely than homogeneous ones to achieve higher quality than uniform sampling with fewer total samples [S1, S2, unofficial][S14]. It differs from its parent [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by explicitly focusing on the modulation of the sampling *rate* (step size or ray count) rather than just step size or termination, and from its siblings by encompassing a broader range of adaptation metrics including variance, gradient magnitude, and opacity [S1, S5, unofficial][S9, S10].

## How it works
**Inputs:** Volumetric dataset (scalar field $f$), camera parameters (ray origin $\mathbf{o}$, direction $\mathbf{d}$), transfer function $T(f)$, and adaptation parameters (thresholds, bounds).
**Outputs:** Final pixel color and opacity.

**Data Structures:**
*   **Ray State:** Current position $t$, accumulated opacity $A$, accumulated color $C$.
*   **Metric Buffer:** Per-pixel or per-block storage for variance, gradient magnitude, or error estimates.
*   **Sampling Rate Parameter:** A scalar $M$ or step size $\Delta t$ updated per ray or block.

**Algorithm Steps:**

1.  **Metric Evaluation:** Compute a scalar metric $M$ representing local complexity or error. Common metrics include:
    *   **Variance:** Ratio of local cell variance to global volume variance: $m = \sqrt{dVar(f_{cell})} / \sqrt{dVar(f_{vol})}$ [S12].
    *   **Gradient Magnitude:** $|\nabla s|$ to detect surface boundaries [S9].
    *   **Opacity/Contribution:** Local opacity or accumulated energy [S3, unofficial][S9].
    *   **Proximity to Isosurface:** Distance $|f - \rho|$ for silhouette detection [S10].

2.  **Adaptation Law:** Adjust the sampling parameter based on $M$.
    *   **Step Size:** $\Delta t \propto 1/M$ or bounded: $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{M}))$ [S6, S8, unofficial][S9].
    *   **Multiplier:** For uniform sampling, use an integer multiplier $M = 2m - 1 + 1$ to scale the base step $\Delta t$ [S12].
    *   **Discrete Rates:** Switch between base rate $r_0$ and max rate $r_1 = n \cdot r_0$ (typically $n=8$) if $|f - \rho| < \delta$ [S10].

3.  **Ray Marching:** Execute the volume integral using the adapted step size or ray count.
    *   Initialize $C = \mathbf{0}$, $A = 0$.
    *   Loop while $A < 1 - \epsilon$ (typically $\epsilon = 0.01$ to $0.05$) [S9][S15, unofficial]:
        *   Sample scalar value $f$ at current position.
        *   Compute opacity $\alpha = 1 - e^{-\kappa(f)\Delta t}$ and color $c = q(f)\Delta t$ [S1, unofficial].
        *   Composite: $C \leftarrow C + (1-A) \cdot c$, $A \leftarrow A + (1-A) \cdot \alpha$.
        *   Update position: $t \leftarrow t + \Delta t$.
        *   Re-evaluate metric $M$ at new position to update $\Delta t$ for the next step [S9].

4.  **Termination:** Stop if $A \ge 1 - \epsilon$ or ray exits volume [S9][S15, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sample_ray(ray_origin, ray_direction, volume, transfer_func, params):
    # Initialization
    t = 0.0
    accumulated_opacity = 0.0
    accumulated_color = vec3(0.0)
    
    # Base parameters
    base_step = params.base_step
    min_step = params.min_step
    max_step = params.max_step
    epsilon = params.epsilon # Typically 0.01
    
    # Initial step size
    step_size = base_step
    
    # Ray bounds
    t_start, t_end = compute_ray_bounds(ray_origin, ray_direction, volume)
    
    while t < t_end and accumulated_opacity < 1.0 - epsilon:
        # 1. Sample scalar field
        pos = ray_origin + t * ray_direction
        scalar_val = volume.sample(pos)
        
        # 2. Evaluate Metric M
        # Example: Gradient-based or Opacity-based
        # For simplicity, using a placeholder metric function
        metric = compute_metric(scalar_val, volume, pos)
        
        # 3. Adapt Step Size
        # Inverse proportionality with bounds
        if metric > 0:
            step_size = max_step / metric
        else:
            step_size = max_step
            
        step_size = clamp(step_size, min_step, max_step)
        
        # 4. Compute Contribution
        # Assuming transfer function returns absorption kappa and emission q
        kappa = transfer_func.get_absorption(scalar_val)
        q = transfer_func.get_emission(scalar_val)
        
        # Exponential attenuation approximation for small steps
        # T_i approx e^(-kappa * delta_x)
        # c_i approx q * delta_x
        opacity_contribution = 1.0 - exp(-kappa * step_size)
        color_contribution = q * step_size
        
        # 5. Front-to-Back Compositing
        weight = (1.0 - accumulated_opacity) * opacity_contribution
        accumulated_color += weight * color_contribution
        accumulated_opacity += weight
        
        # 6. Advance Ray
        t += step_size
        
    return accumulated_color, accumulated_opacity
```

**Edge Cases and Pitfalls:**
*   **GPU Divergence:** Completely adaptive sampling causes neighboring pixels in parallel pipelines to diverge, under-utilizing the texture cache [S10, S14]. Solution: Use discrete sampling rates or per-macrocell adaptation [S10, S12].
*   **Missed Intersections:** Discrete sampling may miss thin isosurfaces if the segment crosses twice. Guaranteed intersections require bounding the gradient length $L$ and using step size $h = |f - \rho|/L$ [S10].
*   **Bricking Overhead:** Adaptive sampling can reduce the performance overhead of bricking (texture indirection) by varying sampling rate based on proximity to the isosurface [S10].

## Subtopics

### [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md)
Uses a sparse initial grid of rays and refines based on spatial complexity heuristics, differing from the parent by explicitly using sparse structures to reduce bricking overhead.

### [Adaptive Sampling with Relative Standard Error](4545-adaptive-sampling-with-relative-standard-error.md)
Tracks per-pixel luminance squares to estimate statistical error, stopping converged pixels, differing from the parent by using statistical bounds rather than geometric metrics.

### [Adaptive Sampling in Volume Rendering](5700-adaptive-sampling-in-volume-rendering.md)
Links discrete step size to octree levels for compact 2D transfer functions, differing from the parent by explicitly connecting sampling density to voxel density and brick resolution.

### Adaptive Sampling for Silhouettes
A strategy using two discrete rates (base $r_0$ and max $r_1$) to improve intersection detection near silhouettes without diverging parallel execution pipelines [S10]. Intersection is detected when sample value $f < \text{isovalue } \rho$. Guaranteed intersections for thin sheets require gradient length bounded by $L$, with step size $h = |f - \rho|/L$ [S10]. Base rate $r_0$ is user-specified (1.0 = one voxel). Max rate $r_1 = n \cdot r_0$ (typical $n=8$). Rate increases to $r_1$ when $|f - \rho| < \delta$ [S10]. Pitfall: Completely adaptive rates cause GPU pipeline divergence; discrete rates mitigate this [S10].

### Adaptive Sampling with Opacity Correction
An approach that guarantees sampling frequency adapts to the finest level cell size while keeping total samples tractable, combined with opacity correction [S16]. Unlike generic adaptive sampling, it is tailored for AMR (Adaptive Mesh Refinement) data, avoiding the under- or over-sampling issues of hierarchy-unaware methods [S16].

### Adaptive Slicing with Slant Limiting
A method that calculates slant $dh = k_{max} - k_{min}$ and slices the volume into a prism of height $(h+dh)$ in the $-z$ direction [S17]. Slopes $k_i$ are calculated as $k_i = -\frac{\overline{GP_i} \cdot \overline{V}}{\overline{z} \cdot \overline{V}}$ [S17]. Points are projected using $P^0_i = P_i + (k_i - k_{max})\vec{-z}$ and $P^j_i = P^0_i + \frac{j}{n-1}(h+dh)\vec{-z}$ [S17]. Slant is limited to $k_M = 2h$ to prevent infinity at grazing angles [S17]. Pitfall: Grazing angles cause infinite slant; handled by $k_M$ [S17]. Transparent fragments from empty regions are rasterized even if discarded by alpha test [S17].

### Bricked Multi-Resolution Single-Pass Ray-Casting
Uses adaptive object- and image-space sampling in a single-pass fragment program [S18, unofficial]. Adapts sampling density along the ray and performs adaptive image-space sampling based on screen tile statistics [S18, unofficial]. Fetches neighborhood samples for interpolation and uses transfer function-based LOD selection [S18, unofficial]. Introduced by Ljung et al. (2006), Beyer et al. (2008), and Jeong et al. (2009) [S18, unofficial].

## Operating characteristics
*   **Time Complexity:** Achieves logarithmic time complexity for low-opacity uniform fog or hit-and-stop high-opacity cases [S3, unofficial].
*   **Performance:** Speed-up of 2.6-2.8 compared to full density sampling [S6, unofficial]. On the Asian Dragon dataset (512x256x256), with brick size 32 and adaptive sampling ($r_1=8r_0$), frame rates are 19.2 fps at rate 0.25, 13.8 fps at 0.5, 10.2 fps at 1, and 6.9 fps at 2 [S10]. Without bricking and adaptive sampling, performance is higher, but adaptive sampling with bricking achieves 80-85% of that performance [S10].
*   **Strengths:** Significantly increases rendering speed in datasets with large empty or low-detail regions [S7, unofficial][S14]. Reduces computational cost by casting fewer rays in homogeneous areas [S5, unofficial][S9].
*   **Limitations:** Poor thread coherence on GPUs if purely adaptive (per-pixel) [S12, S14]. Requires careful tuning of thresholds ($\delta, \epsilon$) and bounds ($\Delta t_{min}, \Delta t_{max}$) [S9, S10].
*   **When to Use:** High-quality rendering of large datasets with varying detail [S14]. Real-time applications where frame budget is tight [S8, unofficial].
*   **When Not to Use:** Uniform, homogeneous volumes where fixed-step is sufficient [S7, unofficial]. Scenes where GPU thread divergence is a critical bottleneck and discrete rates are not feasible [S10].

## Background
Adaptive sampling was introduced by Hanrahan et al. in 1992 [S2, unofficial]. It was developed to address the inefficiency of uniform sampling in volumetric data, which often contains large regions of empty space or homogeneous material [S7, unofficial][S14]. Early implementations focused on ray-based error metrics [S3, unofficial], while later work integrated it with bricking and multi-resolution structures to optimize GPU performance [S10][S18, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering Volume Rendering. Lecture 21. Acknowledgements. These slides are collected from many sources. A particularly valuable source is the IEEE Visualization conference tutorials. Sources fr, https://www.slideserve.com/hayes-levine/volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-25 (unofficial)
[S3] Any fast voxel rendering algorithms? - GameDev.net, https://gamedev.net/forums/topic/597410-any-fast-voxel-rendering-algorithms/4785388/, fetched 2026-10-26 (unofficial)
[S5] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S6] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S7] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S8] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S9] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S10] PDF Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S12] Volume Ray Casting with Peak Finding and Differential ..., https://www.sci.utah.edu/~knolla/avrc.pdf, fetched 2026-10-27
[S13] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S14] Volume ray casting - HandWiki, https://handwiki.org/wiki/Volume_ray_casting, fetched 2026-10-29
[S15] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
[S16] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S17] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S18] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
