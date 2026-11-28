# Analytical and Differentiable Methods


## Overview
Analytical and Differentiable Methods evaluate Signed Distance Functions (SDFs) analytically at render time instead of discretizing them into grids, enabling either gradient-based optimization for inverse graphics or optimized real-time shader evaluation [S1, unofficial][S2, unofficial]. This sets them apart from grid-based or uniform sampling approaches by preserving continuous differentiability or minimizing SDF evaluations per ray through offline compilation [S1, unofficial][S2, unofficial].

## Shared principles
All variants in this family bypass precomputed distance grids, computing SDF values directly from analytical expressions or neural networks during ray marching [S1, unofficial][S2, unofficial]. They rely on the Lipschitz continuity of the underlying functions to guarantee safe step sizes and convergence [S2, unofficial]. Differentiable variants additionally ensure the entire forward and backward rendering pipeline is compatible with automatic differentiation frameworks [S1, unofficial].

## Subtopics

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
Differentiable Sphere Tracing (DIST) is a rendering algorithm for implicit SDFs represented by neural networks, enabling efficient forward and backward passes for inverse graphics optimization. It solves the problem of non-differentiable or computationally prohibitive ray marching in neural implicit representations by optimizing the rendering pipeline for GPU efficiency. It sets itself apart from naive sphere tracing by using a coarse-to-fine approach and aggressive ray marching to save computation, and from standard differentiable renderers by providing a specific gradient approximation that reduces memory consumption while maintaining training performance. Choose this for inverse graphics with neural SDFs. Link: [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md).

### Aeva Analytical SDF Rendering
An approach that avoids discretization by sphere-tracing analytical SDFs at render time, using offline compilation of CSG trees into optimized shaders [S2, unofficial]. It sets itself apart by analyzing CSG trees offline to compile a large set of shaders, each correctly rendering a portion of the scene with the minimum number of SDF evaluations per ray [S2, unofficial]. Choose this for real-time rendering of complex CSG geometries where offline compilation is acceptable. Pitfall: Not suited for real-time modification because continuously recompiling shaders at runtime is not feasible or effective for GPU hardware [S2, unofficial]. Presented by Aeva (2022) [S2, unofficial].

### Analytical Derivatives for Normals and Slopes
A technique using analytical derivatives of noise to compute fast normals and slopes, which helps speed up the raymarching process by adjusting step size based on slope [S2, unofficial]. It sets itself apart by leveraging the Lipschitz constant of noise functions to dynamically adjust step size, taking smaller steps for steeper slopes [S2, unofficial]. Choose this for raymarching noise-based terrains or fractals where normal computation is a bottleneck. Used in the 'Rainforest' Shadertoy example (2016) for raymarching a signed distance field of a tree-populated terrain [S2, unofficial].

## Background
Differentiable Sphere Tracing addresses the computational bottlenecks of neural implicit representations by enabling end-to-end gradient flow for inverse graphics [S1, unofficial]. Analytical SDF rendering was introduced by Aeva (2022) to enable real-time CSG evaluation without grid discretization, though it requires offline shader compilation [S2, unofficial]. Analytical derivatives for normals and slopes were demonstrated in 2016 Shadertoy examples to optimize raymarching performance for noise-based SDFs [S2, unofficial].

## Sources
[S1] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S2] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
