# Object-Order Ray Casting with Min/Max Octrees
Also called: Mora orthogonal object-order
## Overview
Object-Order Ray Casting with Min/Max Octrees is a software-based volume rendering technique for regular grids that uses orthogonal projection and pre-computed min/max octrees to skip empty (transparent) regions [S1, unofficial]. It processes visible volume cells directly rather than pixels, distinguishing it from image-order methods [S1, unofficial].
## How it works
**Inputs:** Regular 3D grid of voxel data; pre-computed min/max octree; orthogonal view direction [S1, unofficial].
**Outputs:** 2D image containing accumulated results for visible cells [S1, unofficial].
**Data Structures:**
1. **Min/Max Octree:** Hierarchical structure storing minimum and maximum scalar values per sub-volume node to skip empty or transparent regions [S1, unofficial].
2. **Hexagon Template:** 2D projection mask of a cubic sub-volume [S1, unofficial]. Orthogonal projection ensures every sub-volume projects to the same hexagon template, differing only by translation [S1, unofficial].
3. **Hierarchical Occlusion Maps (HOMs):** Images of different sizes used in isosurface rendering to indicate whether a ray must be traversed or a previously computed value can be reused [S1, unofficial].
**Steps:**
1. **Pre-computation:** Construct the min/max octree from the input volume data [S1, unofficial].
2. **Sub-volume Division & Projection:** Divide the volume into sub-volumes of $m^3$ voxels [S1, unofficial]. Project each sub-volume onto the image plane using orthogonal projection [S1, unofficial].
3. **Visibility Culling:** Use the min/max octree to determine if a sub-volume contains relevant data [S1, unofficial]. Skip sub-volumes marked as empty or fully transparent [S1, unofficial].
4. **Ray Traversal & HOM Check:** For visible sub-volumes, process rays intersecting the hexagonal footprint [S1, unofficial]. For isosurface rendering, check the HOMs to determine if a ray must be traversed or if a previously computed value can be reused [S1, unofficial].
**Parameters:**
* $m$: Sub-volume side length in voxels. Not given in the sources.
**Edge Cases and Pitfalls:**
* The method relies on orthogonal projection to maintain the constant hexagonal footprint property [S1, unofficial].
* HOMs are specifically applied for isosurface rendering to enable ray reuse [S1, unofficial].
## Subtopics

### [Hardware-Adapted Object-Order Ray Casting](1824-hardware-adapted-object-order-ray-casting.md)
Hardware-Adapted Object-Order Ray Casting adapts this software method for modern graphics hardware by generating fragments for rays intersecting volume cells, using pre-computed projection lists for orthogonal views. It is distinguished from its parent method by its explicit optimization for hardware fragment generation and pre-computed projection lists. Choose it when targeting graphics hardware for orthogonal views.

## Operating characteristics
* **Performance:** Enables interactive frame rates of 2 to 5 fps for mid-sized data sets (e.g., bonsai, engine models) [S1, unofficial].
* **Memory:** Uses a compact data set representation taking advantage of implicit neighbors, achieving similar performance with less memory consumption [S1, unofficial].
* **Strengths:** Efficient skipping of empty regions via min/max octrees; reduced computation via HOMs for isosurfaces; simple projection geometry due to orthogonality [S1, unofficial].
* **Limitations:** Restricted to orthogonal projection; software implementation in the original description; performance depends on min/max octree construction and traversal [S1, unofficial].
## Background
Proposed by Mora et al. in 2002 as a software implementation for interactive volume ray tracing on regular grids [S1, unofficial]. Designed to handle mid-sized data sets efficiently by leveraging orthogonal projection and hierarchical data structures [S1, unofficial].
## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
