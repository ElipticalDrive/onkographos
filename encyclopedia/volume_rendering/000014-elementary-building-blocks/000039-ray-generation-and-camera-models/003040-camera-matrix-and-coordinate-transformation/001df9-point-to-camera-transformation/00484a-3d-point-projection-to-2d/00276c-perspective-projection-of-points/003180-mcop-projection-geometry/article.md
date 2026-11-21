# MCOP Projection Geometry

Also called: Light Field Display Projection Geometry, Light field geometry, MCOP perspective projection, Projector-based multi-view projection, Two-viewpoint projection

## Overview
MCOP Projection Geometry is a projection method for light field displays that uses two distinct viewpoints to map 3D world points to projector screen coordinates: an emitter $E$ for horizontal coordinates and a virtual viewer $V$ for vertical coordinates [S1][S2]. It solves the problem of perspective-correct rendering on holographic screens that are sharply transmissive horizontally but scatter widely vertically, ensuring view separation is maintained horizontally while allowing vertical viewing flexibility [S1][S2]. It differs from standard perspective projection by decoupling the horizontal and vertical projection centers, and from stereo projection by supporting a continuous light field rather than just two views [S1][S2].

## How it works
**Inputs:**
- World space point $P = (P_x, P_y, P_z)$ [S1][S2].
- Emitter origin $E = (E_x, E_y, E_z)$ [S1].
- Virtual viewer position $V = (V_x, V_y, V_z)$, typically defined by height $V_y$ and distance $V_z$ from the screen plane [S1].
- Screen plane defined at $z = 0$ [S1][S2].
- Image rectangle $R^- = (R_x^-, R_y^-)$ and $R^+ = (R_x^+, R_y^+)$ defining the projected image geometry orthogonal to the Z axis [S1].

**Outputs:**
- Screen projected position $S = (S_x, S_y)$ [S1][S2].
- Normalized projected coordinates $H = (H_x, H_y, H_z)$ for rasterization [S1].

**Data Structures:**
- Points are represented as 3D vectors $(x, y, z)$ [S1].
- Rectangle parameters are stored as four scalar values defining the bounds in the image plane [S1].

**Steps:**
1. **Compute Horizontal Screen Coordinate ($S_x$):** Intersect the ray originating from emitter $E$ passing through point $P$ with the screen plane $z=0$ [S1][S2].
2. **Compute Vertical Screen Coordinate ($S_y$):** Intersect the ray originating from point $P$ passing through the virtual viewer $V$ with the screen plane $z=0$ [S1][S2].
3. **Remap to Normalized Coordinates:** Transform $S$ into the image rectangle defined by $R^-$ and $R^+$ to obtain $H_x$ and $H_y$ [S1].

**Formulas:**
The screen projected position $S$ is computed as [S1][S2]:

$$
S_x = E_x - E_z \cdot \frac{E_x - P_x}{E_z - P_z}
$$

$$
S_y = V_y - V_z \cdot \frac{V_y - P_y}{V_z - P_z}
$$

The normalized projected coordinates $H$ are computed as [S1]:

$$
H_x = \frac{(S_x - E_x) \cdot 2}{R_x^+ - R_x^-} - \frac{R_x^+ + R_x^-}{R_x^+ - R_x^-}
$$

$$
H_y = \frac{(S_y - E_y) \cdot 2}{R_y^+ - R_y^-} - \frac{R_y^+ + R_y^-}{R_y^+ - R_y^-}
$$

$$
H_z = -P_z / V_z
$$

**Parameters:**
- $V_y$: Virtual viewer height from screen [S1].
- $V_z$: Virtual viewer distance from screen [S1].
- $E = (E_x, E_y, E_z)$: Projector emitter origin [S1].
- $P = (P_x, P_y, P_z)$: World space point [S1].
- $R^+, R^-$: 2D rectangle defining projected image geometry orthogonal to Z axis [S1].
- Screen plane $z$-coordinate: $z = 0$ [S2].

