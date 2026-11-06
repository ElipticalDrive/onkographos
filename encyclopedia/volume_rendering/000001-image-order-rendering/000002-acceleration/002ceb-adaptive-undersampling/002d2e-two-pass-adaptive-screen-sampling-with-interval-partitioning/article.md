# Two-pass Adaptive Screen Sampling with Interval Partitioning

Also called: Two-pass Adaptive Undersampling with Interval Partitioning, Two-pass adaptive screen sampling plus partitioning the interval of integration into M pieces

## Overview
This method accelerates volume integral calculation by partitioning the ray integration interval into $M$ equal pieces and using a second pass to interpolate or recompute integrals for pixels based on neighborhood coherence [S1]. It builds on [Adaptive Undersampling](5397-adaptive-undersampling.md) by increasing the undersampling factor through interval partitioning, allowing pixels to reuse integral data from neighbors rather than integrating over the entire ray [S1]. Unlike standard adaptive undersampling, it explicitly manages the granularity of integration steps to balance interpolation accuracy against recomputation cost [S1].

## How it works
**Inputs:** 3D volume texture (normalized to unit cube), transfer function, ray origin $s$, ray far-plane intersection $f$, maximum intervals $M$, color difference threshold $\delta$ [S1].
**Outputs:** Final pixel color $C$ [S1].

**Data Structures:**
*   **G-buffer ($G$):** Stores integration results per pixel [S1].
    *   $G[0].rgb$: Final pixel color [S1].
    *   $G[0].a$: Distance to the start of the volume (entry point) [S1].
    *   $G[i]$ for $i > 0$: Intermediate integral results for the $i$-th interval, storing `rgb` (accumulated color contribution) and `a` (accumulated alpha contribution) [S1].
    *   Size: $M+1$ elements per pixel [S1].

**Parameters:**
*   $M$: User-selected maximum number of intervals for partitioning the ray [S1]. Typical value: 8 [S1].
*   $\delta$: Threshold for color difference to determine if bilinear interpolation is sufficient [S1]. Typical value: 0.05 [S1].

**Algorithm Steps:**

1.  **Pass 1 (Coarse Sampling):**
    *   For a subset of pixels (e.g., one per $n \times n$ block), calculate the volume integral over $M$ equal-length intervals [S1].
    *   Initialize step size: $step = \sqrt{3} / M$ [S1]. The factor $\sqrt{3}$ accounts for the maximum ray length in the unit cube texture space [S1].
    *   Calculate maximum number of full steps: $imax = \lfloor length(f - s) / step \rfloor$ [S1].
    *   Calculate length of the last partial interval: $lastStep = length(f - s) - step \times imax$ [S1].
    *   Normalize ray direction: $r = normalize(f - s)$ [S1].
    *   Find entry point: $G[0].a = GetDistanceForStart(s, f)$ [S1]. If $G[0].a \le 0$, set $G[0].rgb = BackgroundColor$ and terminate [S1].
    *   Initialize accumulators: $C = 0$, $A = 0$ [S1].
    *   Set ray start: $v = s + G[0].a \cdot r$ [S1].
    *   Loop $i$ from 0 to $imax - 1$ while $A < 1$:
        *   Compute interval integral: $G[i+1] = VolumeIntegral(v, step)$ [S1].
        *   Update color: $C = C + (1 - A) \cdot G[i+1].rgb \times G[i+1].a$ [S1].
        *   Update alpha: $A = A + (1 - A) \cdot G[i+1].a$ [S1].
        *   Advance ray: $v = v + step \cdot r$ [S1].
    *   Handle last shorter interval if $A < 1$:
        *   Compute $G[imax+1] = VolumeIntegral(v, lastStep)$ [S1].
        *   Update $C$ and $A$ using the same recurrence relations [S1].
    *   Store final color: $G[0].rgb = C$ [S1].

2.  **Pass 2 (Refinement/Interpolation):**
    *   For pixels not processed in Pass 1 (or requiring refinement), check neighborhood coherence using the four neighboring pixels' G-buffer data ($G_i$, $i \in [0,3]$) [S1].
    *   **Optimization:** Accelerate start point calculation by casting the ray from $s + r \cdot \min\{G_i[0].a\}$ instead of $s$, using the minimum start distance from the 4 neighboring pixels [S1].
    *   Check coherence: If $\max_{neighbors} |Color_i - Color_{current}| < \delta$, use bilinear interpolation of neighbor colors for $G[0].rgb$ [S1].
    *   If coherence is insufficient, compute integrals for $M$ intervals:
        *   Initialize $C=0, A=0$, set $G[0].a = \min\{G_i[0].a\}$, and set $v = s + G[0].a \cdot r$ [S1].
        *   Loop $k$ from 0 to $imax - 1$ while $A < 1$:
            *   Compute integral $I = VolumeIntegral(v, step)$ [S1].
            *   Update $C$ and $A$ using recurrence relations [S1].
            *   Advance $v$ [S1].
        *   If $A < 1$, compute final interval $I = VolumeIntegral(v, lastStep)$ and update $C, A$ [S1].
    *   Store final result in $G[0].rgb$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def normalize(v):
    l = length(v)
    return (v[0]/l, v[1]/l, v[2]/l)

