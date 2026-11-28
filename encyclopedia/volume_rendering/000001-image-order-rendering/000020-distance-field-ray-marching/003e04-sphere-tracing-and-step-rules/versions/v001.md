# Sphere Tracing and Step Rules

Also called: Ray marching with SDFs, Distance field ray marching (core mechanism)

## Overview
Sphere tracing is an image-order rendering algorithm for implicit surfaces defined by Signed Distance Functions (SDFs) that finds the intersection between a camera ray and geometry without explicit meshes [6223-sphere-tracing-with-sdfs.md]. It sets itself apart from [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) by using exact geometric distance bounds rather than probabilistic density accumulation, and from [Terrain Raymarching](1357-terrain-raymarching.md) by supporting arbitrary 3D topology rather than height-field constraints [6223-sphere-tracing-with-sdfs.md]. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md).

## Shared principles
The method operates on the principle that a Signed Distance Function (SDF) $f(\mathbf{p})$ returns the shortest Euclidean distance from a query point $\mathbf{p} \in \mathbb{R}^3$ to the nearest surface of the scene geometry [S2, S5, S6, S12, unofficial]. The sign of the result indicates the point's position relative to the surface:
- $f(\mathbf{p}) < 0$: The point is inside the primitive.
- $f(\mathbf{p}) = 0$: The point lies exactly on the surface.
- $f(\mathbf{p}) > 0$: The point is outside the primitive.

The core algorithm iteratively advances a ray $r(t) = \mathbf{o} + t\mathbf{d}$ along its direction $\mathbf{d}$ by querying the SDF at the current position $\mathbf{p}_i$. The step size $\Delta t$ is set to the distance value returned by the SDF: $\Delta t = f(\mathbf{p}_i)$ [S2, S7, S14, unofficial][S26][S33, S48, unofficial]. This step size is "safe" because, by the definition of the distance field, no geometry exists closer than $f(\mathbf{p}_i)$ to $\mathbf{p}_i$; thus, the ray cannot overshoot or penetrate a surface [S8, S27, S49, unofficial]. The position is updated as $\mathbf{p}_{i+1} = \mathbf{p}_i + f(\mathbf{p}_i)\mathbf{d}$ [S14, S33, unofficial].

The loop terminates when one of the following conditions is met:
1.  **Surface Hit:** The distance value $f(\mathbf{p}_i)$ falls below a small convergence threshold $\epsilon$ (e.g., $0.001$), indicating the ray is sufficiently close to the surface [S1, S2, S14, S34, S39, unofficial].
2.  **Escape:** The accumulated distance $t$ exceeds a maximum travel distance $t_{max}$, indicating the ray has exited the scene or bounding volume without hitting geometry [S1, S2, S34, unofficial].
3.  **Max Steps:** A predefined maximum number of iterations is reached, preventing infinite loops in case of non-convergence [S1, S2, S39, unofficial].

Scene composition is achieved by combining primitive SDFs using Boolean operations. The union of two shapes with SDFs $f_1$ and $f_2$ is $\min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S1, S38, unofficial]. The intersection is $\max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S38, unofficial]. Subtraction (difference) is $\max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S38, unofficial]. Smooth blending between shapes can be achieved using smooth minimum functions, such as Inigo Quilez's `smoothMin(a, b, k)`: $h = \text{clamp}(k - |a-b|, 0, k) / k$; $\text{result} = \min(a,b) - h \cdot h \cdot k / 6$ [S9, S24, unofficial].

Transformations are applied to the query point before evaluation. Translation by vector $\mathbf{t}$ uses $f'(\mathbf{p}) = f(\mathbf{p} - \mathbf{t})$ [S35, unofficial]. Scaling by factor $s$ uses $f'(\mathbf{p}) = s \cdot f(\mathbf{p}/s)$ [S35, unofficial]. Domain repetition (tiling) is achieved by taking the modulo of the point coordinates: $f'(\mathbf{p}) = f(\mathbf{p} \pmod 1)$ [S35, unofficial].

Surface normals $\mathbf{n}$ at the hit point $\mathbf{p}$ are estimated via the gradient of the SDF: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S1, S10, unofficial]. Since analytical gradients are often unavailable or expensive, they are typically approximated using central finite differences with a small offset $\epsilon$ (e.g., $0.001$):
$$ \nabla f(\mathbf{p}) \approx \begin{bmatrix} f(\mathbf{p} + \epsilon \hat{x}) - f(\mathbf{p} - \epsilon \hat{x}) \\ f(\mathbf{p} + \epsilon \hat{y}) - f(\mathbf{p} - \epsilon \hat{y}) \\ f(\mathbf{p} + \epsilon \hat{z}) - f(\mathbf{p} - \epsilon \hat{z}) \end{bmatrix} $$
[S1, S35, S39, S45, S46, unofficial].

## Subtopics

### [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md)
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by computing valid Sphere Distance Bounds (SDB) and adaptive step sizes using a Bounding Volume Hierarchy (BVH) derived from the FBM structure itself. Unlike naive sphere tracing which evaluates the full FBM sum at every step, this approach reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md) by replacing explicit spatial acceleration structures with a procedural hierarchy that allows for early termination of octave summation and tighter step size estimation.

### [Step Size and Termination Rules](4132-step-size-and-termination-rules.md)
This family defines the logic for determining the distance $\Delta t$ advanced along a ray and the conditions under which the marching loop terminates. It solves the trade-off between rendering accuracy (avoiding missed geometry or aliasing) and computational efficiency (reducing sample counts in empty regions) [Parent]. It sets itself apart from sibling methods by providing the fundamental traversal mechanics that [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) (fixed intervals), [Sphere Tracing](1674-sphere-tracing.md) (SDF-bounded steps), and [Step Size Optimization](2054-step-size-optimization.md) (context-aware steps) implement.

### [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md)
Sphere Tracing is an image-order rendering algorithm for implicit surfaces defined by Signed Distance Functions (SDFs). It solves the problem of finding the intersection between a camera ray and complex, continuous geometry without requiring explicit polygonal meshes or bounding volume hierarchies. It sets itself apart from its sibling [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md) by using exact geometric distance bounds rather than probabilistic density accumulation, and from [Terrain Raymarching](1357-terrain-raymarching.md) by supporting arbitrary 3D topology rather than height-field constraints. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md).

### Chunked Sphere Tracing
Processes rays in batches to manage memory or compute limits.
- [S1, unofficial] Get total number of rays N from ray_bundle.origins.shape[0].
- [S1, unofficial] Initialize empty list chunk_colors.
- [S1, unofficial] Loop over chunks: for chunk_start in range(0, N, self._chunk_size).
- [S1, unofficial] Slice ray_bundle for current chunk: cur_ray_bundle = ray_bundle[chunk_start : chunk_start + self._chunk_size].
- [S1, unofficial] Call parent forward method: cur_color = super().forward(sampler, implicit_fn, cur_ray_bundle).
- [S1, unofficial] Append color to list: chunk_colors.append(cur_color).
- [S1, unofficial] Concatenate all chunk colors along dimension 0: color = torch.cat(chunk_colors, dim=0).
- [S1, unofficial] Return concatenated color.
- [S1, unofficial] chunk_size: maximum number of rays processed in a single forward pass.

## Background
Origin: The core sphere tracing algorithm was introduced by Turner Whitted in 1980 [S1, unofficial]. It is widely used in real-time graphics for rendering complex implicit surfaces, such as those generated by Fractal Brownian Motion (FBM) or neural networks, where explicit mesh generation is prohibitive [6223-sphere-tracing-with-sdfs.md].

## Sources
[S1] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
