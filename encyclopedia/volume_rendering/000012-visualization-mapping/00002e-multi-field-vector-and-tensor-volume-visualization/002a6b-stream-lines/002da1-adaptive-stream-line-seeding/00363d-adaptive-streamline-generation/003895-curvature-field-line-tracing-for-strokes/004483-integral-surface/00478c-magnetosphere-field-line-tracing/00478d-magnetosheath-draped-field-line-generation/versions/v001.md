# Magnetosheath Draped Field Line Generation
Also called: draped IMF field lines

## Overview
Generates draped interplanetary magnetic field (IMF) line samples within the magnetosheath region, bounded by the bow shock and the magnetopause. Unlike the parent method which traces single lines from ionospheric footpoints, this method seeds lines randomly in the dayside magnetosheath to capture the draped topology of the solar wind magnetic field. It is distinct from general stream line tracing by its specific geometric constraints (dayside-only seeding) and reliance on external magnetosheath field models.

## How it works
**Inputs:**
- `n_lines`: Number of field lines to generate.
- `n_pts`: Maximum number of sample points per field line.
- `magnetopause_radius(theta)`: Function returning the radial distance of the magnetopause surface for a given polar angle `theta` (from the Sun-Earth line).
- `bow_shock_radius(theta)`: Function returning the radial distance of the bow shock surface for a given polar angle `theta`.
- `magnetosheath_field(x, y, z)`: Function returning the magnetic field vector $(B_x, B_y, B_z)$ at position $(x, y, z)$.

**Outputs:**
- A list of records. Each record is a dictionary with keys:
  - `x`, `y`, `z`: Position coordinates (float).
  - `Bx`, `By`, `Bz`: Magnetic field components at that position (float).
  - `intensity`: Magnitude of the magnetic field $|B|$ (float).
  - `source`: String literal `'magnetosheath'`.

**Data Structures:**
- The output is a flat list of dictionaries. No complex spatial indexing is required for generation, though downstream visualization may use these coordinates for rendering.

**Parameters:**
- `n_lines`: Typical values depend on desired density; the source implementation truncates total records to 5,000 if exceeded [S1, unofficial].
- `n_pts`: Number of Euler steps per line.
- `step_size`: Fixed step size for Euler integration, set to $0.20$ Earth radii ($R_E$) [S1, unofficial].
- `dayside_limit`: Polar angle restriction $\theta \in [0, 1.4]$ radians [S1, unofficial].

**Algorithm:**
1. Initialize an empty list `records`.
2. Loop `i` from $0$ to `n_lines` - 1:
   a. **Seed Selection**:
      - Generate $\phi \sim \text{Uniform}(0, 2\pi)$.
      - Generate $\theta \sim \text{Uniform}(0, 1.4)$.
      - Calculate $t_{\cos} = \max(\cos(\theta), -0.98)$.
      - Calculate $r_{mp} = \text{magnetopause\_radius}(\arccos(t_{\cos}))$.
      - Calculate $r_{bs} = \text{bow\_shock\_radius}(\arccos(t_{\cos}))$.
      - Generate $r_{\text{seed}} \sim \text{Uniform}(r_{mp} + 0.3, r_{bs} - 0.3)$.
      - If $r_{\text{seed}} \le r_{mp}$, skip to next iteration (continue).
   b. **Initialization**:
      - Set initial position:
        $$x = r_{\text{seed}} \sin(\theta) \cos(\phi)$$
        $$y = r_{\text{seed}} \sin(\theta) \sin(\phi)$$
        $$z = r_{\text{seed}} \cos(\theta)$$
      - Set `step` = $0.20$.
      - Set `step_count` = 0.
   c. **Tracing Loop**:
      - Loop `j` from $0$ to `n_pts` - 1:
        i. **Boundary Check**:
           - Calculate $\theta_{\text{sun}} = \operatorname{atan2}(\sqrt{y^2 + z^2}, \max(x, -40.0))$.
           - Calculate $r_{mp2} = \text{magnetopause\_radius}(\theta_{\text{sun}})$.
           - Calculate $r_{bs2} = \text{bow\_shock\_radius}(\theta_{\text{sun}})$.
           - Calculate $r = \sqrt{x^2 + y^2 + z^2}$.
           - If $r < r_{mp2}$ or $r > r_{bs2}$, break the tracing loop.
        ii. **Field Evaluation**:
           - Retrieve $\mathbf{B} = (B_x, B_y, B_z) = \text{magnetosheath\_field}(x, y, z)$.
           - Calculate magnitude $|\mathbf{B}| = \sqrt{B_x^2 + B_y^2 + B_z^2}$.
           - If $|\mathbf{B}| < 10^{-9}$, break the tracing loop.
        iii. **Record**:
           - Append record:
             ```python
             {
                 "x": x, "y": y, "z": z,
                 "Bx": Bx, "By": By, "Bz": Bz,
                 "intensity": |B|,
                 "source": "magnetosheath"
             }
             ```
        iv. **Euler Step**:
           - Update position:
             $$x \leftarrow x + \text{step} \cdot \frac{B_x}{|\mathbf{B}|}$$
             $$y \leftarrow y + \text{step} \cdot \frac{B_y}{|\mathbf{B}|}$$
             $$z \leftarrow z + \text{step} \cdot \frac{B_z}{|\mathbf{B}|}$$

**Edge Cases and Pitfalls:**
- **Dayside Restriction**: The method enforces a dayside-only generation by limiting $\theta$ to $[0, 1.4]$ radians and clamping $\cos(\theta)$ to $\ge -0.98$ [S1, unofficial]. This prevents seeding in the nightside magnetosheath.
- **Boundary Precision**: The boundary check uses the current position's projected angle $\theta_{\text{sun}}$ to query surface radii. If the field line crosses the boundary between steps, it may be recorded outside the valid region before the next check breaks the loop.
- **Zero Field**: If the field magnitude drops below $10^{-9}$, tracing stops to avoid division by zero [S1, unofficial].
- **Seeding Gap**: The seed radius is restricted to $(r_{mp} + 0.3, r_{bs} - 0.3)$. If $r_{bs} - r_{mp} \le 0.6$, no valid seeds exist for that angle. The code checks `if r_seed <= mp_r` but relies on the uniform distribution to naturally fail if the interval is invalid or empty.

## Operating characteristics
- **Time Complexity**: $O(N \cdot P)$, where $N$ is `n_lines` and $P$ is `n_pts`. Each step requires a field evaluation and boundary checks.
- **Memory Complexity**: $O(N \cdot P)$ to store the output records. The source implementation truncates the output to 5,000 records if the total count exceeds this limit [S1, unofficial].
- **Strengths**: Simple to implement; captures the general draped structure of the magnetosheath field; efficient random seeding avoids bias towards specific regions.
- **Limitations**: Does not trace lines back to their source (solar wind) or footpoints; relies on external functions for magnetopause/bow shock shapes and field values; fixed step size may lead to inaccuracies in high-gradient regions.

## Background
This method is part of a pipeline for volumetric visualization of astrophysical vector fields, specifically focusing on Earth's magnetosphere. It builds on [Magnetosphere Field Line Tracing](12012-magnetosphere-field-line-tracing.md) by providing complementary data for the magnetosheath region, which is not covered by ionospheric footpoint tracing. The implementation is found in the AstroSplat-Browser repository, which focuses on native volumetric visualization techniques [S1, unofficial].

## Sources
[S1] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
