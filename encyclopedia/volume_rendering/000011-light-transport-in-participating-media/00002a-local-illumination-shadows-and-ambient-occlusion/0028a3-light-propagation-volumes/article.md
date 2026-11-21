# Light Propagation Volumes
Also called: Real-time Volumetric Radiance Propagation, Voxel-based Global Illumination
## Overview
Light Propagation Volumes (LPV) simulates first-bounce indirect illumination by discretizing the scene into a 3D grid of cells, where each cell stores a directional distribution of radiance [S1, unofficial]. It differs from voxel cone tracing by using a continuous uniform grid with spherical harmonic encoding rather than a sparse octree, and from cascade variants by relying on a single global grid structure [S1, unofficial].
## How it works
**Inputs:** Scene geometry and light sources [S2].
**Outputs:** A radiance field stored in the LPV grid, sampled during deferred shading [S2].
**Data Structures:** The LPV grid is a 3D uniform grid of cells [S1, unofficial]. Each cell stores a directional radiance distribution encoded as 2nd-order spherical harmonics (SH), requiring 9 coefficients per color channel (RGB) for a total of 27 floats per cell [S1, unofficial].
**Steps:**
1. **Grid Initialization:** Initialize all cells to zero radiance [G].
2. **Direct Lighting Injection:** Identify cells containing directly lit surfaces using reflective shadow maps or similar techniques [S1, unofficial]. For each lit surface point, determine the incident light color and surface normal [G]. Construct a radiance distribution centered around the surface normal, facing away from the surface, with color based on the surface material [S1, unofficial]. Inject this distribution into the cell's SH coefficients [S2].
3. **Light Propagation:** Iterate for a fixed number of propagation steps $N$ [S1, unofficial]. In each step, every cell analyzes the radiance fields of its immediate neighbors [S1, unofficial]. Each cell modifies its own radiance based on the incoming radiance from all directions from its neighbors [S1, unofficial]. The number of steps $N$ controls the extent of light propagation [S1, unofficial].
4. **Final Gathering:** Sample the grid during the deferred shading pass to apply indirect diffuse lighting to dynamic objects [S2].
**Formulas:** Not given in the sources.
**Parameters:** Grid resolution is not given in the sources. Propagation steps $N$ control the reach of indirect light; higher values increase accuracy but cost more [S1, unofficial]. SH order is fixed at 2nd order [S1, unofficial].
**Edge Cases and Pitfalls:** Temporal stability issues and flickering occur if the number of lights or cells is insufficient for moving lights [S1, unofficial]. Light leaking occurs when indirect light incorrectly bleeds through thin geometry due to insufficient grid resolution [S2]. Improper tuning of grid dimensions causes significant memory overhead or performance bottlenecks [S2]. Boundary conditions are not given in the sources.
## Subtopics

### [Octree Light Propagation Volumes](12437-octree-light-propagation-volumes.md)
Uses a full octree to allow variable resolution, prioritizing proximity to geometry and reflected light over viewer proximity, solving slow propagation in uniform grids.

## Operating characteristics
*   **Time Complexity:** Depends on grid size and propagation steps. Each propagation step requires neighbor communication for all cells [S2].
*   **Memory Complexity:** Proportional to grid volume $\times$ SH coefficients (27 floats/cell) [S2].
*   **Strengths:** Real-time capable, simulates first-bounce indirect illumination, handles dynamic light sources and moving geometry without static rebaking [S2].
*   **Limitations:** Flickering with moving lights if grid/light count is low [S1, unofficial]. Light leaking through thin geometry due to insufficient resolution [S2]. Improper tuning of grid dimensions causes significant memory overhead or performance bottlenecks [S2].
*   **When to use:** Real-time global illumination where first-bounce indirect light is desired and uniform grid overhead is acceptable [S2].
## Background
Introduced by Kaplanyan [S1, unofficial]. Kaplanyan et al. later developed a cascade variant (CLPV) [S1, unofficial]. It is primarily designed for diffuse indirect illumination in real-time applications [S2].
## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Light Propagation Volumes: Technical Guide | wordAZ, https://www.wordaz.com/light-propagation-volumes, fetched 2026-10-29
