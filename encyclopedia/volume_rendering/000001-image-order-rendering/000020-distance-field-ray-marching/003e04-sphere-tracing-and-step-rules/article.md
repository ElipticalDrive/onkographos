# Sphere Tracing and Step Rules

Also called: Ray Marching, Distance Field Ray Marching

## Overview
Sphere tracing is a geometric method for rendering implicit surfaces by iteratively stepping along a ray using the distance to the nearest surface as the step size [S1, S2, S9643]. It solves the problem of rendering complex, continuous geometry without explicit polygonal meshes by leveraging the mathematical guarantee that no geometry exists within the distance value returned by the Signed Distance Function (SDF) [S1, S2, S9643]. It differs from its siblings, such as [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) and [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md), by relying on the exact Lipschitz property of the SDF for step sizing rather than hierarchical acceleration structures or multi-resolution sampling strategies.

## Shared principles
The method relies on the mathematical property of the Signed Distance Function (SDF). An SDF $f(\mathbf{p})$ maps a 3D point $\mathbf{p}$ to the shortest Euclidean distance to the nearest surface of the scene geometry [S2, S5, unofficial]. The sign indicates position: $f(\mathbf{p}) < 0$ inside, $f(\mathbf{p}) = 0$ on the surface, and $f(\mathbf{p}) > 0$ outside [S2, S5, S9643].

The core algorithm, often called sphere tracing, proceeds as follows for each pixel:
1.  **Ray Definition**: A ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector [G].
2.  **Initialization**: The accumulated distance $t$ is initialized to 0 (or a near-plane value) [S1, S3, unofficial].
3.  **Marching Loop**:
    *   Evaluate the scene SDF at the current ray position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to get the distance $d = f(\mathbf{p})$ [S1, S34, unofficial].
    *   **Termination Check**: If $d < \epsilon$ (a small convergence threshold, e.g., $0.001$), the ray has hit the surface [S1, S39, unofficial]. If $t > t_{max}$ (maximum ray distance), the ray has escaped [S1, S34, unofficial].
    *   **Step**: Advance the ray by the distance $d$: $t \leftarrow t + d$ [S1, S33, unofficial]. This step is "safe" because the SDF guarantees no geometry exists within distance $d$ of $\mathbf{p}$ [S50, unofficial].
4.  **Shading**: Upon hit, the surface normal $\mathbf{n}$ is estimated, typically via finite differences of the SDF: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S2055, S4619]. Lighting is then computed [S1].

Scene composition is achieved by combining primitive SDFs using Boolean operations: Union is $\min(d_1, d_2)$, Intersection is $\max(d_1, d_2)$, and Subtraction is $\max(-d_1, d_2)$ [S35, S38, unofficial]. Transformations (rotation, translation, scaling) are applied to the query point $\mathbf{p}$ before evaluation or to the SDF function itself [S3850].

## Subtopics

### [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md)
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by computing valid Sphere Distance Bounds (SDB) and adaptive step sizes using a dynamic Bounding Volume Hierarchy (BVH) derived from the FBM structure itself. It reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness, by progressively evaluating octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach supports features like Adaptive Lipschitz constants, Sub-stepping, Early Stop+k, and Quadratic Taylor inclusion functions.

### [Step Size and Termination Rules](4132-step-size-and-termination-rules.md)
Step Size and Termination Rules define the logic for advancing a ray along its direction vector and determining when the ray-marching loop should halt. They solve the problem of balancing rendering performance (by skipping empty space) against visual fidelity (by avoiding missed geometry or aliasing). What sets this family apart from sibling methods like [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) or [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md) is that it focuses specifically on the scalar decision of $\Delta t$ and the boolean conditions for convergence, rather than on spatial acceleration structures or multi-resolution hierarchies. It builds on [Sphere Tracing and Step Rules](9643-sphere-tracing-and-step-rules.md).

### [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md)
Coarse-to-fine sphere tracing is an acceleration strategy for rendering deep implicit signed distance functions (SDFs) that reduces computational cost by initializing ray marching at a lower resolution and progressively refining it. It differs from standard single-resolution sphere tracing by leveraging the spatial coherence of nearby rays; since neighboring rays often traverse similar paths, information from coarse passes allows fine-resolution rays to start closer to the surface, significantly reducing the number of SDF evaluations required for convergence.

