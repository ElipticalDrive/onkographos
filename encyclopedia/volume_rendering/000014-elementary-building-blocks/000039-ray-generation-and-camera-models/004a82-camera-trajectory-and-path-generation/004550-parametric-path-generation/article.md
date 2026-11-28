# Parametric Path Generation

Also called: Parametric Camera Path Generation, Analytic Trajectory Generation

## Overview
Parametric Path Generation constructs camera trajectories by evaluating explicit mathematical functions of a scalar parameter (typically angle or time) to produce a sequence of camera poses [G]. It solves the problem of generating smooth, continuous novel views from a sparse set of training images by defining the geometry of the path analytically rather than interpolating between discrete keyframes [G]. It differs from [Sparse Trajectory Generation](7215-sparse-trajectory-generation.md) by relying on closed-form equations for position and orientation, and from [Synthetic Camera Trajectory Generation](7942-synthetic-camera-trajectory-generation.md) by deriving parameters directly from scene statistics (e.g., camera bounding boxes) rather than arbitrary artistic constraints [G].

## Shared principles
The family shares the mechanism of mapping a scalar parameter $\theta \in [0, 2\pi]$ (or a linear time interval) to a 4x4 transformation matrix $\mathbf{P}$ [G]. The core steps common to all children are:
1. **Scene Analysis**: Compute scene statistics (centroid, bounding box, or percentiles) from input training poses to define the spatial bounds of the trajectory [G].
2. **Parameterization**: Define position $\mathbf{c}(\theta)$ and orientation (look-at direction $\mathbf{z}(\theta)$, up vector $\mathbf{u}(\theta)$) as functions of $\theta$ [G].
3. **Pose Construction**: Assemble the rotation matrix $\mathbf{R}$ from orthogonal basis vectors and translation $\mathbf{t}$ into $\mathbf{P} = [\mathbf{R} | \mathbf{t}]$ [G].
4. **Sampling**: Evaluate the functions at $N$ evenly spaced intervals, excluding the final duplicate point to ensure a closed loop without redundancy [G].

## Subtopics

### [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md)
Creates a spiral trajectory by combining circular horizontal motion with vertical oscillation. It is distinct from [Elliptical Camera Path Generation](7627-elliptical-camera-path-generation.md) by using a constant radius in the horizontal plane (scaled by percentiles) and a sinusoidal vertical component, whereas the elliptical variant allows for non-circular horizontal bounds and linear interpolation of vertical bounds. It is chosen for forward-facing scenes where the camera orbits a central point while varying height.

### [Elliptical Camera Path Generation](7627-elliptical-camera-path-generation.md)
Generates a closed-loop elliptical trajectory by interpolating between computed lower and upper bounds in both horizontal and vertical axes. It differs from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by allowing the horizontal cross-section to be a non-circular ellipse (defined by 90th percentile bounds) and by using a phase-shifted vertical oscillation that is bounded by 10th/90th percentiles rather than a pure sine wave. It is chosen when the scene has anisotropic horizontal distribution or when a smoother, less oscillatory vertical transition is desired.

### [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md)
Rotates the camera around a fixed axis (Azimuth, Elevation, or Roll) at a constant distance from the center of rotation. It differs from the spiral and elliptical variants by restricting motion to a single planar rotation without vertical oscillation or radius variation. It is chosen for simple panoramic exports or when the scene geometry is centered and symmetric, requiring minimal parameter tuning.

## Background
Parametric path generation is a foundational technique in NeRF (Neural Radiance Fields) and volume rendering novel view synthesis [G]. The spiral path generation method was popularized by the original NeRF implementation, which used it to generate training views for forward-facing scenes [G]. The elliptical and orbit variants emerged as adaptations to handle different scene geometries and rendering pipelines, often found in open-source NeRF implementations and visualization tools [G].

## Sources
