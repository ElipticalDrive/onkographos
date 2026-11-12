# Sphere Tracing with SDFs

Also called: Raymarched SDFs, Implicit Surface Ray Marching

## Overview
Sphere Tracing with Signed Distance Functions (SDFs) is a ray-marching technique that renders implicit geometry by iteratively stepping along a ray using the distance to the nearest surface as the step size [S1, S2, S9643]. It solves the problem of rendering complex, continuous geometry without explicit polygonal meshes by leveraging the mathematical guarantee that no geometry exists within the distance value returned by the SDF [S1, S2, S9643]. It differs from its siblings, such as [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) and [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md), by relying on the exact Lipschitz property of the SDF for step sizing rather than hierarchical acceleration structures or multi-resolution sampling strategies.

## Shared principles
The method relies on the mathematical property of the Signed Distance Function (SDF). An SDF $f(\mathbf{p})$ maps a 3D point $\mathbf{p}$ to the shortest Euclidean distance to the nearest surface of the scene geometry [S2][S5, unofficial]. The sign indicates position: $f(\mathbf{p}) < 0$ inside, $f(\mathbf{p}) = 0$ on the surface, and $f(\mathbf{p}) > 0$ outside [S2, S5, S9643].

The core algorithm, often called sphere tracing, proceeds as follows for each pixel:
1.  **Ray Definition**: A ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector [G].
2.  **Initialization**: The accumulated distance $t$ is initialized to 0 (or a near-plane value) [S1, unofficial][S3, abstract only, unofficial].
3.  **Marching Loop**:
    *   Evaluate the scene SDF at the current ray position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to get the distance $d = f(\mathbf{p})$ [S1, S34, unofficial].
    *   **Termination Check**: If $d < \epsilon$ (a small convergence threshold, e.g., $0.001$), the ray has hit the surface [S1, S39, unofficial]. If $t > t_{max}$ (maximum ray distance), the ray has escaped [S1, S34, unofficial].
    *   **Step**: Advance the ray by the distance $d$: $t \leftarrow t + d$ [S1, S33, unofficial]. This step is "safe" because the SDF guarantees no geometry exists within distance $d$ of $\mathbf{p}$ [S50, unofficial].
4.  **Shading**: Upon hit, the surface normal $\mathbf{n}$ is estimated, typically via finite differences of the SDF: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S2055, S4619]. Lighting is then computed [S1, unofficial].

Scene composition is achieved by combining primitive SDFs using Boolean operations: Union is $\min(d_1, d_2)$, Intersection is $\max(d_1, d_2)$, and Subtraction is $\max(-d_1, d_2)$ [S35, S38, unofficial]. Transformations (rotation, translation, scaling) are applied to the query point $\mathbf{p}$ before evaluation or to the SDF function itself [S3850].

## Subtopics

### [Zeno Implementation](1544-zeno-implementation.md)
A rapid prototyping variant that simplifies geometry definition by requiring only a distance bound function and optionally replacing explicit normals with numerical gradient approximation.

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
An optimization for neural SDFs that enables efficient forward and backward passes for inverse graphics, avoiding prohibitive computational graphs.

### [Raymarched SDF Volume Rendering](3663-raymarched-sdf-volume-rendering.md)
Extends the method to incorporate shading, lighting, and fog calculations within the loop to produce final pixel color, rather than just binary hit/miss.

### [Sphere Tracing with Composite SDF and Adaptive Sampling](3867-sphere-tracing-with-composite-sdf-and-adaptive-sampling.md)
Handles non-solid primitives (lines, arcs) and uses variance-based adaptive sampling for global illumination convergence.

### [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md)
Applies fixed SDF-derived step sizes to volume integration (Beer's Law) rather than surface intersection, suitable for uniform density media.

### [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)
Uses domain repetition (modulo arithmetic) and iterative scaling to generate fractal voids, differing from fixed Boolean unions.

### [Sphere Tracing with Represent Hit](6684-sphere-tracing-with-represent-hit.md)
A deferred technique tracking a secondary "represent hit" proxy to handle transparency and aliasing at object edges.

### [Synchronized-tracing](8552-synchronized-tracing.md)
A tile-based pipeline for implicit volumes (blobtrees) that synchronizes GPU threads via a low-resolution A-buffer to enable coherent field evaluation.

### [Cone-Traced Supersampling](9466-cone-traced-supersampling.md)
An antialiasing technique that identifies contours via cone intersections and performs supersampling along the ray to minimize shading computations.

### [Kernel-Reflection Sequences](10186-kernel-reflection-sequences.md)
Generates fractal geometries using a fixed, non-recursive iterative evaluation of reflected SDFs, enabling continuous LOD.

### [Trace-based SDF Rendering](10785-trace-based-sdf-rendering.md)
Renders sparse voxel models by returning discrete voxel IDs alongside depth, enabling direct attribute lookup from spherical harmonic coefficients.

### Cube-tracing
An algorithm using the chessboard metric (or Manhattan metric) for distance computation, potentially resulting in more efficiently computed distances and ray intersection. This variant is noted as further research [S2].

### Acceleration Structures for SDF Evaluation
A set of optimization nodes embedded directly in hierarchical construction trees to reduce the computational cost of field function evaluation while preserving Lipschitz or conservative properties. It avoids external data structures like octrees by using proxy and continuous level of detail nodes, and applies normal warping to enhance surface details. This speeds up evaluation by up to three orders of magnitude and applies to both ray-surface intersection and polygonization [S3, abstract only, unofficial].

### Harnack Tracing
A method for visualizing surfaces encoded by harmonic functions (including angle-valued functions and those with singularities) by using conservative Harnack bounds on the growth of harmonic functions to determine step sizes. It avoids numerical issues common to generic root finding methods and can visualize surfaces directly from point clouds or polygon soup without linear solves or mesh extraction [S4, unofficial].

### Claybook SDF Construction
A method for fully real-time deformable SDF-based worlds, using a global dense SDF with mip levels for acceleration. It constructs five mip levels where each encodes distance values twice as great as the previous, using coarser levels for conservative travel. Modification is performed via 'brushes' applied to the dense volume, which allows locality-based re-evaluation. Compact data structures are incompatible due to static voxel count requirements [S5, unofficial].

### Aeva Analytical SDF Rendering
An approach that avoids discretization by sphere-tracing analytical SDFs at render time, using offline compilation of CSG trees into optimized shaders. Each shader correctly renders a portion of the scene with minimum SDF evaluations. It is not suited for real-time modification due to the infeasibility of runtime shader recompilation [S5, unofficial].

## Background
The method was introduced by John Hart in 1996 as "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" [S2]. It optimized the earlier ray-marching technique by utilizing the Lipschitz property of SDFs to guarantee safe step sizes, thereby accelerating convergence [S1, unofficial]. The technique has since been extended to differentiable rendering for neural networks [S1, unofficial], volume rendering [S1, unofficial][S2], and complex procedural geometries like fractals [S1, unofficial] and blobtrees [S1, unofficial][S2].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S3] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
[S4] (PDF) Accelerating Sphere Tracing - ResearchGate, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S5] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
