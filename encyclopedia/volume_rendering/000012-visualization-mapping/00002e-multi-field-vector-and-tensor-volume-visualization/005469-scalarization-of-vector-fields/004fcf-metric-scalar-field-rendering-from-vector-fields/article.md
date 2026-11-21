# Metric Scalar Field Rendering from Vector Fields

Also called: Vector Field Metric Extraction

## Overview
Metric Scalar Field Rendering from Vector Fields solves the problem of visualizing vector field data (such as velocity or magnetic fields) using standard scalar rendering techniques like volume rendering or isosurface extraction, which inherently require single-value inputs [S1, unofficial]. It differs from direct vector visualization methods (e.g., arrow glyphs or streamlines) by projecting the multi-component vector data into a scalar metric (e.g., magnitude, divergence, or curl) to leverage the perceptual depth and occlusion handling of volume rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A vector field $\mathbf{F}(x,y,z)$ defined on a 3D grid, potentially time-dependent.
*   **Output:** A scalar field $S(x,y,z)$ derived from $\mathbf{F}$, used as input for a scalar renderer (volume raymarching or isosurface extraction).

**Data Structures:**
*   **Vector Field Grid:** A 3D array storing components $(F_1, F_2, F_3)$ at each voxel $(i,j,k)$.
*   **Scalar Field Grid:** A 3D array storing the computed metric value $S$ at each voxel.

**Steps:**
1.  **Metric Calculation:** For each voxel $(i,j,k)$, compute the desired scalar metric from the vector components. Common metrics include:
    *   **Velocity Magnitude:** $|\mathbf{F}| = \sqrt{F_1^2 + F_2^2 + F_3^2}$.
    *   **Divergence:** $\nabla \cdot \mathbf{F} = \frac{\partial F_1}{\partial x} + \frac{\partial F_2}{\partial y} + \frac{\partial F_3}{\partial z}$.
    *   **Curl Magnitude:** $|\text{curl } \mathbf{F}|$.
2.  **Temporal Interpolation (if time-dependent):** If the data consists of discrete time slices, interpolate the vector values at the current time step $t$ between the surrounding time slices $t_1$ and $t_2$ to support smooth animation [S1, unofficial].
3.  **Rendering:** Pass the resulting scalar field to a standard scalar volume renderer or isosurface extractor. The renderer uses a transfer function to map scalar values to color and opacity [S1, unofficial].

**Formulas:**
*   **Curl Vector:**
    $$ \text{curl } \mathbf{F}(x,y,z) = \begin{pmatrix} \frac{\partial F_3}{\partial y} - \frac{\partial F_2}{\partial z} \\ \frac{\partial F_1}{\partial z} - \frac{\partial F_3}{\partial x} \\ \frac{\partial F_2}{\partial x} - \frac{\partial F_1}{\partial y} \end{pmatrix} $$
    [S2]
*   **Curl Magnitude:**
    $$ |\text{curl } \mathbf{F}(i,j,k)| = \sqrt{ \left(\frac{\partial F_3}{\partial y} - \frac{\partial F_2}{\partial z}\right)^2 + \left(\frac{\partial F_1}{\partial z} - \frac{\partial F_3}{\partial x}\right)^2 + \left(\frac{\partial F_2}{\partial x} - \frac{\partial F_1}{\partial y}\right)^2 } $$
    [S2]

**Parameters:**
*   **Metric Type:** Selection of magnitude, divergence, or curl.
*   **Interpolation Method:** Linear or higher-order interpolation for time steps [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Derivatives required for curl and divergence are sensitive to grid boundaries. Not given in the sources how boundaries are handled.
*   **Loss of Directional Information:** Scalarization discards the vector direction, which may be critical for interpreting flow topology.

## Subtopics

### Curl Magnitude Calculation
A specific metric extraction method that computes the magnitude of the curl of the vector field to highlight swirling features and vorticity (turbulent behavior) [S2]. It is chosen when the visual goal is to identify regions of rotation or turbulence within the flow, distinguishing it from magnitude-based methods which show speed [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, for the metric calculation. Rendering complexity depends on the underlying scalar renderer.
*   **Memory Complexity:** $O(N)$ to store the derived scalar field.
*   **Strengths:** Allows the use of mature scalar volume rendering techniques (e.g., raymarching) for vector data; provides immersive 3D views of internal structures [S1, unofficial].
*   **Limitations:** Discards directional information; requires careful selection of the metric to avoid misleading representations; derivatives (curl/divergence) are noisy and sensitive to discretization.

## Background
The method originates from interactive visual data analysis systems that needed to bridge the gap between vector field simulations and standard scalar rendering pipelines [S1, unofficial]. It was developed to allow simultaneous visualization of scalar structures (like density) and vector-derived features (like vorticity) [S1, unofficial]. The curl magnitude metric is formally defined in the context of multi-dimensional transfer functions for 3D flow visualization [S2].

## Sources
[S1] Interactive Visual data analysis | Daniel Suttor, https://dasutt.github.io/stories/interactive-visual-data-analysis/, fetched 2026-10-28 (unofficial)
[S2] Multi-dimensional Transfer Functions for Interactive 3D Flow Visualization, https://web.cs.ucdavis.edu/~hamann/ParkBudgeLinsenHamannJoy2004.pdf, fetched 2026-10-27
