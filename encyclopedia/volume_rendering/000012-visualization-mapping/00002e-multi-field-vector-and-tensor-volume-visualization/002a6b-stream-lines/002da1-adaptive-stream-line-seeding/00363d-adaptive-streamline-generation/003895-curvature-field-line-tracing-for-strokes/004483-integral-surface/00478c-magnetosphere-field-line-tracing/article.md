# Magnetosphere Field Line Tracing

Also called: Earth Magnetosphere Field Line Tracing, dipole field line integration

## Overview
This method traces a single magnetospheric magnetic field line starting from an ionospheric footpoint using a fixed-step Euler integration along the normalized magnetic field vector [S1, unofficial]. It solves the problem of visualizing the topology of the Earth's magnetic field within the magnetosphere by connecting ionospheric regions to the magnetopause boundary. It is distinct from general stream line tracing by its specific reliance on an analytical dipole field model and its geometric termination conditions (ionosphere and magnetopause surfaces) rather than a uniform domain [S1, unofficial]. It builds on [Integral Surface](11268-integral-surface.md) by providing the specific vector field evaluation and boundary logic for magnetospheric contexts.

## How it works
**Inputs and Outputs**
- **Inputs**:
  - `theta0`: Co-latitude of the ionospheric footpoint in radians [S1, unofficial].
  - `phi0`: Azimuth (longitude) of the footpoint in radians [S1, unofficial].
  - `n_points`: Maximum number of sample points per field line [S1, unofficial].
  - `r_start`: Initial radial distance, typically $3.0 R_E$ to avoid the inner dipole singularity [S1, unofficial].
- **Outputs**: A list of records, where each record is a dictionary containing:
  - `x`, `y`, `z`: Position coordinates (float) [S1, unofficial].
  - `Bx`, `By`, `Bz`: Magnetic field components at the position (float) [S1, unofficial].
  - `intensity`: Magnitude of the magnetic field (float) [S1, unofficial].
  - `source`: String literal `'magnetosphere'` [S1, unofficial].

**Data Structures**
The output is a sequence of dictionaries. Each dictionary maps string keys to float values (except `source`). The `gray` field is not stored in the input but is recomputed from `intensity` for visualization purposes [S1, unofficial].

**Algorithm Steps**
1. **Initialization**: Set the starting position $(x, y, z)$ based on spherical coordinates:
   $$
   x = r_{start} \sin(\theta_0) \cos(\phi_0)
   $$
   $$
   y = r_{start} \sin(\theta_0) \sin(\phi_0)
   $$
   $$
   z = r_{start} \cos(\theta_0)
   $$
   [S1, unofficial].

