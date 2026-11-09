# Adaptive Sampling

Also called: Adaptive Sampling Rate, Adaptive Sampling Strategy, Adaptive Sampling with Bricking, Adaptive density, Opacity-based Adaptation

## Overview
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays or across the image plane to reduce computational cost while maintaining visual fidelity [S1, S2, S3, unofficial]. It solves the inefficiency of uniform sampling, which wastes queries in homogeneous or empty regions and may alias in high-frequency areas [S4, unofficial][S6]. Unlike its siblings, [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md) focuses on latency constraints and [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md) focuses on pre-integration table lookups, this method modulates the step size $\Delta t$ or sample count based on local data metrics (opacity, gradient, variance) [S6, S7, S8].

## How it works
The method deviates from fixed-step ray marching by dynamically adjusting the sampling rate $R$ or step size $\Delta t$ based on a computed metric $M$. The specific implementation varies by adaptation law (opacity, gradient, variance, or discrete rates).

### Inputs and Outputs
*   **Inputs:** Volumetric dataset $V$, ray origin $O$, ray direction $D$, transfer function $T$, isovalue $\rho$ (for isosurface), target frame time $t_{target}$ (for frame-rate control).
*   **Outputs:** Final pixel color $C$, accumulated opacity $A$.

### Data Structures
*   **Ray State:** Current position $P$, accumulated opacity $A$, accumulated color $C$, step size $\Delta t$.
*   **Metric Cache:** For block-based methods, a variance map or macrocell metric $M$ [S2, unofficial][S8].
*   **Control Parameters:** Minimum step $\Delta t_{min}$, maximum step $\Delta t_{max}$, thresholds $\epsilon, \delta$ [S6, S7][S10, unofficial].

### Adaptation Laws and Steps

#### 1. Opacity-Based Adaptation (Continuous)
Used for direct volume rendering or early termination [S6][S10, unofficial].
1.  **Initialize:** $A = 0$, $C = 0$, $t = t_{start}$.
2.  **Evaluate Metric:** Compute local opacity contribution or accumulated opacity change.
    *   For null-space skipping: Check if $\Delta A = A_i - A_{i-1} < \epsilon$ [S10, unofficial].
    *   For general opacity: Compute local opacity $\alpha$ from transfer function.
3.  **Adjust Step Size:**
    *   If in null region ($\Delta A < \epsilon$): Increase $\Delta t$ significantly to skip empty space [S10, unofficial].
    *   Otherwise: $\Delta t \propto \frac{1}{\alpha}$ or bounded by $\Delta t_{min}, \Delta t_{max}$ [S6].
4.  **Advance:** $t \leftarrow t + \Delta t$, $P \leftarrow O + tD$.
5.  **Composite:** Sample $V$ at $P$, compute $\alpha, c$. Update $A \leftarrow A + \alpha(1-A)$, $C \leftarrow C + c(1-A)$ [S1, unofficial].
6.  **Terminate:** If $A > 1 - \epsilon_{term}$ (typically $\epsilon_{term} \in [0.01, 0.05]$), break loop [S6].

#### 2. Gradient-Driven Adaptation
Used for isosurface or high-frequency detail preservation [S6, S7].
1.  **Compute Gradient:** Calculate $|\nabla s|$ at current position $P$ [S6].
2.  **Adjust Step Size:**
    *   $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{|\nabla s|}))$ [S6].
    *   Alternatively, for silhouette detection, use discrete rates $r_0$ and $r_1 = n r_0$ [S7].
3.  **Silhouette Trigger:** If $|s(P) - \rho| < \delta$, switch to $r_1$ (finer sampling) [S7].
4.  **Advance and Sample:** Proceed as in Opacity-Based.

#### 3. Block-Based Variance Adaptation (Two-Step)
Used in local solver or bricked rendering contexts [S2, unofficial][S7].
1.  **Initial Cast:** Cast a sparse set of rays (e.g., 12.5% or 32 rays per block) across the image/block [S2, unofficial].
2.  **Variance Analysis:** Compute variance $\sigma^2$ of the rendered output for each block [S2, unofficial].
3.  **Adaptive Refinement:** For blocks with $\sigma^2 > \text{threshold}$, cast additional rays [S2, unofficial].
4.  **Reconstruction:** Run local solver or compositing with the combined ray set [S2, unofficial].

