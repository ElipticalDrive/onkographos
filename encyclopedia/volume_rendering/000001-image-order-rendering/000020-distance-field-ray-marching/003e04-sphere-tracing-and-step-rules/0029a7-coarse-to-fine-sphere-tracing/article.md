# Coarse-to-Fine Sphere Tracing

Also called: Low-Resolution Acceleration Phase, Aggressive Marching

## Overview
Coarse-to-fine sphere tracing is an acceleration strategy for rendering deep implicit signed distance functions (SDFs) that reduces computational cost by initializing ray marching at a lower resolution and progressively refining it [S1]. It differs from standard single-resolution sphere tracing by leveraging the spatial coherence of nearby rays; since neighboring rays often traverse similar paths, information from coarse passes allows fine-resolution rays to start closer to the surface, significantly reducing the number of SDF evaluations required for convergence [S1].

## How it works
The method combines a multi-resolution initialization phase with an "aggressive" stepping rule to accelerate convergence [S1].

**Inputs and Outputs:**
*   **Input:** Target image resolution $R \times R$ (e.g., 512x512), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and the scene SDF $f(\mathbf{p})$ [G].
*   **Output:** Final rendered image with converged surface positions and shading [G].

**Data Structures:**
*   **Ray State:** For each active ray, store current position $\mathbf{p}$, accumulated distance $t$, and convergence flag [G].
*   **Resolution Grid:** A 2D grid of rays [G]. Initially, the grid size is $R/2 \times R/2$ [S1].

