# THREAD System
Also called: Three dimensional Reconstruction And Display

## Overview
The THREAD System is an early isosurface extraction method for medical imaging that reconstructs 3D surfaces from a sequence of 2D cross-sectional slices [S1, unofficial]. It solves the problem of visualizing internal anatomical structures by tessellating surfaces between outlines detected on consecutive slices [S1, unofficial]. It sets itself apart from grid-based methods like Marching Cubes by operating directly on slice contours rather than a volumetric grid, relying on interpolation between discrete image planes [S1, unofficial].

## How it works
The method processes a stack of 2D medical images to generate a tessellated 3D surface mesh.

**Inputs:** A sequence of $N$ 2D image slices (e.g., $N=20$) containing scalar intensity data representing a sectioned object [S1, unofficial].
**Outputs:** A set of surface polygons (triangles or quadrilaterals) representing the reconstructed 3D surface, ready for rendering with hidden-line removal and shading [S1, unofficial].

**Data Structures:**
1.  **Slice Outlines:** For each slice $i$, a list of 2D coordinates $(x, y)$ defining the boundary of the object of interest.
2.  **Surface Patches:** A collection of polygonal faces connecting vertices from slice $i$ to slice $i+1$.

**Steps:**
1.  **Threshold Segmentation:** Apply a threshold value to each 2D slice to separate the object of interest from the background. Manual correction may be applied to refine the segmentation [S1, unofficial].
2.  **Automatic Boundary Detection:** Identify the connected component outlines on the thresholded slices. This yields a set of 2D contours for each slice [S1, unofficial].
3.  **Inter-slice Tessellation:** For each pair of consecutive slices $i$ and $i+1$, map the outline vertices from slice $i$ to the corresponding vertices on slice $i+1$. The surface is defined by tessellating the area between these two outlines [S1, unofficial].
4.  **Shading:** Compute surface normals for the resulting polygons. Apply simple surface-normal based shading to provide depth cues [S1, unofficial].
5.  **Rendering:** Display the resulting mesh at various angles, optionally applying hidden-line removal to occlude back-facing or internal surfaces [S1, unofficial].

**Parameters:**
*   **Threshold Value:** Determines the segmentation boundary. Not given in the sources.
*   **Number of Slices:** The method was demonstrated with 20 slices [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Topology Changes:** If the object's topology changes between slices (e.g., splitting or merging), the correspondence between outlines on slice $i$ and $i+1$ becomes ambiguous. The sources do not specify how this is handled [S1, unofficial].
*   **Missing Outlines:** If a slice contains no object (background only), the tessellation step must handle the absence of vertices. Not given in the sources.
*   **Precision:** The method relies on the accuracy of the 2D boundary detection. Errors in slice segmentation propagate directly into the 3D geometry [S1, unofficial].

## Subtopics
## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Strengths:** Capable of displaying 3D structures from a limited number of 2D sections (e.g., 30 images from 20 sections) [S1, unofficial]. Simple implementation using standard polygonal tessellation and shading [S1, unofficial].
**Limitations:** Limited to processing discrete slices; does not utilize volumetric data between slices for smoothing or interpolation beyond linear tessellation. Resolution is constrained by the slice spacing and 2D image resolution [S1, unofficial].
**When to use:** Historical analysis of early medical imaging visualization techniques.
**When not to use:** Modern applications requiring high-fidelity volumetric reconstruction or handling of complex topologies between slices.

## Background
The THREAD System was developed in 1976 by Mazziotta et al. [S1, unofficial]. It was designed for applications in stereotactic neurosurgery planning, allowing surgeons to view sectioned object outlines at different angles with or without hidden-line removal [S1, unofficial]. The system represents an early approach to 3D reconstruction from 2D medical images, predating widespread grid-based isosurface extraction methods.

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
