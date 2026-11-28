# Sub-interval Radiative Transfer Equation (RTE) Combination

Also called: Interval splitting for RTE, Optical depth splitting, Radiance combination, Slice Rendering

## Overview
Sub-interval RTE Combination is a method for computing full-interval radiance and optical depth by partitioning the integration path of the Radiative Transfer Equation (RTE) at an intermediate point, allowing solutions from multiple slice planes to be combined [S1, unofficial]. It solves the problem of visualizing participating media (such as smoke and fire) by rendering one partially transparent slice at a time and merging them using video hardware, rather than integrating over the entire volume in a single pass [S1, unofficial]. It differs from [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) by processing data in discrete 2D slices rather than a continuous ray integral, and from [Discrete Volume Transmittance Integration](3905-discrete-volume-transmittance-integration.md) by relying on hardware compositing of pre-solved slice radiance rather than per-sample accumulation [S1, unofficial].

## How it works
**Inputs:**
*   3D computational domain data (e.g., soot density, temperature) defined on a grid.
*   Viewer position and line of sight.
*   Grid spacing parameters (e.g., $\Delta x$ between adjacent YZ planes).

**Outputs:**
*   Final composite image (radiance $C$) for the current frame.

**Data Structures:**
*   **Slice Planes:** A series of 2D planes partitioning the 3D domain. The orientation is chosen to be most perpendicular to the viewer's line of sight (parallel to XY, XZ, YZ cartesian planes, or diagonal) [S1, unofficial].
*   **Opacity Data:** Computed per grid node on each slice. Preprocessed using run-length encoding (RLE) for compression [S1, unofficial].
*   **RTE Solution State:** For each slice, the local radiance contribution ($C_i$) and optical depth ($\tau_i$) are computed independently [S1, unofficial].

**Steps:**

1.  **Partition Domain:** Divide the 3D computational domain into a series of 2D slices [S1, unofficial].
2.  **Select Orientation:** Choose the slice orientation (plane) most perpendicular to the viewer's line of sight [S1, unofficial].
3.  **Compute Opacity:** Calculate opacity at each grid node for the selected slice.
    *   For YZ planes, use the distance $\Delta x$ between adjacent planes and soot density data [S1, unofficial].
    *   If the slice orientation is not YZ, adjust the computed opacities if the distance between planes differs from $\Delta x$ [S1, unofficial].
4.  **Compress Data:** Compress the opacity data using run-length encoding as a preprocessing step [S1, unofficial].
5.  **Solve RTE per Slice:** For each slice plane, solve the RTE accounting only for conditions between adjacent slices. This yields a partial radiance $C_i$ and optical depth $\tau_i$ for that specific sub-interval [S1, unofficial].
6.  **Decompress and Render:** Decompress opacity data one frame at a time as data is displayed [S1, unofficial]. Draw the resulting partially transparent slice planes individually [S1, unofficial].
7.  **Combine Slices:** Combine the individual slice solutions using video hardware (alpha blending) to form the final image. This combination relies on the sub-interval composition formulas below [S1, unofficial].

**Formulas:**
The method relies on the multiplicative property of optical depth and the linear combination of radiance. Let the full interval be $[x_0, x_N]$ and the splitting point be $D$.

1.  **Optical Depth Splitting:**
    $$ \tau = \tau(x_0, x_N) = \tau(x_0, D)\tau(D, x_N) = \tau_1 \tau_2 $$
    Where $\tau_1$ is the optical depth from the start to the split point, and $\tau_2$ is from the split point to the end [S1, unofficial].

2.  **Radiance Combination:**
    $$ C = \tau_2 C_1 + C_2 $$
    Where $C_1$ is the radiance contribution from the first sub-interval (closer to the viewer or previous slice) and $C_2$ is the radiance contribution from the second sub-interval (further from the viewer or current slice) [S1, unofficial].

    *Note: In the context of hardware compositing, $C_1$ typically corresponds to the accumulated color from previous slices (already composited), and $C_2$ is the color of the current slice being drawn, with $\tau_2$ acting as the transmittance of the current slice.*

**Parameters:**
*   **Slice Density:** Determined by the simulation grid. Higher density (smaller separation distance) increases accuracy but risks numerical errors [S1, unofficial].
*   **Compression:** Run-length encoding is used for opacity data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Round-off:** If two slices are too close together, computed opacity values are subject to increased round-off error due to finite precision arithmetic [S1, unofficial].
*   **Truncation to Zero:** If planes are sufficiently close, computed opacities may truncate to zero, causing the slice to become invisible. In this situation, volume rendering methods (integrating over the entire mesh) are required instead of slice rendering [S1, unofficial].
*   **Orientation Adjustment:** Opacity must be adjusted if the slice orientation is not the primary YZ plane and the inter-plane distance differs from the base grid spacing $\Delta x$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of slices and the cost of solving the RTE per slice. Preprocessing (RLE compression) adds overhead but reduces per-frame data transfer [S1, unofficial].
*   **Memory Complexity:** Requires storage for the 3D domain data and compressed opacity maps for the current frame [S1, unofficial].
*   **Performance:** Relies on video hardware for combining slices, which can be efficient for real-time visualization if the number of slices is manageable [S1, unofficial].
*   **Strengths:** Allows rendering of complex 3D data by breaking it into manageable 2D problems; leverages standard graphics hardware for compositing [S1, unofficial].
*   **Limitations:** Susceptible to numerical round-off errors and opacity truncation when slice spacing is very small; requires careful selection of slice orientation relative to the viewer [S1, unofficial].
*   **When to use:** Suitable for visualizing smoke and fire data where the grid spacing is not excessively fine, and real-time or near-real-time feedback is desired [S1, unofficial].
*   **When not to use:** When slice planes are so close that opacities truncate to zero, or when high precision is required without the risk of finite precision errors associated with slice splitting [S1, unofficial].

## Background
The method originates from techniques for visualizing fire dynamics simulation data, specifically smoke and fire [S1, unofficial]. It is described in the technical reference guide for Smokeview (Version 6), a tool for visualizing Fire Dynamics Simulation (FDS) data [S1, unofficial]. The approach distinguishes itself from volume rendering by partitioning the integration path of the RTE across individual slice planes within a single mesh, rather than integrating over the entire data mesh from front to back [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
