# Stream-Balls


## Overview
Stream-Balls is a technique for visualizing steady-state vector fields that constructs a scalar potential field from advected particles or streamlines using blobby functions, then extracts iso-contours to mimic stream surfaces [S1, unofficial]. It sets itself apart from its parent [Stream Surfaces](7696-stream-surfaces.md) by replacing explicit geometric stitching with an implicit scalar field, which allows the resulting surface to gracefully split apart in areas of high divergence and break apart in areas of high velocity to indicate magnitude [S1, unofficial].

## How it works
**Inputs:** A set of seed points and a steady-state vector field to advect particles [S1, unofficial].
**Outputs:** Iso-contour surfaces that mimic stream surfaces [S1, unofficial].

**Data Structures:**
- **Grid:** A regular 3D grid for sampling the analytical function [S1, unofficial].
- **Particle State:** A list of positions $\mathbf{p}_{i,t}$ for each particle $i$ at time step $t$ [S1, unofficial].

**Steps:**
1. **Advect Particles:** Generate trajectories from the seed points through the vector field to obtain a sequence of positions for each particle over time [S1, unofficial].
2. **Define Kernel:** Place a Gaussian blobby function at each advected particle position [S1, unofficial].
3. **Sum Contributions:** Compute the total potential field by summing the Gaussian functions over all time steps for each particle, and then summing over all particles, yielding an analytical scalar function [S1, unofficial].
4. **Sample Grid:** Evaluate the analytical function at the centers of a regular 3D grid covering the domain of interest [S1, unofficial].
5. **Extract Surface:** Compute an iso-contour of the sampled scalar grid to produce the final visualization surface [S1, unofficial].

**Formulas:** The potential field is constructed as $V(\mathbf{x}) = \sum_{i} \sum_{t} G(\mathbf{x}; \mathbf{p}_{i,t})$, where $G$ is a Gaussian function and $\mathbf{p}_{i,t}$ is the position of particle $i$ at time $t$ [G].

**Parameters:** The exact mathematical form of the Gaussian kernel and the iso-contour threshold value are Not given in the sources.

**Edge Cases and Pitfalls:** In areas of high divergence, the surface gracefully splits apart, losing the strict separation property inherent in explicit stream surfaces [S1, unofficial]. In areas of high velocity, the surface breaks apart, which helps indicate velocity magnitude but fragments the visualization [S1, unofficial].

## Subtopics

### [Streamtube Rendering](7723-streamtube-rendering.md)
Visualizes 3D vector fields by integrating trajectories from seed points and generating explicit tube geometry around the path, colored by vector magnitude. It sets itself apart from Stream-Balls by replacing the scalar potential field extraction with direct geometric tube primitives, offering explicit control over tube radius and cross-sectional shape. Choose it when analyzing specific trajectories without occlusion from neighboring flows is required, rather than when visualizing divergence and velocity magnitude through implicit surfaces.

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Scaling:** Not given in the sources.
**Strengths:** Indicates velocity magnitude through surface fragmentation; handles divergence gracefully [S1, unofficial].
**Limitations:** Loses the nice property of separation inherent in stream surfaces; surface breaks apart in areas of high velocity [S1, unofficial].
**When to use:** Not given in the sources.
**When not to use:** Not given in the sources.

## Background
Stream-Balls were proposed by Brill [S1, unofficial]. The concept is similar to van Wijk’s implicit stream surfaces but uses blobby functions (meta-balls) to build the scalar potential field [S1, unofficial]. It is an alternative to stream surfaces and flow volumes for visualizing steady-state flows [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