**Edge Cases and Pitfalls:**
- Division by zero occurs if $P_z = E_z$ or $P_z = V_z$, requiring $P_z \neq E_z$ and $P_z \neq V_z$ for valid projection [S1].
- The model assumes a pinhole emitter model [S1].
- The solution is exact for all viewers at the same distance from the screen and height as the virtual viewer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcop_projection(P, E, V, R_minus, R_plus):
    """
    Projects a 3D point P to normalized coordinates H using MCOP geometry.
    
    Args:
        P: tuple (Px, Py, Pz)
        E: tuple (Ex, Ey, Ez)
        V: tuple (Vx, Vy, Vz)
        R_minus: tuple (Rx_min, Ry_min)
        R_plus: tuple (Rx_max, Ry_max)
        
    Returns:
        H: tuple (Hx, Hy, Hz)
    """
    Px, Py, Pz = P
    Ex, Ey, Ez = E
    Vx, Vy, Vz = V
    Rx_min, Ry_min = R_minus
    Rx_max, Ry_max = R_plus

    # Compute screen projected position S
    # Sx: Intersection of ray E->P with z=0 plane
    # Formula: Sx = Ex - Ez * (Ex - Px) / (Ez - Pz)
    if Ez == Pz:
        raise ValueError("Emitter and point are coplanar in Z")
    Sx = Ex - Ez * (Ex - Px) / (Ez - Pz)

    # Sy: Intersection of ray P->V with z=0 plane
    # Formula: Sy = Vy - Vz * (Vy - Py) / (Vz - Pz)
    if Vz == Pz:
        raise ValueError("Virtual viewer and point are coplanar in Z")
    Sy = Vy - Vz * (Vy - Py) / (Vz - Pz)

    # Remap to normalized projected coordinates H
    # Hx: Linear transform to image rectangle
    Rx_range = Rx_max - Rx_min
    if Rx_range == 0:
        raise ValueError("Invalid image rectangle width")
    Hx = ((Sx - Ex) * 2 / Rx_range) - ((Rx_max + Rx_min) / Rx_range)

    # Hy: Linear transform to image rectangle
    Ry_range = Ry_max - Ry_min
    if Ry_range == 0:
        raise ValueError("Invalid image rectangle height")
    Hy = ((Sy - Ey) * 2 / Ry_range) - ((Ry_max + Ry_min) / Ry_range)

    # Hz: Depth based on distance to screen relative to viewer
    Hz = -Pz / Vz

    return (Hx, Hy, Hz)
```

## Subtopics

### [Stereo Projection](5789-stereo-projection.md)
Projects points to two distinct views for stereoscopic 3D. Choose when only two viewpoints are needed. MCOP supports a continuous light field.

### [Perspective Projection of Points](4037-perspective-projection-of-points.md)
Standard single-center projection. MCOP builds on this but uses two centers for light field displays.

## Operating characteristics
- **Time Complexity:** Constant time $O(1)$ per point, involving basic arithmetic operations [S1][S2].
- **Memory Complexity:** Minimal, storing only point coordinates and projection parameters [S1].
- **Measured Performance:** The hardware implementation using 96 fast 320x240 LCD displays fed by FPGA input processing units is capable of visualizing 7.4M beams/frame [S2]. On-screen 2D pixel size is $s_0 = 1.25$mm, with angular accuracy of $0.8^\circ$ [S2].
- **Strengths:** Provides perspective-correct images for a specific virtual viewer while maintaining horizontal view separation on holographic screens [S1][S2]. Efficient for rasterization applications [S1][S2].
- **Limitations:** Perspective is exact only for viewers at the same height and distance as the virtual viewer [S1]. Vertical viewing angle is fixed by the virtual viewer assumption, though the screen scatters widely vertically [S1][S2].
- **When to use:** For rendering on light field displays with holographic screens that have anisotropic scattering properties (horizontal selectivity, vertical scattering) [S1][S2].
- **When not to use:** For standard displays or when a wide range of vertical viewing angles are required without a fixed virtual viewer model [S1].

## Background
The method is described in the context of interactive light field displays [S1]. It addresses the specific optical properties of holographic screens used in light field displays, which maintain sub-degree view separation horizontally but scatter widely vertically [S1][S2]. The approach was developed to allow 3D graphics applications to project models correctly onto such displays by accounting for the screen's transformation properties [S1][S2]. The hardware implementation described involves an array of LCD displays and FPGA processing units [S2].

## Sources
[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
[S2] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