**Parameters:**
*   **Initial Resolution Factor:** Start at $1/4$ of the target pixel count (e.g., $R/2 \times R/2$ for target $R \times R$) [S1].
*   **Steps per Split:** Perform 3 marching steps at the current resolution before splitting [S1].
*   **Aggressive Factor ($\alpha$):** A multiplier for the step size, typically $\alpha = 1.5$ [S1].
*   **Convergence Threshold ($\epsilon$):** A small value indicating surface hit. The source defines $\epsilon$ as the estimated minimal distance between the 3D points of two neighboring pixels at the current resolution [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Create a ray grid at resolution $R_{init} = R/2$ [S1].
    *   For each pixel in the coarse grid, compute the initial ray direction $\mathbf{d}$ and origin $\mathbf{o}$ [G].
    *   Initialize ray position $\mathbf{p} = \mathbf{o}$ and accumulated distance $t = 0$ [G].

2.  **Coarse-to-Fine Loop:**
    *   While current resolution $R_{curr} < R$:
        *   **Aggressive Marching (3 steps):**
            *   For $k = 1$ to 3:
                *   Evaluate SDF: $d = f(\mathbf{p})$ [S1].
                *   Check termination: If $d < \epsilon_{curr}$, mark ray as converged and skip further steps for this ray [S1].
                *   Update position: $\mathbf{p} \leftarrow \mathbf{p} + \alpha \cdot d \cdot \mathbf{d}$ [S1].
                *   Update accumulated distance: $t \leftarrow t + \alpha \cdot d$ [S1].
        *   **Resolution Split:**
            *   If $R_{curr} < R$:
                *   Double the resolution: $R_{next} = 2 \cdot R_{curr}$ [S1].
                *   For each ray in the current grid, spawn 4 new rays for the corresponding $2 \times 2$ block in the higher resolution grid [S1].
                *   Initialize the 4 child rays with the same position $\mathbf{p}$ and accumulated distance $t$ from the parent [G].
                *   Update $\epsilon_{next}$ based on the new pixel spacing [S1].
                *   Replace the current grid with the new grid [G].

3.  **Final Convergence:**
    *   Once at full resolution $R$, continue standard sphere tracing for any rays not yet converged [S1].
    *   For each non-converged ray:
        *   Evaluate SDF: $d = f(\mathbf{p})$ [G].
        *   Check termination: If $d < \epsilon_{final}$, converge [G].
        *   Step: $\mathbf{p} \leftarrow \mathbf{p} + d \cdot \mathbf{d}$ (using $\alpha=1$ for final precision) [G].
        *   Repeat until all rays converge or max iterations reached [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, pos, dir, t=0.0, converged=False):
        self.pos = list(pos)
        self.dir = list(dir)
        self.t = t
        self.converged = converged

def coarse_to_fine_sphere_tracing(target_res, sdf, alpha=1.5, max_iter=1000):
    # Initialize at 1/4 resolution (half width/height) [S1]
    curr_res = target_res // 2
    rays = []
    # Standard ray generation from camera origin [G]
    for y in range(curr_res):
        for x in range(curr_res):
            ox, oy, oz = 0.0, 0.0, -5.0
            dx, dy, dz = (x - curr_res/2) / curr_res, (y - curr_res/2) / curr_res, 1.0
            l = (dx*dx + dy*dy + dz*dz)**0.5
            rays.append(Ray([ox, oy, oz], [dx/l, dy/l, dz/l]))

    while curr_res < target_res:
        # 3 aggressive marching steps per resolution level [S1]
        for _ in range(3):
            for ray in rays:
                if ray.converged: continue
                dist = sdf(ray.pos)
                # Epsilon is the estimated minimal distance between neighboring 3D points [S1]
                # Approximated as current ray distance divided by resolution [G]
                eps = ray.t / curr_res if ray.t > 0 else 1e-4
                if dist < eps:
                    ray.converged = True
                    continue
                # Aggressive step: advance by alpha * dist [S1]
                ray.pos[0] += alpha * dist * ray.dir[0]
                ray.pos[1] += alpha * dist * ray.dir[1]
                ray.pos[2] += alpha * dist * ray.dir[2]
                ray.t += alpha * dist

        # Split resolution: double it, spawn 4 rays per parent [S1]
        new_rays = []
        for ray in rays:
            if ray.converged: continue
            for dy in (0, 1):
                for dx in (0, 1):
                    # Sub-pixel direction interpolation [G]
                    nd = [ray.dir[0] + dx*0.001, ray.dir[1] + dy*0.001, ray.dir[2]]
                    nl = (nd[0]**2 + nd[1]**2 + nd[2]**2)**0.5
                    new_rays.append(Ray(ray.pos, [v/nl for v in nd], ray.t))
        rays = new_rays
        curr_res *= 2

    # Final convergence at full resolution [S1]
    for ray in rays:
        if ray.converged: continue
        for _ in range(max_iter):
            dist = sdf(ray.pos)
            eps = ray.t / target_res if ray.t > 0 else 1e-4
            if dist < eps:
                ray.converged = True
                break
            # Standard step for final precision [G]
            ray.pos[0] += dist * ray.dir[0]
            ray.pos[1] += dist * ray.dir[1]
            ray.pos[2] += dist * ray.dir[2]
            ray.t += dist
    return rays
```

**Edge Cases and Pitfalls:**
*   **Surface Overshoot:** The aggressive factor $\alpha > 1$ can cause rays to overshoot the surface, entering the interior ($f(\mathbf{p}) < 0$). The algorithm must handle negative SDF values correctly, as the step size remains positive but the direction is still along the ray. This is noted as crucial for sampling space behind the surface [S1].
*   **Threshold Sensitivity:** The convergence threshold $\epsilon$ is resolution-dependent. If set too low, the aggressive phase may not terminate early enough; if too high, artifacts may appear. The source defines $\epsilon$ as the estimated minimal distance between neighboring 3D points [S1].
*   **Ray Divergence:** At very low resolutions, rays may diverge significantly. The 3-step limit ensures that the coarse approximation remains valid before splitting [S1].

## Subtopics

### [Multi-Resolution Sphere Tracing](9465-multi-resolution-sphere-tracing.md)
Multi-resolution sphere tracing is the specific implementation of the coarse-to-fine strategy where rendering occurs at progressively higher resolutions. It sets itself apart from single-resolution tracing by reducing the computational cost of the final high-resolution steps, as rays only need to trace the remaining short distance to the surface after the coarse phases. It is chosen when rendering scenes with large empty spaces or smooth surfaces where low-resolution approximations are sufficient for initial convergence.

### Low-Resolution Acceleration Phase
This variant refers to the optimization for primary rays in GPU scheduling that uses low-resolution sphere tracing for initial steps. It provides a speedup of 30-50% over straightforward per-pixel ray tracing by deferring expensive high-resolution SDF evaluations until necessary [S2]. It is chosen when GPU occupancy is a bottleneck and the scene geometry allows for coarse approximations without visible artifacts.

## Operating characteristics
*   **Time Complexity:** The method reduces the average number of SDF evaluations per pixel compared to standard sphere tracing. By starting at $1/4$ resolution, the initial steps are cheaper, and the aggressive stepping reduces the total iterations needed for convergence [S1, S2].
*   **Memory Complexity:** Requires storing ray states for the current resolution grid. Memory usage scales with the number of active rays, which is highest at full resolution [G].
*   **Performance:** The aggressive marching ($\alpha=1.5$) accelerates convergence, especially when the ray is far from the surface or when the angle between the surface normal and ray direction is small (ill-posed conditions) [S1].
*   **Strengths:** Significant speedup for deep implicit surfaces; handles large scenes efficiently by leveraging spatial coherence [S1].
*   **Limitations:** May overshoot surfaces due to aggressive stepping, requiring careful handling of negative SDF values. Artifacts may occur if the convergence threshold is not properly scaled with resolution [S1].
*   **When to Use:** Ideal for rendering complex implicit surfaces (e.g., fractals, deep SDFs) where standard sphere tracing is too slow [S1].
*   **When Not to Use:** Not suitable for scenes with high-frequency details that require immediate high-resolution sampling, or when strict surface accuracy is required without post-processing [G].

## Background
Coarse-to-fine sphere tracing was introduced in the context of rendering deep implicit signed distance functions. The method leverages the property that nearby rays march similarly, allowing information from coarse passes to accelerate fine passes [S1]. The aggressive stepping strategy was proposed to further speed up the marching process, particularly for ill-posed conditions [S1]. The technique is part of the broader field of distance-field ray marching and sphere tracing [9643-sphere-tracing-and-step-rules.md].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
