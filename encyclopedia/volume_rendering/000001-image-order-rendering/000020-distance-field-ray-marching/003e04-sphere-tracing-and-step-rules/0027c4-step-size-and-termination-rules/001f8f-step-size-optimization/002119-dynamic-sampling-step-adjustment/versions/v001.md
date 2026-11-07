# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Empty Space Step Doubling

## Overview
Dynamic Sampling Step Adjustment accelerates ray marching through empty or low-density regions by exponentially increasing the step size, thereby reducing the number of samples required to traverse sparse volumes [S1, unofficial]. It differs from [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) and [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md) by using a discrete doubling strategy rather than continuous estimation or gradient-based refinement, and from [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md) by not explicitly calculating geometric gaps [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, initial step size $s_0$, initial mip level offset $m_0$, and a volume texture $V$ with mipmaps.
**Outputs:** Composed color and opacity for the ray.

**Data Structures:**
- $V$: Multi-resolution volume texture. Access at mip level $m$ requires scaling coordinates by $2^{-m}$.
- $s$: Current step size.
- $m$: Current mip level offset.
- $N_{empty}$: Counter for consecutive empty samples.

**Algorithm:**
1. Initialize $s = s_0$, $m = m_0$, and $N_{empty} = 0$.
2. March along the ray:
   a. Sample the volume at current position $p$ using mip level $m$. Let density be $\rho = V(p, m)$.
   b. If $\rho < \epsilon$ (empty threshold):
      i. Increment $N_{empty}$.
      ii. If $N_{empty} \ge 2$ (consecutive empty samples), double the step size: $s \leftarrow 2s$.
      iii. Increase mip offset: $m \leftarrow m + 1$. [S1, unofficial]
   c. If $\rho \ge \epsilon$ (non-empty):
      i. Step back one step: $p \leftarrow p - s_{prev}$ (where $s_{prev}$ is the step size used in the previous iteration). [S1, unofficial]
      ii. Restore step size: $s \leftarrow s_0$. [S1, unofficial]
      iii. Restore mip offset: $m \leftarrow m_0$. [S1, unofficial]
      iv. Reset $N_{empty} = 0$.
   d. Advance position: $p \leftarrow p + s$.
3. Composite color using front-to-back blending with the sampled densities.

**Parameters:**
- $\epsilon$: Empty density threshold. Not given in the sources.
- $s_0$: Initial step size. Not given in the sources.
- $m_0$: Initial mip level. Not given in the sources.

**Edge Cases and Pitfalls:**
- The optimization provides small performance gains and may cause artifacts in some cases [S1, unofficial].
- Stepping back one step upon encountering density requires storing the previous position and step size. [G]
- Doubling the step size may skip thin features if the threshold $\epsilon$ is too high or the initial step $s_0$ is too large. [G]

## Subtopics

### Gradient-driven Sampling
A sampling strategy that refines step sizes according to the magnitude of the scalar field gradient $|\nabla s|$, which indicates local feature sharpness; smaller steps are taken near high-gradient regions resembling surfaces to capture transitions accurately [S2]. The step size is dynamically adjusted based on the gradient direction and a threshold, bounded by minimum and maximum distances to prevent under- or over-sampling [S2]. This method enhances detail preservation in boundary-heavy volumes, such as CT angiography data, while coarsening traversal in smooth interiors, achieving efficiency gains of 2-3x over uniform sampling without aliasing artifacts [S2].

## Operating characteristics
- **Time Complexity:** Reduces sample count in empty regions by a factor of 2 per doubling event. Overall speedup is modest.
- **Memory Complexity:** Requires storing previous step size and position. No additional memory for the volume itself.
- **Performance:** Provides small performance gains [S1, unofficial].
- **Strengths:** Simple to implement; effective for sparse volumes.
- **Limitations:** May cause artifacts [S1, unofficial]; small performance gains [S1, unofficial].
- **When to use:** Sparse volumes with large empty regions.
- **When not to use:** Dense volumes or when artifacts are unacceptable.

## Background
This technique is used in real-time volume rendering, such as in cloud rendering pipelines [S1, unofficial]. It is often combined with other optimizations like blue noise dithering and temporal upsampling to mitigate sampling artifacts [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
