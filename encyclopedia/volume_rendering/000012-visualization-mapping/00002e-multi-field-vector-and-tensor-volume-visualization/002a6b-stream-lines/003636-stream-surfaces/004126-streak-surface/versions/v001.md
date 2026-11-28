# Streak Surface


## Overview
Streak surfaces visualize time-dependent flows by tracking particles emerging from a seeding curve at different ages, modeling the region of flow passing through a domain [S1, unofficial]. Unlike stream surfaces which assume steady-state flow, streak surfaces capture the history of particle advection in unsteady fields [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A time-dependent velocity field $f(\mathbf{x}, t)$, a univariate seeding curve $c: [0, 1] \to \mathbb{R}^3$, and a current time $t$.
*   **Output:** An integral surface $S$ parameterized by age $r$ and curve index $s$.

**Data Structures:**
The surface is defined parametrically. No explicit mesh is required by the definition, though numerical implementations typically discretize $r$ and $s$ into a grid of points $(r_i, s_j)$ to form a point cloud or mesh [S2].

**Formulas:**
The streak surface $S$ at time $t$ is defined by the integral:
$$ S(r, s, t) = c(s) + \int_{t-r}^{t} f(S(r - (t - \tau), s, \tau), \tau) d\tau $$
Where:
*   $r \in [0, t]$ is the particle age parameter, identifying individual particle instances [S1, unofficial].
*   $s$ indexes the seeding curve $c(s)$ [S1, unofficial].
*   $f$ is the velocity field [S1, unofficial].
*   For a constant $s$, $S$ reduces to a streakline [S1, unofficial].

**Steps:**
1.  Define the seeding curve $c(s)$ for $s \in [0, 1]$.
2.  For each point on the curve $s$, compute the trajectory of particles emitted at different times.
3.  The position of a particle with age $r$ at current time $t$ is found by integrating the velocity field backwards from $t$ to $t-r$, starting from $c(s)$ [S1, unofficial].
4.  Numerical integration (e.g., Riemann sums or Runge-Kutta methods) is required as analytic integration is usually not possible [S1, unofficial].
5.  Collect points $S(r, s, t)$ for discrete values of $r$ and $s$ to form the surface representation [S2].

**Edge Cases and Pitfalls:**
*   **Numerical Integration:** Analytic integration is generally impossible; discretization errors depend on the step size in time and curve parameterization [S1, unofficial].
*   **Backwards Integration:** Streak surfaces can also be modeled using backwards integration to trace flow regions [S1, unofficial].

## Subtopics

### Time and Streak Surface Computation
Approaches for computing time and streak surfaces vary by hardware and dataset size. CPU-based parallel algorithms are used for treating large CFD datasets, while GPU-based approaches are designed for real-time visualization of smaller datasets [S2]. These methods often employ point-based algorithms with dense particle sets advected at interactive speeds, combined with point-based rendering, rather than explicit mesh generation [S2]. Visualization options include striped textures and advanced transparency via depth peeling [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
CPU-based approaches handle large datasets but are not real-time. GPU-based approaches enable real-time visualization for smaller datasets [S2].

**Strengths and Limitations:**
*   **Strengths:** Captures time-dependent flow history; suitable for both large offline analysis (CPU) and interactive exploration (GPU) [S2].
*   **Limitations:** Requires numerical integration; explicit mesh generation is optional but often avoided in favor of point-based rendering for performance [S2].

## Background
The concept of streaklines and streak surfaces was defined in the context of multi-field visualization [S1, unofficial]. Approaches for time and streak surface computation were presented by Krishnan et al. [21], Bürger et al. [5], and von Funck et al. [34] [S2].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
[S2] IRIS: Illustrative Rendering for Integral Surfaces, https://doi.org/10.1109/tvcg.2010.173, fetched 2026-10-27
