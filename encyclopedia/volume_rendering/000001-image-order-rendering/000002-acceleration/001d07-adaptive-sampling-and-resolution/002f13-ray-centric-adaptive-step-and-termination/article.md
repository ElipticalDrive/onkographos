# Ray-Centric Adaptive Step and Termination


## Overview
Ray-Centric Adaptive Step and Termination adjusts the integration step size $\Delta t$ along a ray based on local volume properties to skip empty space and refine detail near boundaries [G]. It differs from [Adaptive Sampling](1385-adaptive-sampling.md) by dynamically changing the step size within a single ray cast rather than changing the number of rays or blocks, and from [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by operating on explicit volumetric data structures rather than neural uncertainty estimates [G].

## Shared principles
Builds on [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md), this family varies the discretization of the rendering process by adjusting the integration step size $\Delta t$ along the ray parameter $t$ [G]. The step size is typically a function of the local scalar field value $f(\mathbf{x})$, its gradient $\nabla f(\mathbf{x})$, or the density $\rho(\mathbf{x})$ [G]. The general ray marching loop iterates $t$ from $t_{near}$ to $t_{far}$:
1. **Step Calculation:** Compute $\Delta t$ based on local metrics. For density-based methods, $\Delta t$ is often inversely proportional to density or gradient magnitude to maintain constant error bounds [G].
2. **Termination:** The loop terminates when the accumulated opacity $\alpha$ exceeds a threshold (e.g., $1.0$) or $t$ exceeds $t_{far}$ [G].
3. **Coherence:** To maintain GPU thread coherence, step sizes are often quantized or limited to a small set of discrete values, preventing divergence in warp execution [G].

## Subtopics

### [Adaptive Sampling](1385-adaptive-sampling.md)
Adaptive sampling is a rendering optimization that varies the density and distribution of samples along rays to reduce computational cost in low-detail regions while preserving quality in high-detail areas [G]. It differs from its sibling [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md) by focusing on static or semi-static volume fidelity rather than interactive frame-rate stabilization via global rate adjustment [G]. Choose it when visual fidelity in static scenes is prioritized over interactive responsiveness [G].

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Adaptive Sampling for Interaction is a technique to maintain interactivity during user input, such as transfer function editing, by lowering the volume sampling rate, then re-rendering at a high rate once the action is completed [G]. It differs from [Adaptive Sampling](1385-adaptive-sampling.md) by using a binary state switch (interaction vs. idle) rather than continuous error estimation [G]. Choose it when user-driven parameter editing requires immediate visual feedback [G].

### Adaptive Sampling with Uniform Fallback
Samples points along rays using adaptive sampling, with a fallback to uniform sampling for rays that receive too few samples [G]. The implementation details, including ray-AABB intersection, sample counting, and tensor concatenation, are not given in the sources.

## Background
The origin, specific historical development, and exact application domains of this family are not given in the sources. It is generally applied in real-time volume rendering to balance visual fidelity against computational cost [G].

## Sources