def get_distance_for_start(s, f, volume, tf):
    """Finds distance to first non-zero transfer function value along ray s->f."""
    ray = normalize((f[0]-s[0], f[1]-s[1], f[2]-s[2]))
    dist = 0.0
    step = length((f[0]-s[0], f[1]-s[1], f[2]-s[2])) / 100
    for _ in range(100):
        pos = (s[0]+dist*ray[0], s[1]+dist*ray[1], s[2]+dist*ray[2])
        if tf(pos) > 0.0:
            return dist
        dist += step
    return length((f[0]-s[0], f[1]-s[1], f[2]-s[2]))

def volume_integral(v, step, volume, tf):
    """Computes front-to-back compositing over a segment of length `step`."""
    ray = normalize((0.0, 0.0, 0.0)) # Direction is implicit in caller
    samples = max(1, int(step * 10))
    seg_step = step / samples
    c_rgb = (0.0, 0.0, 0.0)
    c_a = 0.0
    for _ in range(samples):
        pos = (v[0], v[1], v[2])
        density = volume(pos)
        alpha = tf(pos) * density * seg_step
        if alpha > 0.0:
            c_rgb = (c_rgb[0] + (1.0 - c_a) * alpha * density,
                     c_rgb[1] + (1.0 - c_a) * alpha * density,
                     c_rgb[2] + (1.0 - c_a) * alpha * density)
            c_a = c_a + (1.0 - c_a) * alpha
        v = (v[0], v[1], v[2]) # Direction handled by caller's v update
    return {'rgb': c_rgb, 'a': c_a}

def bilinear_interpolate(colors, u=0.5, v=0.5):
    """Standard 2D bilinear interpolation for 4 neighbors."""
    tl, tr, bl, br = colors
    top = (1.0 - u) * tl + u * tr
    bot = (1.0 - u) * bl + u * br
    return (1.0 - v) * top + v * bot

def pass1(s, f, M, volume, tf):
    dist_start = get_distance_for_start(s, f, volume, tf)
    if dist_start <= 0.0:
        return {'rgb': (0.0, 0.0, 0.0), 'a': 0.0}
    
    G = [None] * (M + 2)
    G[0] = {'rgb': (0.0, 0.0, 0.0), 'a': dist_start}
    
    ray_len = length((f[0]-s[0], f[1]-s[1], f[2]-s[2]))
    step = math.sqrt(3.0) / M
    imax = int(math.floor(ray_len / step))
    last_step = ray_len - step * imax
    r = normalize((f[0]-s[0], f[1]-s[1], f[2]-s[2]))
    
    v = (s[0] + dist_start * r[0], s[1] + dist_start * r[1], s[2] + dist_start * r[2])
    C = (0.0, 0.0, 0.0)
    A = 0.0
    
    for i in range(imax):
        if A >= 1.0: break
        interval_result = volume_integral(v, step, volume, tf)
        C = (C[0] + (1.0 - A) * interval_result['rgb'][0] * interval_result['a'],
             C[1] + (1.0 - A) * interval_result['rgb'][1] * interval_result['a'],
             C[2] + (1.0 - A) * interval_result['rgb'][2] * interval_result['a'])
        A = A + (1.0 - A) * interval_result['a']
        G[i + 1] = interval_result
        v = (v[0] + step * r[0], v[1] + step * r[1], v[2] + step * r[2])
        
    if A < 1.0:
        interval_result = volume_integral(v, last_step, volume, tf)
        C = (C[0] + (1.0 - A) * interval_result['rgb'][0] * interval_result['a'],
             C[1] + (1.0 - A) * interval_result['rgb'][1] * interval_result['a'],
             C[2] + (1.0 - A) * interval_result['rgb'][2] * interval_result['a'])
        A = A + (1.0 - A) * interval_result['a']
        G[imax + 1] = interval_result
        
    G[0]['rgb'] = C
    return G