#### 4. Frame-Rate Control (Global)
Used for interactive applications [S5, unofficial].
1.  **Measure Time:** Record actual render time $t_{actual}$.
2.  **Calculate Target Rate:** $R_{target} = \frac{t_{actual}}{t_{target}}$ [S5, unofficial].
3.  **Smooth Update:** $R_{current} \leftarrow \alpha R_{current} + (1-\alpha) R_{target}$ (e.g., $\alpha=0.8$) [S5, unofficial].
4.  **Apply:** Adjust global $\Delta t$ or sample count based on $R_{current}$ [S5, unofficial].

#### 5. Macrocell Metric Adaptation
Used in differential sampling [S8].
1.  **Compute Metric:** For each macrocell, calculate $m = \frac{\sqrt{dVar(f_{cell})}}{\sqrt{dVar(f_{vol})}}$ [S8].
2.  **Calculate Multiplier:** $M = 2m - 1 + 1$ (rounded to positive integer) [S8].
3.  **Adjust Sampling:**
    *   Uniform: $\Delta t_{new} = M \Delta t$ [S8].
    *   Differential: Modify increments $\frac{\partial^2 T_M}{\partial x^2} = \frac{M(M+1)}{2} a$ and $\frac{\partial T_M}{\partial x} = M a + \frac{\partial^2 T_M}{\partial x^2} + b$ [S8].

### Pseudocode: General Adaptive Ray Marching
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_march(ray_origin, ray_direction, volume, transfer_func, isovalue=None):
    # Initialization
    accumulated_opacity = 0.0
    accumulated_color = vec3(0.0)
    t = ray_origin_length  # Distance along ray
    step_size = initial_step_size
    
    # Parameters
    epsilon_term = 0.05
    epsilon_null = 1e-6
    delta_silhouette = 0.1
    max_step = 1.0
    min_step = 0.01
    
    # Loop
    while accumulated_opacity < 1.0 - epsilon_term:
        # Sample volume
        pos = ray_origin + t * ray_direction
        scalar_value = volume.sample(pos)
        
        # Determine step size based on adaptation law
        if isovalue is not None:
            # Gradient/Silhouette adaptation
            gradient_mag = volume.gradient_magnitude(pos)
            if abs(scalar_value - isovalue) < delta_silhouette:
                step_size = max_step # Finer sampling near surface
            else:
                step_size = clamp(min_step, max_step, 1.0 / (gradient_mag + 1e-6))
        else:
            # Opacity/Null-space adaptation
            alpha = transfer_func(scalar_value)
            if alpha < epsilon_null:
                step_size = max_step # Skip empty space
            else:
                step_size = clamp(min_step, max_step, 1.0 / (alpha + 1e-6))
        
        # Composite
        alpha = transfer_func(scalar_value)
        color = transfer_func_color(scalar_value)
        
        # Front-to-back compositing
        weight = alpha * (1.0 - accumulated_opacity)
        accumulated_color += weight * color
        accumulated_opacity += weight
        
        # Advance
        t += step_size
        
        # Safety break
        if t > ray_exit_distance:
            break
            
    return accumulated_color, accumulated_opacity
