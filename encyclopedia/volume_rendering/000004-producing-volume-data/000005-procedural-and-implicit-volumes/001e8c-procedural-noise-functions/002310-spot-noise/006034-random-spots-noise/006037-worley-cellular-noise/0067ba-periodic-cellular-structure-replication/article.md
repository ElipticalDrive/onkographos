# Periodic Cellular Structure Replication

Also called: FRep cellular model, Periodic Cellular Microstructure Replication, Space Mapping Replication, Unit Cell Replication, infinite cellular structure generation

## Overview
Periodic Cellular Structure Replication is a method for generating infinite cellular microstructures by applying a periodic space mapping function to a base unit cell defined on a finite interval [S1]. It solves the problem of modeling regular lattices or truss-based microstructures without explicit geometric duplication, allowing for continuous representation of infinite domains [S1]. Unlike sibling methods that rely on discrete tiling or voxelization, this approach uses a continuous real function to map coordinates, enabling seamless blending and parametric control over the lattice geometry [S1].

## How it works
The method constructs an infinite structure by transforming input coordinates through a periodic replication function before evaluating a base geometric function [S1].

**Inputs and Outputs:**
*   **Input:** A continuous real function $f(x, y, z)$ defining the unit cell geometry, and a periodic replication function $g(t)$ [S1].
*   **Output:** A scalar field $r(x, y, z)$ representing the infinite cellular structure [S1].

**Data Structures:**
*   **Unit Cell Interval $I$:** Defined by bounds $x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$ [S1].
*   **Replication Function $g(t)$:** A periodic function mapping real numbers to $[0, 1]$ [S1].

**Steps:**
1.  Define the unit cell geometry using the continuous real function $f$ on the interval $I$ [S1].
2.  Select a periodic replication function $g(t)$, typically constrained to the range $[0, 1]$ [S1].
3.  For any input coordinate $(x, y, z)$, compute the mapped coordinates by applying $g$ to each axis, scaled by the interval dimensions, and offset by the minimum bounds [S1].
4.  Evaluate the unit cell function $f$ at the mapped coordinates to produce the final value [S1].

**Formula:**
The replicated structure is defined by the equation [S1]:

$$
r(x, y, z) = f \left( x_{min} + g(x) \cdot (x_{max} - x_{min }), \ y_{min} + g(y) \cdot (y_{max} - y_{min }), \ z_{min} + g(z) \cdot (z_{max} - z_{min }) \right)
$$

Where:
*   $f(x, y, z)$ is the continuous real function defining the unit cell geometry on interval $I$ [S1].
*   $g(t)$ is the periodic replication function, where $g(t) \in [0, 1]$ for all $t$ [S1].
*   $I$ is the bounding interval of the unit cell, defined as $x_{min} \le x \le x_{max}$, $y_{min} \le y \le y_{max}$, $z_{min} \le z \le z_{max}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def periodic_cellular_replication(x, y, z, f, g, bounds):
    """
    Computes the value of an infinite cellular structure at (x, y, z).
    
    Args:
        x, y, z: Input coordinates.
        f: Function defining the unit cell geometry.
        g: Periodic replication function mapping to [0, 1].
        bounds: Tuple (xmin, xmax, ymin, ymax, zmin, zmax).
        
    Returns:
        Scalar value r(x, y, z).
    """
    xmin, xmax, ymin, ymax, zmin, zmax = bounds
    
    mapped_x = xmin + g(x) * (xmax - xmin)
    mapped_y = ymin + g(y) * (ymax - ymin)
    mapped_z = zmin + g(z) * (zmax - zmin)
    
    return f(mapped_x, mapped_y, mapped_z)
```

**Parameters and Typical Values:**
*   $g(t)$: Common choices include the Sawtooth wave or Triangle wave [S1]. The Sawtooth wave is often implemented using Fourier series with a finite number of harmonics (e.g., 20 or 50) to approximate the discontinuity [S1].
*   $f$: Depends on the specific microstructure (e.g., spherical shell, rods) [S1].

**Edge Cases and Pitfalls:**
*   **Discontinuity:** Geometric connectivity discontinuities ($C^0$ discontinuity) can occur at the boundaries of replicated unit cells if the base function $f$ or the replication function $g$ is not continuous [S1].
*   **Artifacts:** When using Fourier series approximations for the Sawtooth wave, artifacts may appear in the neighborhood of discontinuities (Gibbs phenomenon) if the number of harmonics is insufficient [S1].

## Subtopics

### [Sawtooth Replication Function](19979-sawtooth-replication-function.md)
The fundamental component of this method, resulting in $C^0$-discontinuous functions on cell faces unless the unit cell possesses full connectivity.

### Irregular Microstructure Modelling
A procedure to model irregular microstructures like porous media by replicating basic pore shapes with variable parameters and pseudo-random variations [S1]. The procedure selects a basic pore shape such as a sphere or ellipsoid, replicates it in space with variable parameters, introduces pseudo-random variations of pore positions and sizes, and subtracts the pores from a given initial shape [S1]. The space mapping for a basic pore with defining function $F_p$ is computed as $x_0 = p_x \sin(q_x x)$, $y_0 = p_y \sin(q_y x)$, $z_0 = p_z \sin(q_z x)$, yielding $F_r = F_p(x_0, y_0, z_0)$ [S1]. The final microstructure is obtained via set-theoretic subtraction: $F_m(x, y, z) = F \setminus \alpha F_r$ [S1]. Parameters $p_x, p_y, p_z > 0$ specify linear scaling for the coordinates, while $q_x, q_y, q_z > 0$ control the replication frequency along each axis [S1]. $F$ defines the initial shape, $F_m$ defines the final microstructure, and $\alpha$ is a parameter in the subtraction operation [S1].

## Operating characteristics
*   **Complexity:** The computational cost is dominated by the evaluation of the base function $f$ and the replication function $g$ [S1]. It is $O(1)$ per sample point.
*   **Strengths:** Enables the modeling of infinite cellular structures and microstructures without explicit geometric duplication, supporting blending operations and parametric control [S1].
*   **Limitations:** Discontinuities at cell boundaries can cause visual or geometric artifacts if not handled by the choice of $g$ or $f$ [S1].

## Background
The method originates from the field of procedural function-based modeling of volumetric microstructures, specifically within the context of FRep (Function Representation) models [S1]. It was developed to address the limitations of BRep and voxel models, which require finite bounding boxes and explicit sets of unit cell copies, by providing a continuous space mapping approach for infinite lattices [S1].

## Sources
[S1] Procedural function-based modelling of volumetric microstructures, https://doi.org/10.1016/j.gmod.2011.03.001, fetched 2026-10-26
