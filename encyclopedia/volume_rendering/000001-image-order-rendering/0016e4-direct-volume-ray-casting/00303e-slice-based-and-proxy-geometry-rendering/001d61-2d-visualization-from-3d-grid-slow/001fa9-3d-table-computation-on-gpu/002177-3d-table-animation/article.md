# 3D Table Animation

Also called: Three-dimensional Table Animation, solid space animation, volume control tables

## Overview
3D Table Animation is a method for controlling the movement and shape of volumetric gases by storing control values in a 3D table surrounding the gas volume in world space [S1, unofficial]. It solves the problem of directing gas flow by interpolating values from lattice points to determine sample-specific control parameters [S1, unofficial]. It is distinguished from its parent method, [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md), by its specific application to gas animation via functional flow fields and vector fields, rather than general volume shadowing or density computation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D table of control values stored at lattice points in world space; a sample point in screen space [S1, unofficial].
*   **Output:** Interpolated control values (e.g., direction vector, density scaling, velocity) for the sample point [S1, unofficial].

**Data Structures:**
*   **3D Table:** A 3D grid where each entry stores control data [S1, unofficial].
*   **Lattice Points:** The vertices of the table grid [S1, unofficial].
*   **Parallelepiped:** The volume defined by the eight lattice points surrounding a given sample point in table space [S1, unofficial].
*   **Indices:** Entries are defined by integer indices $(i, j, k)$ [G].

**Steps:**
1.  **Map Screen to World:** Convert the 3D screen space coordinate of the ray sample point into world space coordinates $(x_w, y_w, z_w)$ [S1, unofficial].
2.  **Map World to Table Space:** Calculate the location of the point within the 3D table $(x_t, y_t, z_t)$ using the world space position and the table's spatial parameters [S1, unofficial].
3.  **Identify Surrounding Entries:** Determine the eight table entries forming the parallelepiped surrounding the calculated table coordinate [S1, unofficial].
4.  **Interpolate:** Perform tri-linear interpolation on the values from the eight surrounding entries to determine the final control value [S1, unofficial].

**Formulas:**
The location within the table $(x_t, y_t, z_t)$ is determined by the point in world space $(x_w, y_w, z_w)$ [S1, unofficial]:

$$
x_t = (x_w - x_{start}) \cdot s_x
$$
$$
y_t = (y_w - y_{start}) \cdot s_y
$$
$$
z_t = (z_w - z_{start}) \cdot s_z
$$

Where:
*   $(x_{start}, y_{start}, z_{start})$ is the location in world space of the starting table entry (origin of the table grid) [S1, unofficial].
*   $s_x, s_y, s_z$ is the inverse of the step size between table elements in each dimension (i.e., $1/\Delta x, 1/\Delta y, 1/\Delta z$) [S1, unofficial].