```

### Edge Cases and Pitfalls
*   **Thread Divergence:** Fully adaptive sampling causes neighboring pixels to diverge in execution paths, under-utilizing GPU texture caches [S7]. Solution: Use discrete sampling rates (e.g., base $r_0$ and max $r_1$) [S7].
*   **Missed Intersections:** Large step sizes may skip thin structures if the gradient is not bounded. Guaranteed intersection requires $h \le |f - \rho|/L$ where $L$ is the gradient bound [S7].
*   **Grazing Angles:** In slicing methods, grazing view angles cause slant to approach infinity. Limit slant $dh$ to $k_M = 2h$ [S13].
*   **Bricking Overhead:** Adaptive sampling can mitigate bricking overhead, achieving 80-85% of non-bricked performance [S7].

## Subtopics

### [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md)
A rendering optimization that exploits coherence in projections to reduce computational overhead by casting a sparse grid of rays (less than one per pixel) and adaptively increasing ray density in regions of high image complexity. It differs from uniform sampling by prioritizing high-frequency details and from pure brute-force methods by achieving savings of more than two orders of magnitude for many datasets. It is particularly well-suited for scan-conversion methods where polygons are filtered prior to ray tracing.

### [Adaptive Sampling with Relative Standard Error](4545-adaptive-sampling-with-relative-standard-error.md)
An adaptive sampling strategy that tracks per-pixel luminance squares to estimate relative standard error, stopping converged pixels and skipping noisy neighborhoods. It differs from [Adaptive Sampling](1385-adaptive-sampling.md) by using statistical error metrics rather than geometric proximity to the isosurface to determine sampling rates, and from [Adaptive Sampling (Sparse Grid)](1656-adaptive-sampling-sparse-grid.md) by operating in image order per-pixel rather than modifying the underlying voxel data structure.

### Adaptive Sampling for Silhouettes
A sampling strategy using two discrete rates (base $r_0$ and max $r_1$) to improve intersection detection near silhouettes without diverging parallel execution pipelines [S7]. Intersection is detected when sample value $f < \text{isovalue } \rho$. To guarantee intersection for thin sheets, the gradient length must be bounded by $L$, requiring step size $h = |f - \rho|/L$ [S7]. Base rate $r_0$ is user-specified (1.0 = one voxel). Max rate $r_1 = n r_0$ (typically $n=8$) is used when $|f - \rho| < \delta$ [S7]. Threshold $\delta$ is constant for distance fields ($L=1$) but data-dependent for CT scans [S7].

### Adaptive Sampling with Opacity Correction
An adaptive sampling approach that guarantees the sampling frequency can adapt to the finest level cell size while keeping total sample counts tractable, combined with opacity correction [S11]. Unlike OSPRay's existing adaptive sampling code which is not aware of the underlying AMR hierarchy and may severely under- or over-sample, this method is tailored for AMR data [S11].

### Adaptive Sampling in Volume Rendering
A technique where the fragment shader enumerates non-empty leaves in front-to-back order and chooses a step size matching local voxel density. It adapts sampling density to brick resolution [S12]. For isosurface rendering, it looks for intervals bracketing the isovalue. For semitransparent DVR, it implements a Phong illumination model with boundary enhancement and view-dependent transparency, stepping by discrete intervals associated with octree levels [S12]. Uses a compact precomputed 2D transfer function, using octree level for the second dimension to adjust opacity and color weighting [S12].

### Adaptive Slicing with Slant Limiting
A method that calculates slant $dh = k_{max} - k_{min}$ and slices the volume into a prism of height $(h+dh)$ in the $-z$ direction [S13]. It chooses a sampling rate close to voxel size to improve rendering time, unlike applications that choose higher rates for quality [S13]. Grazing view angles are handled by limiting slant to $k_M = 2h$ [S13]. Transparent fragments from empty regions are rasterized even if discarded by alpha test, handled by clip registers or accepting rasterization cost [S13].

## Operating characteristics
*   **Time Complexity:** Reduces the number of volume queries from $O(N)$ to $O(N_{eff})$ where $N_{eff} \ll N$ in homogeneous regions [S4, unofficial][S6].
*   **Memory Complexity:** Low overhead for metric calculation; block-based methods require additional variance buffers [S2, unofficial].
*   **Performance:**
    *   Achieves speed-up of 2.6-2.8 compared to full density sampling [S3, unofficial].
    *   Sampling distance 1.0 yields 24 fps; 5.0 yields 66 fps [S3, unofficial].
    *   On Asian Dragon (512x256x256), brick size 32, adaptive sampling ($r_1=8r_0$) yields 19.2 fps at rate 0.25, 13.8 fps at 0.5, 10.2 fps at 1, and 6.9 fps at 2 [S7].
    *   Reduces bricking overhead to 80-85% of non-bricked performance [S7].
*   **Strengths:** Significant speedup in empty/homogeneous regions; improved quality in high-frequency areas; better cache utilization with discrete rates [S4, unofficial][S7].
*   **Limitations:** Thread divergence with fully adaptive rates; complexity in implementation; potential for missed thin structures if step size is too large [S7][S10, unofficial].
*   **When to Use:** Interactive rendering, large datasets with significant empty space, high-frequency details near surfaces [S4, unofficial][S6].
*   **When Not to Use:** Highly homogeneous volumes where uniform sampling is already optimal, or when strict geometric accuracy is required without bounding gradient assumptions [S7].

## Background
Adaptive sampling originated as a response to the inefficiencies of uniform ray marching in volume rendering [S1, unofficial][S6]. Early work highlighted that volumetric datasets often consist mostly of low-detail regions (e.g., empty space), making uniform sampling wasteful [S4, unofficial]. The technique evolved to include opacity-based termination [S6], gradient-driven refinement for isosurfaces [S7], and block-based variance analysis for local solvers [S2, unofficial]. Modern implementations address GPU-specific issues like thread divergence by using discrete sampling rates [S7] and integrate with hierarchical data structures like AMR and octrees [S11, S12].

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
