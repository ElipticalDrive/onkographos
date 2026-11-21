# Adaptive Undersampling

Also called: Adaptive undersampling with pre-computed interval integrals, Fast Adaptive Undersampling, Two-Pass Adaptive Undersampling with G-Buffer, Two-pass Adaptive Undersampling with Pre-computed Intervals, Two-pass adaptive screen sampling, Two-pass adaptive undersampling with interval G-buffer

## Overview
Adaptive Undersampling is a two-pass rendering technique that accelerates direct volume ray casting by partitioning the integration domain into equal intervals and storing partial integrals in a G-buffer [S1, S2]. Unlike standard adaptive undersampling which interpolates final pixel colors, this method interpolates pre-computed interval integrals, allowing the second pass to calculate integrals only over specific intervals where coherence fails, rather than integrating over the entire ray [S1, S2]. This approach increases frames per second (FPS) by a factor of 1.5 on average compared to standard adaptive undersampling [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume texture (unit cube, side length 1), transfer function $f$, ray origin $s$, ray end $f$, block size $n$, number of intervals $M$, color threshold $\delta$.
*   **Outputs:** Final pixel color $C$ and opacity $A$.

**Data Structures:**
*   **G-Buffer:** A set of $M+1$ parallel textures with resolution $1/n$ of the viewport along each side.
    *   Each entry is a 4-component vector (RGBA).
    *   Index 0: $G[0].a$ stores the distance to the start of the non-zero transfer function domain; $G[0].rgb$ stores the final composited color.
    *   Indices $1 \dots M$: $G[i].rgb$ stores the color contribution of the $i$-th interval; $G[i].a$ stores the opacity contribution of the $i$-th interval.

**Parameters:**
*   $M$: Maximum number of intervals. Typical value: 8 [S1, S2].
*   $n$: Side length of square blocks for pixel partitioning. One pixel is processed per $n \times n$ block in Pass 1 [S1, S2].
*   $\delta$: Threshold for color difference comparison. Typical value: 0.05 [S1, S2].

**Algorithm Steps:**

**Pass 1: Pre-computation**
1.  Partition pixels into $n \times n$ blocks. Select one pixel per block.
2.  For each selected pixel, calculate the distance $a$ from ray start $s$ to the first point $v$ where the transfer function is non-zero using `GetDistanceForStart(s, f)` [S1, S2].
3.  If $a \le 0$ (no volume hit), set $G[0].rgb = \text{BackgroundColor}$ and halt for this pixel [S1].
4.  Initialize $G = 0$, $i = 0$, $C = 0$, $A = 0$ [S1].
5.  Calculate step size: $\text{step} = \sqrt{3} / M$. This assumes the volume is in a unit cube, so the maximum ray length is $\sqrt{3}$ [S1, S2].
6.  Calculate maximum iterations: $\text{imax} = \lfloor \text{length}(f - s) / \text{step} \rfloor$ [S1].
7.  Calculate last step remainder: $\text{lastStep} = \text{length}(f - s) - \text{step} \times \text{imax}$ [S1].
8.  Normalize ray direction: $r = \text{normalize}(f - s)$ [S1].
9.  Set start point: $v = s + G[0].a \times r$ [S1].
10. Loop while $A < 1$ and $i \le \text{imax}$:
    a. Compute partial integral: $G[i + 1] = \text{VolumeIntegral}(v, \text{step})$ [S1].
    b. Update color: $C = C + (1 - A) \times G[i + 1].\text{rgb} \times G[i + 1].a$ [S1].
    c. Update opacity: $A = A + (1 - A) \times G[i + 1].a$ [S1].
    d. Advance ray: $v = v + \text{step} \times r$ [S1].
    e. Increment $i$ [S1].
11. If $A < 1$ after the loop, compute final partial integral with `lastStep`:
    a. $G[\text{imax} + 1] = \text{VolumeIntegral}(v, \text{lastStep})$ [S1].
    b. $C = C + (1 - A) \times G[\text{imax} + 1].\text{rgb} \times G[\text{imax} + 1].a$ [S1].
12. Store final color: $G[0].\text{rgb} = C$ [S1].

**Pass 2: Interpolation or Partial Integration**
1.  For each remaining pixel, check if adjacent pixels' colors are sufficiently close. Condition: $\max_j \| G_j[0].\text{rgb} - G_i[0].\text{rgb} \| < \delta$ for all neighbors $j$ [S1].
2.  If true, perform bilinear interpolation of neighbor colors: $C = \text{BilinearInterpolation}(G_i.\text{rgb})$ [S1].
3.  If false, calculate $C$ using recurrence relation:
    a. Initialize $A = 0$, $C = 0$ [S1].
    b. Accelerate start point search: $t = r \times \min\{G_i[0].a\}$ [S1].
    c. Calculate new start: $v = s + r \times \text{GetDistanceForStart}(s + t, f)$ [S1].
    d. Loop $k$ from 1 to $M$:
        i. Check if neighbors' interval $k$ integrals are close: $\max_j \| G_j[k].\text{rgb} - G_i[k].\text{rgb} \| < \delta / M$ [S1].
        ii. If true, $I = \text{BilinearInterpolation}(G_i[k])$ [S1].
        iii. If false, determine length: if $k = M$, $\text{Length} = \text{lastStep}$, else $\text{Length} = \text{step}$ [S1]. Calculate $I = \text{VolumeIntegral}(v, \text{Length})$ [S1].
        iv. Update color: $C = C + (1 - A) \times I.\text{rgb} \times I.a$ [S1].
        v. Update opacity: $A = A + (1 - A) \times I.a$ [S1].
        vi. Advance ray: $v = v + \text{step} \times r$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pass1(s, f, M, n, step_size_func, volume_integral_func):
    # Partition logic handled by caller selecting one pixel per n*n block
    a = get_distance_for_start(s, f)
    if a <= 0:
        return BackgroundColor, 0
    
    # Initialize G-buffer entry for this pixel
    # G[0].a = a
    # G[0].rgb = BackgroundColor (default)
    
    C = 0.0
    A = 0.0
    i = 0
    
    # Max ray length in unit cube is sqrt(3)
    step = sqrt(3) / M
    ray_length = length(f - s)
    imax = floor(ray_length / step)
    last_step = ray_length - step * imax
    r = normalize(f - s)
    v = s + a * r
    
    while A < 1.0 and i <= imax:
        # VolumeIntegral returns (rgb, a)
        rgb, alpha = volume_integral_func(v, step)
        
        C = C + (1.0 - A) * rgb * alpha
        A = A + (1.0 - A) * alpha
        
        # Store in G-buffer at index i+1
        # G[i+1].rgb = rgb
        # G[i+1].a = alpha
        
        v = v + step * r
        i = i + 1
        
    if A < 1.0:
        rgb, alpha = volume_integral_func(v, last_step)
        C = C + (1.0 - A) * rgb * alpha
        # G[imax+1].rgb = rgb
        # G[imax+1].a = alpha
        
    # G[0].rgb = C
    return C, A

def pass2(s, f, M, delta, G_buffer, step_size_func, volume_integral_func):
    # Get neighbors' G-buffer entries (indices 0..3)
    neighbors = get_neighbor_gbuffer_entries()
    
    # Check color coherence
    max_color_diff = max(norm(n[0].rgb - neighbors[0].rgb) for n in neighbors)
    
    if max_color_diff < delta:
        return bilinear_interpolation(neighbors[0].rgb)
    
    # Incoherent: perform partial integration
    C = 0.0
    A = 0.0
    ray_length = length(f - s)
    step = sqrt(3) / M
    imax = floor(ray_length / step)
    last_step = ray_length - step * imax
    r = normalize(f - s)
    
    # Accelerate start point search
    min_a = min(n[0].a for n in neighbors)
    t = r * min_a
    v = s + r * get_distance_for_start(s + t, f)
    
    for k in range(1, M + 1):
        # Check interval coherence
        max_interval_diff = max(norm(n[k].rgb - neighbors[k].rgb) for n in neighbors)
        
        if max_interval_diff < delta / M:
            # Interpolate interval integral
            rgb, alpha = bilinear_interpolation_interval(neighbors, k)
        else:
            # Calculate specific interval
            length = last_step if k == M else step
            rgb, alpha = volume_integral_func(v, length)
            
        C = C + (1.0 - A) * rgb * alpha
        A = A + (1.0 - A) * alpha
        v = v + step * r
        
    return C
```

## Subtopics

### [Two-pass Adaptive Screen Sampling with Interval Partitioning](5460-two-pass-adaptive-screen-sampling-with-interval-partitioning.md)
This method accelerates volume integral calculation by partitioning the ray integration interval into $M$ equal pieces and using a second pass to interpolate or recompute integrals for pixels based on neighborhood coherence. It builds on [Adaptive Undersampling](5397-adaptive-undersampling.md) by increasing the adaptive undersampling factor, allowing fewer pixels to be processed in the first pass while maintaining image quality through spatial interpolation in the second pass. Compared to standard adaptive undersampling, it increases frames per second (FPS) by a factor of 1.5 on average, and provides further acceleration over two-pass adaptive screen sampling without interval partitioning.

## Operating characteristics
*   **Time Complexity:** Depends on data coherence. In the best case (high coherence), most pixels are interpolated in Pass 2. In the worst case, integrals are calculated for many intervals [S1].
*   **Memory Complexity:** Requires a G-buffer with $M+1$ textures of resolution $1/n$ of the viewport [S1, S2].
*   **Performance:** On a 3.4GHz Intel Core i7 2600, GTX 780 Ti, 1200x900 viewport, with $M=8$ and $\delta=0.05$, the method accelerates standard adaptive undersampling by a factor of 1.5 to 3.7 depending on the dataset [S1]. In tested datasets, 91.7% of pixels were interpolated, 5.7% required 1 interval calculation, 2.1% required 2, 0.1% required 3, and 0.4% required >3 intervals [S1, S2].
*   **Strengths:** Reduces redundant calculations by leveraging spatial coherence of interval integrals. Avoids recursive subdivision overhead on GPUs [S2].
*   **Limitations:** Efficiency decreases on low coherence datasets. Performance is reduced if many rays end early (e.g., bones), as intervals become short and harder to partition [S1].
*   **When to use:** High-resolution volume rendering where coherence exists between adjacent rays.
*   **When not to use:** Highly incoherent data or scenes with rapid geometry changes in screen space [S3, unofficial].

## Background
The method was proposed by Smirnova and Shubnikov in 2019, published in the Journal of WSCG [S1, S2]. It builds upon earlier adaptive undersampling techniques [S1]. The authors implemented it in Unity3D using OpenGL ES 3.1 and tested it on CT datasets (head, lungs, bones) [S1]. The technique addresses the computational expense of direct volume rendering, which is proportional to the number of pixels [S1].

## Sources
[S1] Fast Adaptive Undersampling for Volume Rendering, https://doi.org/10.24132/jwscg.2019.27.1.1, fetched 2026-10-26
[S2] Fast Adaptive Undersampling for Volume Rendering, http://wscg.zcu.cz/wscg2019/Full/A37-full.PDF, fetched 2026-10-27
[S3] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