**Parameters:**
*   **Table Bounds:** Defined by the starting table entry location and the step size (resolution) in each dimension [S1, unofficial].
*   **Interpolation Method:** Tri-linear interpolation is normally sufficient [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Sampling:** The formula for table location involves mapping world space to table space [S1, unofficial]. Incorrect mapping leads to sampling outside the table bounds [S1, unofficial]. Implementers must clamp or handle out-of-bounds coordinates [G].
*   **Precision:** Floating-point precision in the mapping step can affect which lattice points are selected for interpolation [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def get_table_value(sample_screen_pos, camera, table_origin, table_inv_step, table_data, table_dims):
    """
    sample_screen_pos: (x, y, z) in screen space
    camera: object with method to convert screen to world
    table_origin: (x_start, y_start, z_start) in world space
    table_inv_step: (sx, sy, sz) inverse step sizes
    table_data: 3D array of control values
    table_dims: (N, M, P) dimensions of the table
    """
    # 1. Map screen to world
    world_pos = camera.screen_to_world(sample_screen_pos)
    x_w, y_w, z_w = world_pos

    # 2. Map world to table space
    x_t = (x_w - table_origin[0]) * table_inv_step[0]
    y_t = (y_w - table_origin[1]) * table_inv_step[1]
    z_t = (z_w - table_origin[2]) * table_inv_step[2]

    # 3. Identify surrounding entries and clamp to valid range
    i = int(math.floor(x_t))
    j = int(math.floor(y_t))
    k = int(math.floor(z_t))
    i = max(0, min(i, table_dims[0] - 2))
    j = max(0, min(j, table_dims[1] - 2))
    k = max(0, min(k, table_dims[2] - 2))

    # 4. Interpolate (Tri-linear)
    fx = x_t - i
    fy = y_t - j
    fz = z_t - k

    v000 = table_data[i, j, k]
    v100 = table_data[i+1, j, k]
    v010 = table_data[i, j+1, k]
    v110 = table_data[i+1, j+1, k]
    v001 = table_data[i, j, k+1]
    v101 = table_data[i+1, j, k+1]
    v011 = table_data[i, j+1, k+1]
    v111 = table_data[i+1, j+1, k+1]

    v00 = lerp(v000, v100, fx)
    v01 = lerp(v010, v110, fx)
    v10 = lerp(v001, v101, fx)
    v11 = lerp(v011, v111, fx)

    v0 = lerp(v00, v01, fy)
    v1 = lerp(v10, v11, fy)

    final_value = lerp(v0, v1, fz)
    return final_value

def lerp(a, b, t):
    return a + t * (b - a)
```

## Subtopics

### Functional Flow Field Tables
A type of 3D table where each entry contains a reference to a function (or list of functions) that calculates direction, density scaling, velocity, and usage percentage for gas movement [S1, unofficial]. It is chosen when choreographed animation of gases is required, offering infinite detail in motion compared to fixed-resolution vector field tables, at the cost of higher evaluation expense [S1, unofficial].
*   **Steps:**
    1.  Calculate the default path and velocity for the gas movement [S1, unofficial].
    2.  Evaluate the vector field tables (if used) and the functional flow field functions at the current point [S1, unofficial].
    3.  Combine the default path vector, the vector from the vector field table, and the vector from the flow field function to produce the new path [S1, unofficial].
*   **Parameters:**
    *   `percent_of_vector_to_use`: A value used to smoothly transition between control sources (flow field, vector field, default path) and between different flow field functions [S1, unofficial].
*   **Pitfalls:**
    *   The `percent_to_use` value decreases as the sample point moves away from the center of control for a given flow field function [S1, unofficial].

### Vector Field Tables
A type of 3D table that stores direction vectors, density scaling factors, and other information at each lattice point, suited for visualizing CFD simulations [S1, unofficial]. Vector field tables are cheaper to evaluate than functional flow field tables but provide fixed resolution detail, whereas functional tables provide infinite detail [S1, unofficial].

## Operating characteristics
*   **Evaluation Cost:** Functional flow field functions are much more expensive to evaluate than simply interpolating values from a vector field table [S1, unofficial].
*   **Strengths:** Allows for complex, choreographed gas animation with infinite detail via functional flow fields [S1, unofficial].
*   **Limitations:** Functional flow field functions are computationally expensive to evaluate compared to vector field tables [S1, unofficial].
*   **When to use:** When precise, artistic control over gas movement is needed, and computational budget allows for function evaluation per sample [S1, unofficial].
*   **When not to use:** When performance is critical and simple, fixed-resolution control (vector field tables) is sufficient [S1, unofficial].
*   **Time and Memory Complexity:** Not given in the sources.

## Background
3D Table Animation was introduced by David Ebert in the context of procedural modeling, animation, and rendering of gases, fluids, and textures [S1, unofficial]. It is an extension of previous work using three-dimensional tables for volume shadowing effects [S1, unofficial]. The method is described in detail in Ebert's work on procedural gas rendering [S1, unofficial].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
