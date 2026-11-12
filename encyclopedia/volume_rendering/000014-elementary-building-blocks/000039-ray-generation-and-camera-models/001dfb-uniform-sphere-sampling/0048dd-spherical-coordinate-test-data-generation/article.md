# Spherical Coordinate Test Data Generation
Also called: Spherical Shell Test Data, Concentric Sphere Validation Data

## Overview
A coordinate transformation method to generate concentric spherical test data for validating the sampling process in volume rendering pipelines [S1, unofficial]. It serves as a visual validation tool to ensure that the structure of input data grids is retained during the sampling process, allowing multiple spheres to be viewed simultaneously by assigning different data values to each shell [S1, unofficial]. It differs from sibling methods like [Random Hemisphere Sampling](2034-random-hemisphere-sampling.md) by providing deterministic, structured geometric primitives rather than stochastic samples, and from [Rejection Sampling for Uniform Sphere Sampling](3286-rejection-sampling-for-uniform-sphere-sampling.md) by focusing on data grid integrity verification rather than probability distribution correctness.

## How it works
**Inputs and Outputs:**
- Input: A set of discrete radius values $r$, and angular ranges for $\theta$ and $\phi$.
- Output: A 3D Cartesian grid $(x, y, z)$ representing concentric spherical shells.

**Data Structures:**
- A 3D array or grid where each cell corresponds to a Cartesian coordinate.
- The value stored at each grid point is arbitrary for validation purposes, but distinct values are assigned to different radii to distinguish shells [S1, unofficial].

**Formulas:**
The transformation from spherical coordinates $(r, \theta, \phi)$ to Cartesian coordinates $(x, y, z)$ is defined as [S1, unofficial]:

$$
x = r \cos(\theta) \sin(\phi)
$$
$$
y = r \sin(\theta) \sin(\phi)
$$
$$
z = r \cos(\phi)
$$

Where:
- $0 < \theta < 2\pi$
- $0 < \phi < \pi$
- $r$ is the radial distance.

**Parameters:**
- $r$: Starts at 1 and increases by units of 1 (e.g., $r = 1, 2, 3, \dots$) to obtain a series of concentric spheres [S1, unofficial].
- $\theta$: Azimuthal angle, range $(0, 2\pi)$ [S1, unofficial].
- $\phi$: Polar angle, range $(0, \pi)$ [S1, unofficial].

**Steps:**
1. Initialize an empty 3D grid.
2. For each integer radius $r$ starting from 1:
   a. Iterate through discrete angular samples for $\theta$ in $(0, 2\pi)$.
   b. Iterate through discrete angular samples for $\phi$ in $(0, \pi)$.
   c. Compute $x, y, z$ using the formulas above.
   d. Map $(x, y, z)$ to the nearest grid index.
   e. Assign a unique data value to this shell to allow visual distinction from other spheres [S1, unofficial].
3. Render or process the resulting grid to verify that the spherical structure is maintained during sampling [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision errors in trigonometric functions may cause gaps or overlaps in the shells if the angular resolution is too low [G].
- The boundary conditions $\theta = 0, 2\pi$ and $\phi = 0, \pi$ must be handled to avoid duplicate points at the poles or prime meridian [G].
- The value of the data items is not important for the structural validation, but distinct values are required for visual differentiation [S1, unofficial].

## Subtopics

### Cylindrical Coordinate Test Data Generation
A coordinate transformation method to generate cylindrical and truncated conic shell test data for validating the sampling process [S1, unofficial]. It uses the same validation purpose as the spherical variant but employs cylindrical coordinates. The transformation is defined as [S1, unofficial]:
$$
x = r \cos(\theta)
$$
$$
y = r \sin(\theta)
$$
$$
z = z
$$
Where $0 < \theta < 2\pi$ and $r$ increases by units of 1 starting at 1 to obtain cylindrical shells [S1, unofficial]. For truncated cones, $r$ varies linearly as a function of $z$ [S1, unofficial]. This allows validation of internal and external structure visibility in cylindrical geometries, similar to the spherical case [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of grid points generated. The generation is deterministic and fast, suitable for pre-computation.
- **Memory Complexity:** Proportional to the volume of the bounding box of the largest sphere or cylinder.
- **Strengths:** Provides clear visual confirmation that the sampling process preserves geometric structure. Allows simultaneous viewing of internal and external structures [S1, unofficial].
- **Limitations:** Only validates structural integrity, not statistical distribution or density accuracy. Does not account for actual simulation data characteristics [S1, unofficial].
- **When to use:** During the initial development and testing of volume rendering pipelines to ensure data grid retention during sampling [S1, unofficial].
- **When not to use:** For performance benchmarking or validation of statistical sampling properties.

## Background
Originated in the context of volume visualization validation, specifically in the work by Dr. Phil Webster at the Wright Research and Development Center at Wright-Patterson Air Force Base [S1, unofficial]. The method was part of a broader visual validation process that included fabricated data sets (spheres, cylinders, truncated cones) and actual simulation data from finite-element analysis [S1, unofficial]. The goal was to ensure that the structure of the input data grid was retained during the sampling process onto rectangular grids [S1, unofficial].

## Sources
[S1] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