2. **Iteration Loop**: Repeat for `n_points` iterations:
   a. Calculate radial distance: $r = \sqrt{x^2 + y^2 + z^2}$ [S1, unofficial].
   b. **Ionosphere Check**: If $r < 1.1 R_E$, terminate the trace [S1, unofficial].
   c. Calculate the polar angle from the sunward X-axis:
      $$
      \theta_{sun} = \operatorname{atan2}\left(\sqrt{y^2 + z^2}, \max(x, -40.0)\right)
      $$
      [S1, unofficial].
   d. **Magnetopause Check**: Calculate the magnetopause radius $r_{mp} = \text{magnetopause\_radius}(\theta_{sun})$. If $r > r_{mp}$, terminate the trace [S1, unofficial].
   e. **Field Evaluation**: Evaluate the dipole magnetic field components $(B_x, B_y, B_z)$ at $(x, y, z)$ [S1, unofficial].
   f. Calculate magnitude: $mag = \sqrt{B_x^2 + B_y^2 + B_z^2}$ [S1, unofficial].
   g. **Singularity Check**: If $mag < 10^{-9}$, terminate to prevent division by zero [S1, unofficial].
   h. **Record**: Append the current state to the output list:
      ```python
      {
          'x': x, 'y': y, 'z': z,
          'Bx': Bx, 'By': By, 'Bz': Bz,
          'intensity': mag,
          'source': 'magnetosphere'
      }
      ```
      [S1, unofficial].
   i. **Euler Step**: Update position using a fixed step size $h$:
      $$
      x \leftarrow x + h \frac{B_x}{mag}
      $$
      $$
      y \leftarrow y + h \frac{B_y}{mag}
      $$
      $$
      z \leftarrow z + h \frac{B_z}{mag}
      $$
      [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trace_dipole_field_line(theta0, phi0, n_points, r_start=3.0, step=0.1):
    """
    Traces a single magnetospheric field line from an ionospheric footpoint.
    
    Args:
        theta0 (float): Co-latitude of the ionospheric footpoint [radians].
        phi0 (float): Azimuth (longitude) of the footpoint [radians].
        n_points (int): Maximum number of sample points per field line.
        r_start (float): Initial radial distance in R_E. Default 3.0.
        step (float): Fixed step size for Euler integration. Default 0.1.
        
    Returns:
        list[dict]: List of field line records.
    """
    # 1. Initialize position
    x = r_start * math.sin(theta0) * math.cos(phi0)
    y = r_start * math.sin(theta0) * math.sin(phi0)
    z = r_start * math.cos(theta0)
    
    records = []
    
    for _ in range(n_points):
        # 2a. Calculate radial distance
        r = math.sqrt(x**2 + y**2 + z**2)
        
        # 2b. Ionosphere boundary check
        if r < 1.1:
            break
            
        # 2c. Calculate polar angle from sunward X axis
        # Note: max(x, -40.0) prevents atan2 issues for large negative x
        theta_sun = math.atan2(math.sqrt(y**2 + z**2), max(x, -40.0))
        
        # 2d. Magnetopause boundary check
        # magnetopause_radius is an external function/model not defined in the core algorithm
        mp_r = magnetopause_radius(theta_sun)
        if r > mp_r:
            break
            
        # 2e. Evaluate dipole field
        Bx, By, Bz = dipole_field(x, y, z)
        
        # 2f. Calculate magnitude
        mag = math.sqrt(Bx**2 + By**2 + Bz**2)
        
        # 2g. Singularity/Zero field check
        if mag < 1e-9:
            break
            
        # 2h. Record current state
        records.append({
            'x': x,
            'y': y,
            'z': z,
            'Bx': Bx,
            'By': By,
            'Bz': Bz,
            'intensity': mag,
            'source': 'magnetosphere'
        })
        
        # 2i. Euler step along normalized B-field
        x += step * (Bx / mag)
        y += step * (By / mag)
        z += step * (Bz / mag)
        
    return records

def dipole_field(x, y, z):
    """
    Evaluates the magnetic field of a centered dipole.
    Assumes dipole moment aligned with Z-axis.
    B = (3(m.r)r - r^2 m) / r^5
    For m = (0,0,1):
    Bx = 3xz / r^5
    By = 3yz / r^5
    Bz = (3z^2 - r^2) / r^5
    """
    r2 = x**2 + y**2 + z**2
    r5 = r2 * r2 * math.sqrt(r2)
    
    if r5 < 1e-20:
        return (0.0, 0.0, 0.0)
        
    Bx = 3 * x * z / r5
    By = 3 * y * z / r5
    Bz = (3 * z**2 - r2) / r5
    
    return (Bx, By, Bz)

def magnetopause_radius(theta):
    """
    Placeholder for the magnetopause surface model.
    The source notes reference this function but do not provide its implementation.
    [G] A common empirical model is the Shue et al. (1998) model.
    """
    # Not given in the sources. Implementation depends on the specific magnetopause model used.
    return 10.0 
```

**Parameters**
- `theta0`: Co-latitude of the ionospheric footpoint [radians] [S1, unofficial].
- `phi0`: Azimuth (longitude) of the footpoint [radians] [S1, unofficial].
- `n_points`: Maximum number of sample points per field line [S1, unofficial].
- `r_start`: Initial radial distance, set to $3.0 R_E$ to avoid the inner dipole singularity [S1, unofficial].
- `step`: Fixed step size for Euler integration. Not explicitly given in the notes, but required for the algorithm. [G] Typical values range from 0.01 to 0.1 $R_E$ depending on desired resolution.

**Edge Cases and Pitfalls**
- **Singularity**: The dipole field has a singularity at $r=0$. This is avoided by starting the trace at $r=3.0 R_E$ [S1, unofficial].
- **Zero Field**: If the magnetic field magnitude drops below $10^{-9}$, the trace terminates to prevent division by zero during normalization [S1, unofficial].
- **Boundary Conditions**: The trace terminates if it crosses the ionosphere ($r < 1.1 R_E$) or the magnetopause surface. The magnetopause radius is a function of the polar angle from the sunward X-axis [S1, unofficial].

## Subtopics

### [Magnetosheath Draped Field Line Generation](12013-magnetosheath-draped-field-line-generation.md)
Generates draped interplanetary magnetic field (IMF) line samples within the magnetosheath region, bounded by the bow shock and the magnetopause. Unlike the parent method which traces single lines from ionospheric footpoints, this method seeds lines randomly in the dayside magnetosheath to capture the draped topology of the solar wind magnetic field. It is distinct from general stream line tracing by its specific geometric constraints (dayside-only seeding) and reliance on external magnetosheath field models.

## Operating characteristics
- **Time Complexity**: $O(N)$ per field line, where $N$ is the number of points traced before termination. Each step involves a constant number of arithmetic operations and a dipole field evaluation.
- **Memory Complexity**: $O(N)$ to store the output records for a single field line.
- **Performance**: The method is computationally lightweight as it uses an analytical dipole model rather than interpolating from a grid. Performance is limited by the number of points $N$ and the cost of the `magnetopause_radius` function if it is complex.
- **Strengths**: Simple to implement; provides physically meaningful field lines for the Earth's magnetosphere; handles boundary conditions explicitly.
- **Limitations**: Fixed-step Euler integration is less accurate than higher-order methods (e.g., Runge-Kutta); accuracy degrades with larger step sizes; relies on an external magnetopause model which may vary.
- **When to use**: When visualizing individual field lines from ionospheric footpoints in a magnetospheric context.
- **When not to use**: When high-precision trajectory integration is required (use RK4); when tracing through complex, non-dipole fields (use grid-based interpolation).

## Background
This method is derived from the `AstroSplat-Browser` project, which provides native volumetric visualization for astrophysical vector fields [S1, unofficial]. The specific implementation traces magnetospheric field lines using a dipole model, reflecting the standard approximation for Earth's magnetic field in the near-Earth environment. The approach is part of a broader family of stream line and field line tracing techniques used in space physics and computational electromagnetics.

## Sources
[S1] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