def pass2(s, f, M, delta, neighbors_G, volume, tf):
    neighbor_colors = [n[0]['rgb'] for n in neighbors_G]
    # Compute max scalar color difference across RGB channels
    max_diff = max(max(abs(c[j] - neighbor_colors[0][j]) for j in range(3)) for c in neighbor_colors)
    
    if max_diff < delta:
        G[0]['rgb'] = bilinear_interpolate(neighbor_colors)
        return G[0]
    
    min_start_dist = min(n[0]['a'] for n in neighbors_G)
    G = [None] * (M + 2)
    G[0] = {'rgb': (0.0, 0.0, 0.0), 'a': min_start_dist}
    
    ray_len = length((f[0]-s[0], f[1]-s[1], f[2]-s[2]))
    step = math.sqrt(3.0) / M
    imax = int(math.floor(ray_len / step))
    last_step = ray_len - step * imax
    r = normalize((f[0]-s[0], f[1]-s[1], f[2]-s[2]))
    
    v = (s[0] + min_start_dist * r[0], s[1] + min_start_dist * r[1], s[2] + min_start_dist * r[2])
    C = (0.0, 0.0, 0.0)
    A = 0.0
    
    for k in range(imax):
        if A >= 1.0: break
        interval_result = volume_integral(v, step, volume, tf)
        C = (C[0] + (1.0 - A) * interval_result['rgb'][0] * interval_result['a'],
             C[1] + (1.0 - A) * interval_result['rgb'][1] * interval_result['a'],
             C[2] + (1.0 - A) * interval_result['rgb'][2] * interval_result['a'])
        A = A + (1.0 - A) * interval_result['a']
        G[k + 1] = interval_result
        v = (v[0] + step * r[0], v[1] + step * r[1], v[2] + step * r[2])
        
    if A < 1.0:
        interval_result = volume_integral(v, last_step, volume, tf)
        C = (C[0] + (1.0 - A) * interval_result['rgb'][0] * interval_result['a'],
             C[1] + (1.0 - A) * interval_result['rgb'][1] * interval_result['a'],
             C[2] + (1.0 - A) * interval_result['rgb'][2] * interval_result['a'])
        A = A + (1.0 - A) * interval_result['a']
        G[imax + 1] = interval_result
        
    G[0]['rgb'] = C
    return G[0]
```

**Edge Cases and Pitfalls:**
*   **High/Low Coherence:** Performance decreases if data has high coherence (many pixels interpolated, reducing partitioning benefit) or low coherence (many pixels require full recomputation) [S1]. Optimal performance is at medium coherence [S1].
*   **Early Ray Termination:** If a significant number of rays end early (e.g., bones), acceleration factor decreases because intervals are short and harder to partition [S1].
*   **G-buffer Limits:** Hardware limits the number of G-buffer elements [S1]. On GTX 780 Ti, max is 8, limiting $M$ to 8 unless components are packed [S1].

## Subtopics

### Visibility Prefiltering
A technique to reduce correlated artifacts from Riemann sums by replacing the single blocker depth $S[\gamma]$ with a distribution of depths, approximating the percentage of visibility that changes along $\beta$ [S2]. For each rectified $\gamma$ coordinate, it samples blocker depth at four sub-intervals, clusters nearby depths into modes, and inserts these modes into the termination depths structure as fractional updates [S2]. Basic Riemann sums require approximately 10x the number of samples for equal visual quality compared to Monte Carlo sampling; prefiltering reduces the required light rays [S2].

## Operating characteristics
*   **Time Complexity:** Reduces the number of full volume integrations by leveraging interpolation and interval partitioning [S1]. Average FPS increase by a factor of 1.5 compared to standard adaptive undersampling [S1]. Further acceleration of 1.6-2.1x over two-pass adaptive screen sampling without interval partitioning [S1].
*   **Memory Complexity:** Requires a G-buffer of size $M+1$ per sampled pixel [S1]. Limited by hardware G-buffer size (e.g., 8 elements) [S1].
*   **Measured Performance:** On a 3.4GHz Intel Core i7 2600, 4.0GB RAM, NVidia GForce GTX 780 Ti, resolution 1200x900, dataset 256x256x256: Standard Adaptive Undersampling (A): 32 FPS [S1]. Two-pass Adaptive Screen Sampling (B): 72 FPS [S1]. Proposed Method (C): 118 FPS [S1]. Acceleration C/A = 3.69, C/B = 1.64 [S1].
*   **Scaling:** Increasing $M$ from 8 to 16 (by storing two integrals in one G-buffer element) increases performance by ~21% on average, but further increases are considered ineffective [S1].
*   **Strengths:** Effective for spatially coherent data with transfer functions that exclude early ray termination [S1].
*   **Limitations:** Performance degrades with high coherence (less gain) or low coherence (more recomputation). Less effective for data with significant early ray termination (e.g., bones) [S1].

## Background
The method was proposed in "Fast Adaptive Undersampling for Volume Rendering" (2019) [S1]. It was implemented in Unity3D using OpenGL ES 3.1 [S1]. The approach extends previous work on two-pass adaptive screen sampling by introducing interval partitioning to better manage the trade-off between interpolation accuracy and computational cost [S1].

## Sources
[S1] Fast Adaptive Undersampling for Volume Rendering, http://wscg.zcu.cz/wscg2019/Full/A37-full.PDF, fetched 2026-10-27
[S2] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
