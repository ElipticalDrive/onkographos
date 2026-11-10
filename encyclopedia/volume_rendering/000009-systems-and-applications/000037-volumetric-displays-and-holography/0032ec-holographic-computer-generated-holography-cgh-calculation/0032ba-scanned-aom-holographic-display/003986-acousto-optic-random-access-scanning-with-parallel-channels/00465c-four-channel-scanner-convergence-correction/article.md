# Four-Channel Scanner Convergence Correction

Also called: Acousto-Optic Scanner Convergence Correction

## Overview
Four-Channel Scanner Convergence Correction is a geometric alignment method for four-channel acousto-optic (AO) holographic displays, ensuring that four scanned laser beams (organized as two pyramidal sources) intersect uniformly throughout the volumetric display space [S1, unofficial]. It solves the problem of beam separation caused by the physical offset of output lenses, which would otherwise cause the beams to intersect at only a single point rather than throughout the entire volume [S1, unofficial]. Unlike its sibling [Acousto-Optic Random Access Scanning with Parallel Channels](8508-acousto-optic-random-access-scanning-with-parallel-channels.md), which defines the parallel channel architecture, this method specifically addresses the spatial registration errors inherent in that architecture. It differs from [Acousto-Optic Modulator R-Factor Analysis](11746-acousto-optic-modulator-r-factor-analysis.md) by focusing on macroscopic geometric alignment rather than device physics limits.

## How it works
**Inputs and Outputs**
*   **Inputs:** Scan coordinates for four channels (A, B, C, D); helix volume geometry (height, width); target intersection plane (top or bottom of helix).
*   **Outputs:** Corrected scan deflection angles or pixel addresses for each channel to ensure uniform beam overlap.

**Data Structures**
The method operates on scan position variables. Let $x$ be the horizontal scan coordinate (0 to 1, normalized) and $y$ be the vertical helix axis coordinate (0 to 1, normalized from bottom to top).
*   Channel Group 1: {A, B}
*   Channel Group 2: {C, D}

**Steps**
1.  **X-Axis Correction (Primary):** Apply a linear correction across the scan direction to counteract the ~10 mm separation of the pyramidal sources at the helix base [S1, unofficial].
    *   For channels C and D: The correction offset $\Delta x$ is 0 mm at the scan start ($x=0$) and increases linearly to 10 mm at the scan end ($x=1$).
    *   For channels A and B: The correction offset $\Delta x$ is 10 mm at the scan start ($x=0$) and decreases linearly to 0 mm at the scan end ($x=1$).
    *   Formula for C/D: $\Delta x_{C/D}(x) = 10 \cdot x$ mm
    *   Formula for A/B: $\Delta x_{A/B}(x) = 10 \cdot (1 - x)$ mm
    *   [G] The correction is applied as an additive offset to the deflection angle or pixel coordinate before driving the AO modulator.

2.  **Y-Axis Correction (Secondary):** Apply a linear correction along the helix axis ($y$) to match the intersection point of the pyramidal sources [S1, unofficial].
    *   If the intersection is targeted at the top of the helix, the correction varies linearly from the bottom ($y=0$) to the top ($y=1$).
    *   Formula: $\Delta y(y) = k \cdot y$, where $k$ is the required vertical shift to align the pyramid apexes.
    *   [G] The sign of the correction depends on whether the pyramids converge upward or downward.

3.  **Vertical Line Constraint (Optional):** If perfectly vertical display lines are required, a slight non-zero correction must be applied even at the beginning of each scan, and both A/B and C/D channels must be corrected in the y-direction [S1, unofficial]. This distinguishes "convergence" (beam overlap) from "linearity" (straight scan lines).

**Parameters**
*   **Beam Separation:** ~10 mm at the bottom of the helix [S1, unofficial].
*   **Helix Height:** 36 inches (approx. 914 mm) [S1, unofficial].
*   **Correction Range:** 0 to 10 mm for X-axis [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Y-Axis Ignorance:** The Y-axis convergence error is barely detectable and can be ignored for general use, simplifying implementation to X-axis correction only [S1, unofficial].
*   **Vertical Lines:** If vertical lines are mandatory, ignoring Y-axis correction results in slanted lines. Both channel groups must be corrected in Y [S1, unofficial].
*   **Intersection Point:** The correction profile depends on whether the pyramids are designed to intersect at the top or bottom of the helix. The linear slope reverses based on this choice [S1, unofficial].

## Subtopics

### Color Convergence Correction
Optical and software correction technique to align red, green, and blue scan beams that diverge due to wavelength-dependent acousto-optic deflection [S1, unofficial]. AO deflection is proportional to wavelength, causing the red scanner to have a wider scan angle than green, which is wider than blue [S1, unofficial]. To correct this, telephoto projection lenses are used to continuously adjust angular magnification, achieving rough alignment of the red, green, and blue scan angles [S1, unofficial]. Software-based minor corrections are then applied to fine-tune color registration [S1, unofficial]. A key pitfall is relying solely on software correction, which causes the red channel to have lower resolution than the blue channel due to the wider red scan angle mapping to the same pixel grid [S1, unofficial].

## Operating characteristics
*   **Complexity:** Low. The correction is linear and can be implemented with simple arithmetic offsets in the scan controller [S1, unofficial].
*   **Performance:** Corrects the primary geometric distortion (X-axis) effectively. Y-axis errors are negligible for general viewing [S1, unofficial].
*   **Limitations:** Does not address chromatic aberration (color convergence), which requires additional optical hardware (telephoto lenses) [S1, unofficial].
*   **When to use:** When operating a four-channel AO holographic display in a helical volume configuration.
*   **When not to use:** When single-channel or two-channel operation is used, or when color registration is the primary concern without optical correction hardware.

## Background
This method originates from the development of the Second-Generation 3-D Volumetric Display System, specifically the "big helix" optical system implemented around 1997 [S1, unofficial]. The system distributes four channels (two red, two green) into the four quadrants of the helix volume to increase voxel count (up to 10,000 green and 10,000 red voxels per quadrant) [S1, unofficial]. The convergence problem was identified as one of the most challenging aspects of the immediate future for this design due to the physical separation of output lenses creating non-intersecting pyramidal sources [S1, unofficial].

## Sources
[S1] DTIC ADA358303: Improved Second-Generation 3-D Volumetric Display System. Revision 2, https://archive.org/details/DTIC_ADA358303, fetched 2026-10-26 (unofficial)
