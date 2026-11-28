# Elementary building blocks


## Overview
This family comprises the fundamental geometric and computational primitives required to construct a volume rendering pipeline. It addresses the foundational tasks of defining the viewing geometry (rays) and executing the rendering computations efficiently on modern hardware. It sets itself apart from sibling methods by providing the low-level infrastructure—ray parameterization and parallel execution models—upon which higher-level techniques like surface extraction or light transport are built.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model.

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model.

**Parallel Execution**
Computations are mapped to parallel threads. Each thread typically processes one or more pixels or samples. Memory access patterns are optimized for coalescing. Synchronization is used to manage data dependencies between threads.

## Subtopics

### [Ray generation and camera models](770-ray-generation-and-camera-models.md)
Defines the geometric mapping from 2D image coordinates to 3D rays. It establishes the viewing frustum and ray origins/directions. Choose this when you need to define the camera's position, orientation, and field of view.

### [Parallel and GPU primitives](771-parallel-and-gpu-primitives.md)
Provides the mechanisms for executing rendering tasks on parallel hardware (GPUs/CPUs). It covers thread mapping, memory hierarchy, and synchronization. Choose this when implementing the actual rendering loop to ensure performance.

## Background
The concept of ray generation is rooted in computer graphics since the advent of ray tracing [S2]. Parallel execution primitives have evolved with the rise of GPUs, becoming essential for real-time volume rendering [S1, S3].

## Sources
[S1] Parallel and GPU primitives notes
[S2] Ray generation and camera models notes
[S3] Ray generation and camera models notes
