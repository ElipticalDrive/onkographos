# THREAD System
Also called: Three dimensional Reconstruction And Display

## Overview
The THREAD System reconstructs 3D surfaces from a sequence of 2D medical image slices by tessellating between detected outlines on consecutive planes [S1, unofficial]. It sets itself apart from volumetric grid methods by operating directly on discrete slice contours rather than interpolating a full 3D scalar field [S1, unofficial].

## How it works
**Inputs:** A sequence of 2D image slices (e.g., 20 sections) containing scalar intensity data representing a sectioned object [S1, unofficial].
**Outputs:** A set of surface polygons representing the reconstructed 3D surface, ready for rendering with hidden-line removal and shading [S1, unofficial].

**Data Structures:**
1. **Slice Outlines:** For each slice $i$, a list of 2D coordinates $(x, y)$ defining the boundary of the object of interest [S1, unofficial].
2. **Surface Patches:** A collection of polygonal faces connecting vertices from slice $i$ to slice $i+1$ [S1, unofficial].

**Steps:**
1. **Threshold Segmentation:** Apply threshold segmentation with optional manual correction to each slice to separate the object of interest from the background [S1, unofficial].
2. **Automatic Boundary Detection:** Identify connected component outlines on the thresholded slices [S1, unofficial].
3. **Inter-slice Tessellation:** Define surfaces by tessellating the area between the outlines on consecutive slices [S1, unofficial].
4. **Shading:** Apply a simple surface-normal based shading algorithm to the resulting surface polygons [S1, unofficial].
5. **Rendering:** Display the sectioned object outlines and reconstructed surfaces at different angles, with or without hidden-line removal [S1, unofficial].

**Parameters:**
* **Threshold Value:** Determines the segmentation boundary [G]. Not given in the sources.
* **Number of Slices:** The method was demonstrated with 20 sections [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Topology Changes:** If the object's topology changes between slices (e.g., splitting or merging), the correspondence between outlines on slice $i$ and $i+1$ becomes ambiguous [G].
* **Missing Outlines:** If a slice contains no object (background only), the tessellation step must handle the absence of vertices [G].

## Subtopics
[Simulation Grid Cell Skipping](4474-simulation-grid-cell-skipping.md): An optimization technique that accelerates surface extraction by skipping over empty and fully occupied simulation grid cells, processing only partially filled cells [S1, unofficial]. It leverages coarse-grained cell status to bypass uniform data, building on the THREAD System's ray-marching framework [S1, unofficial].

## Operating characteristics
**Time Complexity:** Not given in the sources [S1, unofficial].
**Memory Complexity:** Not given in the sources [S1, unofficial].
**Measured Performance:** The system was demonstrated processing 20 sections to display 30 images [S1, unofficial].
**Strengths:** Simple implementation using standard polygonal tessellation and surface-normal based shading [S1, unofficial].
**Limitations:** Resolution is constrained by the slice spacing and 2D image resolution [G].
**When to use:** The sources do not specify operational guidelines for specific use cases [S1, unofficial].
**When not to use:** The sources do not specify operational guidelines for specific use cases [S1, unofficial].

## Background
The THREAD System was developed in 1976 by Mazziotta et al. [S1, unofficial]. It was designed for stereotactic neurosurgery planning to visualize internal anatomical structures from sectioned object outlines [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
