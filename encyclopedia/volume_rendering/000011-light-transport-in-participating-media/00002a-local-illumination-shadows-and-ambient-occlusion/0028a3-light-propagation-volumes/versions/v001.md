# Light Propagation Volumes


## Overview
Light Propagation Volumes (LPV) simulates first-bounce indirect illumination by discretizing the scene into a 3D grid of cells, where each cell stores a directional distribution of radiance [S1, unofficial]. It differs from voxel cone tracing by using a continuous uniform grid with spherical harmonic encoding rather than a sparse octree, and from cascade variants by relying on a single global grid structure [S1, unofficial].

## How it works
**Inputs:** Scene geometry (for surface detection), light sources, camera position.
**Outputs:** Radiance field stored in the LPV grid.

**Data Structures:**
*   **LPV Grid:** A 3D uniform grid of cells.
*   **Cell Data:** Each cell contains a directional radiance distribution encoded as 2nd-order spherical harmonics (SH). 2nd-order SH requires 9 coefficients per color channel (RGB), totaling 27 floats per cell [S1, unofficial].

**Steps:**
1.  **Grid Initialization:** Initialize all cells to zero radiance.
2.  **Direct Lighting Injection:**
    *   Identify cells containing directly lit surfaces. This is typically done using Reflective Shadow Maps (RSM) or similar techniques to map surface points to grid cells [S1, unofficial].
    *   For each lit surface point, determine the incident light color and surface normal.
    *   Construct a radiance distribution in the corresponding cell. This distribution is centered around the surface normal, facing away from the surface, with color based on the surface material's reflection properties [S1, unofficial].
    *   Inject this distribution into the cell's SH coefficients.
3.  **Light Propagation:**
    *   Iterate for a fixed number of propagation steps $N$.
    *   In each step, every cell analyzes the radiance fields of its immediate neighbors.
    *   Each cell modifies its own radiance based on the incoming radiance from all directions from its neighbors. This simulates light bouncing between cells [S1, unofficial].
    *   The number of steps $N$ controls the extent of light propagation [S1, unofficial].

**Parameters:**
*   **Grid Resolution:** Not given in the sources.
*   **Propagation Steps ($N$):** Controls the reach of indirect light. Higher values increase accuracy but cost more. Not given in the sources.
*   **SH Order:** 2nd order is specified [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Temporal Stability:** If the number of lights or cells is insufficient, moving lights can cause flickering due to heuristic selection of grid regions [S1, unofficial].
*   **Boundary Conditions:** Not given in the sources.

## Subtopics

### [Octree Light Propagation Volumes](12437-octree-light-propagation-volumes.md)
Uses a full octree to allow variable resolution, prioritizing proximity to geometry and reflected light over viewer proximity, solving slow propagation in uniform grids.

## Operating characteristics
*   **Time Complexity:** Depends on grid size and propagation steps. Each propagation step requires neighbor communication for all cells.
*   **Memory Complexity:** Proportional to grid volume $\times$ SH coefficients (27 floats/cell).
*   **Strengths:** Real-time capable, simulates first-bounce indirect illumination.
*   **Limitations:** Flickering with moving lights if grid/light count is low; uniform grid may waste memory in empty areas compared to sparse methods [S1, unofficial].
*   **When to use:** Real-time global illumination where first-bounce indirect light is desired and uniform grid overhead is acceptable.

## Background
Introduced by Kaplanyan [S1, unofficial]. Kaplanyan et al. later developed a cascade variant (CLPV) [S1, unofficial]. It is a precursor to or alternative to voxel cone tracing (VXGI) by Crassin, which uses sparse octrees [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
