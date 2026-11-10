# Integral Surface

Also called: Path Surface, Rake, Stream Surface

## Overview
An integral surface (also known as a path surface or rake) generalizes the concept of a stream line to a continuous curve of seed points, generating a 2D manifold in 3D space [S1, unofficial]. In steady flows, it represents a stream surface; in time-varying flows, it represents a path surface [S1, unofficial]. It is distinct from single stream lines by integrating along a univariate seeding curve (rake) rather than a single point, and from streak surfaces by tracking particles released at different times from the rake rather than continuously from a fixed point [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A velocity field $v(x, t)$ (time-varying) or $v(x)$ (steady); a univariate seeding curve $c(s)$ mapping parameter $s \in I \subseteq \mathbb{R}$ to $\mathbb{R}^n$ [S1, unofficial].
*   **Output:** A parametric surface $S(s, t)$ [S1, unofficial].

**Data Structures:**
The surface is defined parametrically by two variables: $s$ (position along the rake) and $t$ (time or integration depth). No explicit mesh structure is required for the definition; implementation typically involves evaluating the integral for a grid of $(s, t)$ pairs.

**Steps:**
1.  Define the seeding curve $c(s)$ for $s \in I$.
2.  For each $s$, compute the trajectory of a particle starting at $c(s)$ at time $t_0$.
3.  Integrate the velocity field along this trajectory from $t_0$ to $t$.
4.  Sum the initial position $c(s)$ with the integrated displacement to obtain $S(s, t)$.

**Formula:**
For a time-varying velocity field $v$, the integral surface is defined as [S1, unofficial]:

$$ S(s, t) = c(s) + \int_{t_0}^{t} v(\tau, S(s, \tau)) \, d\tau $$

Where:
*   $S(s, t)$ is the position on the surface at rake parameter $s$ and time $t$.
*   $c(s)$ is the position on the seeding curve at parameter $s$.
*   $v(\tau, x)$ is the velocity vector at time $\tau$ and position $x$.
*   $t_0$ is the initial time.

If the seeding curve $c$ varies over time, the definition characterizes a generalized path surface [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_integral_surface(c, v, t0, t1, s_samples, t_samples):
    """
    Computes an integral surface given a seeding curve c and velocity field v.
    
    Args:
        c: Function mapping s -> (x, y, z)
        v: Function mapping (t, x, y, z) -> (vx, vy, vz)
        t0: Start time
        t1: End time
        s_samples: List of s values in I
        t_samples: List of t values in [t0, t1]
        
    Returns:
        Surface points as a 2D array of shape (len(s_samples), len(t_samples), 3)
    """
    surface = empty_array(shape=(len(s_samples), len(t_samples), 3))
    
    for i, s in enumerate(s_samples):
        # Initial position from the rake
        current_pos = c(s)
        surface[i, 0] = current_pos
        
        # Integrate from t0 to t1
        # Using a simple Euler method for demonstration; RK4 is preferred for accuracy
        dt = (t1 - t0) / (len(t_samples) - 1)
        current_time = t0
        
        for j in range(1, len(t_samples)):
            current_time += dt
            
            # Evaluate velocity at current position and time
            # Note: v depends on the current position S(s, tau), which is current_pos
            vel = v(current_time, current_pos)
            
            # Update position
            current_pos = current_pos + vel * dt
            
            surface[i, j] = current_pos
            
    return surface
```

**Edge Cases and Pitfalls:**
*   **Self-intersection:** Integral lines on a path surface may self-intersect and are no longer tangential to the flow field everywhere [S1, unofficial].
*   **Timeline Definition:** Lines on a path surface for fixed $t$ are timelines [S1, unofficial].
*   **Higher Dimensions:** Higher-dimensional seeding geometry generalizes these definitions to path volumes and time surfaces [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Strengths:** Provides a continuous visualization of flow topology originating from a line source, useful for identifying coherent structures or transport barriers.
*   **Limitations:** Susceptible to self-intersection in complex flows; requires accurate integration of the velocity field along the surface.

## Background
The concept is formally defined in the context of multi-field visualization and time-varying flow fields [S1, unofficial]. It generalizes the integral line to a rake (curve) of seed points.

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
