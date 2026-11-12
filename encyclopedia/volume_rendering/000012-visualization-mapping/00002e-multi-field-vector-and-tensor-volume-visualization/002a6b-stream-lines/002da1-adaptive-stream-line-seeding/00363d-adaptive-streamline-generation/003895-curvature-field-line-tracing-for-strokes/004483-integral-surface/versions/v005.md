# Integral Surface
Also called: Path Surface, Rake, Stream Surface

## Overview
An integral surface generalizes the concept of integral lines to higher-dimensional seeding geometries, creating a surface in time-varying velocity fields or a stream surface in steady flow [S1, unofficial]. It is defined by a univariate seeding curve (or "rake") and integrates the velocity field along trajectories emanating from this curve [S1, unofficial]. Unlike single integral lines, it captures the collective evolution of a continuous set of particles, serving as a fundamental structure for visualizing flow topology and particle advection [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A time-varying velocity field $v(\mathbf{x}, t)$, a univariate seeding curve $c: I \subseteq \mathbb{R} \to \mathbb{R}^n$, and a time interval $[t_0, t]$ [S1, unofficial].
- Output: A parametric surface $S(s, t)$ representing the integral surface [S1, unofficial].

**Data Structures:**
- The seeding curve $c(s)$ is typically represented as a sequence of points or a parametric function mapping the parameter $s \in I$ to $\mathbb{R}^n$ [S1, unofficial].
- The resulting surface is defined by parameters $s$ (along the rake) and $t$ (time) [S1, unofficial].

**Steps:**
1. Define the seeding curve $c(s)$ which serves as the initial locus of particles [S1, unofficial].
2. For each point $s$ on the curve, integrate the velocity field forward in time from $t_0$ to $t$ [S1, unofficial].
3. Compute the position $S(s, t)$ using the integral formulation [S1, unofficial]:
   $$ S(s, t) = c(s) + \int_{t_0}^{t} v(\tau, S(s, \tau)) \, d\tau $$
   where:
   - $S(s, t)$ is the position of the particle originating from $c(s)$ at time $t$ [S1, unofficial].
   - $c(s)$ is the initial position of the particle at time $t_0$ [S1, unofficial].
   - $v(\tau, S(s, \tau))$ is the velocity field evaluated at the particle's current position and time $\tau$ [S1, unofficial].
   - The integral accumulates the displacement over time [S1, unofficial].
4. The resulting set of points $\{S(s, t) \mid s \in I, t \in [t_0, T]\}$ forms the integral surface [S1, unofficial].

**Parameters:**
- $t_0$: Start time of integration. Not given in the sources.
- $t$: End time of integration. Not given in the sources.
- $I$: Domain of the seeding curve parameter $s$. Not given in the sources.

**Edge Cases and Pitfalls:**
- For a fixed time $t$, the lines on the integral surface are timelines [S1, unofficial].
- In time-varying fields, the lines on the integral surface are no longer tangential to the flow field and may self-intersect [S1, unofficial].
- Higher-dimensional seeding geometry generalizes these definitions to path volumes and time surfaces [S1, unofficial].

## Subtopics

### [Magnetosphere Field Line Tracing](12012-magnetosphere-field-line-tracing.md)
Generates synthetic magnetospheric field lines using an analytical dipole model and ionospheric seeding, outputting compact JSON for volumetric visualization. It differs by using specific geometric constraints and an analytical model rather than grid-based integration.

### [Particle Traces on 2D Surfaces](12772-particle-traces-on-2d-surfaces.md)
Represents 3D vector fields on arbitrary 2D surfaces using evenly spaced particle traces with hierarchical lattice seeding. It differs by projecting the field onto the surface and breaking traces into segments to resolve directional ambiguity.

## Operating characteristics
Not given in the sources.

## Background
The concept of integral surfaces, also known as path surfaces or rakes, is defined in the context of multi-field visualization and time-varying flow fields [S1, unofficial]. It generalizes the notion of integral lines to continuous seeding geometries, allowing for the visualization of particle advection from a curve rather than a point [S1, unofficial]. The definition is closely related to streak surfaces, which track particles emerging from a seeding location over time [S1, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