### [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md)
Sphere Tracing with Signed Distance Functions (SDFs) is a ray-marching technique that renders implicit geometry by iteratively stepping along a ray using the distance to the nearest surface as the step size. It solves the problem of rendering complex, continuous geometry without explicit polygonal meshes by leveraging the mathematical guarantee that no geometry exists within the distance value returned by the SDF. It differs from its siblings, such as [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) and [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md), by relying on the exact Lipschitz property of the SDF for step sizing rather than hierarchical acceleration structures or multi-resolution sampling strategies.

### [Bidirectional Sphere Tracing](7433-bidirectional-sphere-tracing.md)
Bidirectional sphere tracing is a robust surface extraction technique that mitigates the convergence failures of standard sphere tracing when rays miss surface edges or intersect surfaces non-orthogonally. Unlike standard sphere tracing, which relies on a single forward march that may stall or overshoot, this method uses forward and backward traces to narrow the candidate range of the surface position, then refines the zero-level set using additional solvers. It is distinct from [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) and [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md) by replacing the iterative step-size heuristic with a bounded search interval derived from two opposing traces.

### [Relaxed Sphere Tracing](9463-relaxed-sphere-tracing.md)
Relaxed sphere tracing is an acceleration technique for distance-field ray marching that increases the step size beyond the conservative SDF value to reduce iteration counts. It scales the step by a relaxation factor $\omega$, reverting to the standard safe step if the new sample indicates the ray has jumped over the surface (i.e., the unbounding spheres of consecutive steps are disjoint). It is distinguished from [Enhanced Sphere Tracing](3867-enhanced-sphere-tracing.md) by its simpler geometric assumption; while relaxed tracing uses a fixed scalar multiplier, enhanced tracing computes an optimal tangential step size for planar surfaces, offering superior convergence at the cost of higher per-step complexity.

### Triangle Inequality Enhancement for Sphere Tracing
#### definition
- [S1] An optimization to eliminate unnecessary distance computations for collections of objects by using the triangle inequality to bound the distance to an object based on previous evaluations and ray traversal.
- [S1] An optimization using the triangle inequality to reduce unnecessary distance evaluations, particularly effective for large assortments of objects.
#### pitfall
- [S1] If all primitives (e.g., 14 Bezier curves) are nearly equidistant from the eye, the triangle inequality does not significantly reduce unnecessary distance evaluations until sphere tracing has traversed much of each ray.
#### number
- [S1] For 9 spheres/plane, triangle inequality reduced execution time from 2:53 to 1:19 (46% relative time).
- [S1] For the zeno logo, triangle inequality reduced execution time from 26:29 to 19:23 (73% relative time).
- [S1] When combined with convexity, triangle inequality enhances ordinary sphere tracing by 60%.

### Chunked Sphere Tracing
#### step
- [S2, unofficial] Get total number of rays N from ray_bundle.origins.shape[0].
- [S2, unofficial] Initialize empty list chunk_colors.
- [S2, unofficial] Loop over chunks: for chunk_start in range(0, N, self._chunk_size).
- [S2, unofficial] Slice ray_bundle for current chunk: cur_ray_bundle = ray_bundle[chunk_start : chunk_start + self._chunk_size].
- [S2, unofficial] Call parent forward method: cur_color = super().forward(sampler, implicit_fn, cur_ray_bundle).
- [S2, unofficial] Append color to list: chunk_colors.append(cur_color).
- [S2, unofficial] Concatenate all chunk colors along dimension 0: color = torch.cat(chunk_colors, dim=0).
- [S2, unofficial] Return concatenated color.
#### parameter
- [S2, unofficial] chunk_size: maximum number of rays processed in a single forward pass.

## Background
The term "sphere tracing" and the geometric method for antialiased ray tracing of implicit surfaces were introduced by Turner Whitted in [Kay & Kajiya, 1986] [S1]. The method was presented as a geometric alternative to standard ray tracing, particularly suited for implicit surfaces defined by distance fields. It gained prominence for its ability to handle complex geometric modeling operations and its inherent support for antialiasing through symbolic prefiltering [S1].

## Sources
[S1] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
