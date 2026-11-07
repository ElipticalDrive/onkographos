# Coarse-to-Fine Sphere Tracing

Also called: Aggressive Marching, Multi-Resolution Sphere Tracing (strategy)

## Overview
Coarse-to-Fine Sphere Tracing is a rendering optimization that accelerates distance-field ray marching by initiating the trace at a lower image resolution and progressively refining it. It exploits the spatial coherence of rays, where neighboring rays in the early stages of the trace follow similar paths, allowing the algorithm to skip redundant computations at high resolution [S1]. Unlike standard sphere tracing which processes every pixel at full resolution from the start, this method reduces the total number of Signed Distance Function (SDF) queries required to converge [S1].

## How it works
The method combines a hierarchical resolution strategy with an "aggressive" step-size modification.

**Inputs:**
- Target image resolution $W \times H$.
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel.
- SDF function $f(\mathbf{p})$.
- Convergence threshold $\epsilon$ (typically derived from the minimal distance between 3D points of neighboring pixels) [S1].
- Maximum travel distance $t_{max}$ (implicit or explicit).

**Outputs:**
- Final hit distance $t$ or surface normal for each pixel at full resolution.

**Data Structures:**
- A 2D grid of ray states, where each state stores the current accumulated distance $t$, current position $\mathbf{p}$, and active status. The grid dimensions change dynamically.

**Algorithm Steps:**

1.  **Initialization:**
    Start with a ray grid at $1/4$ of the target resolution (e.g., $W/2 \times H/2$). Initialize each ray's position $\mathbf{p} = \mathbf{o}$ and accumulated distance $t = 0$ [S1].

2.  **Coarse Marching Loop:**
    Repeat the following until the grid resolution matches the target resolution:
    a.  **Aggressive Marching:** Perform 3 sphere tracing steps. In each step, query the SDF at the current position $\mathbf{p}_i$. Instead of using the standard step size $\Delta t = f(\mathbf{p}_i)$, use an amplified step size:
        $$ \Delta t = \alpha \cdot f(\mathbf{p}_i) $$
        where $\alpha = 1.5$ [S1]. Update the position:
        $$ \mathbf{p}_{i+1} = \mathbf{p}_i + \Delta t \cdot \mathbf{d} $$
        This aggressive sampling accelerates convergence, especially when the ray is far from the surface or when the angle between the surface normal and ray direction is small [S1]. Note that this may cause the ray to pass through the surface (entering $f(\mathbf{p}) < 0$) [S1].
    b.  **Resolution Split:** Double the resolution of the ray grid. Each existing ray splits into four new rays corresponding to the four sub-pixels in the higher resolution grid. The new rays inherit the position $\mathbf{p}$ and accumulated distance $t$ from the parent ray [S1].

3.  **Fine Convergence:**
    Once the full resolution grid is reached, continue standard sphere tracing (using $\alpha = 1.0$) for all active rays until they converge. Convergence is defined as $f(\mathbf{p}) < \epsilon$ [S1].

**Parameters:**
- $\alpha = 1.5$: The aggressive marching multiplier [S1].
- Steps per split: 3 marching steps before each resolution doubling [S1].
- Initial resolution: $1/4$ of target [S1].
- Total splits: 6 steps to go from $1/4$ to full resolution (since $2^6 = 64$, but the text specifies 6 steps for 1/4 to full, implying a specific hierarchy or counting method; strictly, $1/4 \to 1/2 \to 1 \to 2 \to 4 \to 8 \to 16$ is 5 doublings. The source states "After six steps... full resolution", which may include the initial setup or a specific implementation detail. We follow the source's count of 6 steps for the splitting phase) [S1].

**Edge Cases:**
- **Surface Penetration:** Because $\alpha > 1$, rays may overshoot the surface. The algorithm must handle negative SDF values correctly during the fine convergence phase [S1].
- **Convergence Threshold:** $\epsilon$ should be estimated based on the minimal distance between 3D points of neighboring pixels to ensure consistent convergence across resolutions [S1].

## Subtopics

### [Multi-Resolution Sphere Tracing](9465-multi-resolution-sphere-tracing.md)
A strategy where sphere tracing is applied at gradually increasing resolutions, leveraging the fact that rays have already been traced far along, reducing computation for the final high-resolution steps.

## Operating characteristics
- **Performance:** Significantly reduces the number of SDF evaluations compared to standard sphere tracing by avoiding high-resolution queries for rays that are still far from the surface [S1].
- **Complexity:** The number of rays grows exponentially ($4^k$) with each split, but the total work is dominated by the coarse stages where fewer rays are processed.
- **Limitations:** The aggressive marching ($\alpha=1.5$) can lead to overshooting, requiring careful handling of negative SDF values. It assumes high spatial coherence between neighboring rays, which may break down in complex, high-frequency scenes [S1].
- **When to use:** Ideal for rendering smooth, implicit surfaces where rays travel long distances before hitting geometry. Less effective for highly detailed, high-frequency SDFs where neighboring rays diverge quickly.

## Background
The method was introduced in the context of differentiable rendering of deep implicit signed distance functions. It was proposed to address the computational cost of standard sphere tracing by leveraging the similarity of ray paths in the early stages of the trace [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
