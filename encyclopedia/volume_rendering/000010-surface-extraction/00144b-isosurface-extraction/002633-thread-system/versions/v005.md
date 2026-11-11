# THREAD System

Also called: Three dimensional Reconstruction And Display

## Overview
The THREAD System is an early isosurface extraction method developed for medical imaging that reconstructs 3D surfaces from a sequence of 2D cross-sectional slices [S1, unofficial]. It solves the problem of visualizing internal anatomical structures by tessellating surfaces between outlines detected on consecutive slices [S1, unofficial]. It sets itself apart from grid-based methods like Marching Cubes by operating directly on discrete slice contours rather than a volumetric scalar field, relying on interpolation between adjacent 2D boundaries [S1, unofficial].

## How it works
The method processes a stack of 2D images representing cross-sections of an object.

**Inputs and Outputs:**
- **Input:** A sequence of $N$ 2D images (slices), typically $N=20$ as in the original implementation [S1, unofficial].
- **Output:** A 3D polygonal mesh representing the surface of the object of interest, shaded with surface normals [S1, unofficial].

**Data Structures:**
- **Slice Outlines:** For each slice $i$, a set of 2D coordinates $(x, y)$ defining the boundary of the object. These are derived via threshold segmentation and automatic boundary detection [S1, unofficial].
- **Surface Patches:** Triangular or polygonal faces connecting vertices from slice $i$ to slice $i+1$.

**Steps:**
1. **Threshold Segmentation:** Apply threshold segmentation to each of the $N$ images to isolate the object from the background. Manual correction may be applied to refine the outlines [S1, unofficial].
2. **Automatic Boundary Detection:** Use an automatic boundary detection scheme to extract the 2D outlines of the area of interest from the segmented images [S1, unofficial].
3. **Tessellation:** Define the 3D surface by tessellating between the outlines on consecutive slices ($i$ and $i+1$). This involves connecting vertices from the outline of slice $i$ to vertices on the outline of slice $i+1$ to form surface patches [S1, unofficial].
4. **Shading:** Apply a simple surface-normal based shading algorithm to the generated surface polygons to provide visual depth cues [S1, unofficial].
5. **Rendering:** Display the resulting 3D model at different angles, supporting both hidden-line removal and wireframe views [S1, unofficial].

**Parameters:**
- **Number of Slices:** The original system processed 20 slices to display 30 images [S1, unofficial].
- **Isosurface Threshold:** Determined by the threshold segmentation step [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Slice Spacing:** The quality of the tessellation depends on the distance between slices; large gaps may require more complex interpolation or result in jagged surfaces [G].
- **Outline Correspondence:** Matching vertices between consecutive outlines requires a consistent ordering or correspondence algorithm to avoid twisted or self-intersecting patches [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The system was capable of displaying 30 images from 20 sections [S1, unofficial].
- **Strengths:** Conceptually simple; naturally renders curved surfaces using surface patches; effective for medical data visualization [S1, unofficial].
- **Limitations:** Relies on sequential slices; may struggle with complex topology changes between slices; limited to the specific medical imaging context described [S1, unofficial].

## Background
The THREAD System was developed in 1976 by Mazziotta et al. [S1, unofficial]. It was designed for applications in stereotactic neurosurgery planning, specifically for the volumetric rendering of medical data [S1, unofficial]. The system utilized threshold segmentation and automatic boundary detection to outline areas of interest on 20 images, which were then tessellated to define surfaces [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
