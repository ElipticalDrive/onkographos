# Elementary building blocks
## Overview
This family provides the fundamental geometric and computational primitives required to construct a volume rendering pipeline [G]. It addresses the foundational tasks of defining the viewing geometry (rays) and executing rendering computations efficiently on modern hardware [G]. It sets itself apart from sibling methods by providing the low-level infrastructure—ray parameterization and parallel execution models—upon which higher-level techniques are built [G].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Ray-primitive intersection](769-ray-primitive-intersection.md)
Computes geometric intersection between parametric rays and scene primitives to determine hit distances. It sets itself apart from [Ray-marching](52-ray-marching.md) by relying on exact analytical solutions rather than iterative stepping, and from [Ray-box intersection](776-ray-box-intersection.md) by encompassing a broader set of geometric shapes. Choose this when identifying which objects a viewing ray encounters for early rejection or spatial bounds.

### [Ray generation and camera models](770-ray-generation-and-camera-models.md)
Defines the geometric mapping from discrete 2D pixel coordinates to continuous 3D world-space rays. It sets itself apart from [Ray-primitive intersection](769-ray-primitive-intersection.md) by focusing solely on ray geometry creation rather than traversal. Choose this when establishing a valid view frustum and constructing camera basis vectors.

### [Parallel and GPU primitives](771-parallel-and-gpu-primitives.md)
Encompasses fundamental computational mechanisms to map volume rendering tasks onto parallel hardware architectures. It sets itself apart from its siblings by focusing on the execution model and data movement primitives rather than geometric algorithms or high-level rendering strategies. Choose this when distributing work across thousands of threads to solve computational bottlenecks.

## Background
The concept of ray generation is rooted in computer graphics since the advent of ray tracing [G]. Parallel execution primitives have evolved with the rise of GPUs, becoming essential for real-time volume rendering [G].

## Sources
[S1] Parallel and GPU primitives notes
[S2] Ray generation and camera models notes
[S3] Ray generation and camera models notes
