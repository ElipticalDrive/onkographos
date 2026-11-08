# THREAD System
Also called: Three dimensional Reconstruction And Display

## Overview
The THREAD System reconstructs 3D surfaces from a sequence of 2D cross-sectional outlines by tessellating between adjacent slice contours and applying surface-normal shading [S1, unofficial]. It operates on discrete 2D slices rather than a continuous 3D volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of 2D images or slices containing scalar data or binary masks [S1, unofficial].
*   **Output:** A tessellated polygonal mesh representing the surface, shaded with surface normals [S1, unofficial].

**Data Structures:**
*   **Slice Outlines:** 2D polygonal chains or closed loops representing the boundary of the object of interest on each slice [S1, unofficial].
*   **Connectivity Structure:** A structure linking vertices on slice $i$ to vertices on slice $i+1$ to define the surface via tessellation [S1, unofficial].

**Steps:**
1.  **Threshold Segmentation:** Apply a threshold to each 2D slice to separate the object of interest from the background, with manual correction applied as needed [S1, unofficial].
2.  **Automatic Boundary Detection:** Detect the outline of the segmented area on each slice to produce 2D contours [S1, unofficial].
3.  **Inter-slice Tessellation:** Define the 3D surface by tessellating between the outlines on consecutive slices [S1, unofficial].
4.  **Shading:** Apply simple surface-normal based shading to the generated surface polygons [S1, unofficial].
5.  **Rendering:** Display the resulting mesh at different angles, optionally with hidden-line removal [S1, unofficial].

**Parameters:**
*   **Threshold Value:** The intensity value used for segmentation. Not given in the sources.
*   **Isosurface Value:** Implicitly defined by the boundary detection step. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Topology Changes:** If the topology of the object changes between slices, the tessellation between consecutive slices may become ambiguous. The sources do not specify how THREAD handles topological changes.
*   **Alignment:** Misalignment between slices can result in distorted geometry. The sources do not specify how THREAD handles slice misalignment.

## Subtopics

### Simulation Grid Cell Skipping
An optimization technique for accelerating surface extraction in particle-based simulations by skipping over empty and fully occupied simulation grid cells, iterating only over partially filled cells [S2]. It accelerates voxelization by avoiding processing of uniform regions in the simulation grid [S2]. The mechanism iterates over packed machine words rather than single bits, using bit counting intrinsics to quickly determine the next relevant bits [S2]. It marches along the view ray corresponding to each voxel column through the simulation grid [S2]. It keeps marching through full or empty simulation grid cells until encountering a succession of partially filled cells [S2]. It interrupts ray marching to iterate over the corresponding voxels when partially filled cells are encountered [S2]. It resumes ray marching in the simulation grid when the iteration reaches the end of the current set of connected partially filled cells [S2]. A key pitfall is that cells with partially filled neighbors must not be skipped to avoid cutting off detailed boundary surfaces potentially formed by particles overlapping cell boundaries; only inner full and empty cells may be skipped [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Allows manual correction of boundaries and supports hidden-line removal for interactive viewing [S1, unofficial].
*   **Limitations:** Relies on explicit 2D boundary detection and inter-slice tessellation [S1, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The THREAD System was developed in 1976 by Mazziotta et al. [S1, unofficial]. It was designed for applications in stereotactic neurosurgery planning, capable of displaying 30 images from 20 sections [S1, unofficial]. The system used threshold segmentation and automatic boundary detection to outline areas of interest, then tessellated between these outlines to define surfaces [S1, unofficial]. It is an early example of surface reconstruction from slice data [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
