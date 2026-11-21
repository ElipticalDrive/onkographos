# Lazy Sweep Ray Casting
Also called: Lazy Sweep, Sweep-Plane Ray Casting

## Overview
Lazy Sweep Ray Casting is a fast algorithm for rendering general irregular grids, including disconnected and nonconvex unstructured grids with holes [S1, unofficial]. It accelerates ray casting by exploiting spatial coherence via a sweep-plane paradigm, allowing rendering costs to decrease as the grid's "disconnectedness" decreases [S1, unofficial]. It distinguishes itself from sibling methods by its ability to handle arbitrary irregular topologies while maintaining low memory consumption and high depth-sorting accuracy [S1, unofficial].

## How it works
**Inputs:** An irregular grid (unstructured mesh) defined by cells and their connectivity; a set of rays defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
**Outputs:** Compositing results for each ray, sorted by depth.

**Data Structures:**
The method relies on a spatial data structure to store boundary cells, avoiding expensive search operations during re-entry for non-convex geometries [S1, unofficial]. The specific layout is not detailed in the provided sources, but it must support efficient retrieval of cells intersected by a sweeping plane [S1, S2, unofficial].

**Steps:**
1.  **Initialization:** For each ray, determine the initial intersection with the grid boundaries.
2.  **Sweep-Plane Advancement:** Instead of advancing ray-by-ray, the algorithm advances a virtual "sweep plane" across the grid. This plane moves in the direction of the ray's dominant axis or a precomputed normal, collecting all cells it intersects [S1, unofficial].
3.  **Cell Processing:** As the sweep plane intersects a cell, the ray's intersection with that cell is computed. The entry and exit points are tracked [S1, S2, unofficial].
4.  **Coherence Exploitation:** The algorithm exploits spatial coherence between adjacent rays. If a ray's path is similar to a previously processed ray, the traversal state can be reused or adjusted, significantly reducing computation [S1, unofficial].
5.  **Compositing:** Intersections are sorted by depth along the ray. Front-to-back compositing is performed to handle transparency and occlusion [S1, unofficial].

**Formulas:**
The traversal time to the next wall in an irregular grid is computed using the inverse velocity [S1, unofficial]:
$$
t_{hit} = (W_{next} - o) \cdot \frac{1}{d}
$$
where $W_{next}$ is the position of the next wall/cell boundary, $o$ is the ray origin, and $d$ is the ray direction magnitude [S1, unofficial].

**Parameters:**
Typical values for the sweep plane step size or coherence window are not given in the sources [S1, unofficial].

**Edge Cases:**
-   **Nonconvex/Holes:** The method handles disconnected and nonconvex grids with holes by using a hierarchical spatial data structure to store boundary cells, ensuring correct re-entry [S1, S2, unofficial].
-   **Resolution Mismatch:** The algorithm is tailored to exploit spatial coherence even if the image resolution differs substantially from the object space resolution [S1, unofficial].

## Subtopics

### [Garrity Ray Casting](7132-garrity-ray-casting.md)
Garrity Ray Casting is a method for rendering unstructured meshes that exploits intrinsic mesh connectivity to optimize ray traversal by tracking entrance and exit points of a ray from cell to cell. It distinguishes itself from sibling methods like Bunyk et al.'s approach by using a hierarchical spatial data structure to store boundary cells, thereby avoiding expensive search operations during re-entry when rays traverse non-convex geometries.

## Operating characteristics
-   **Speed:** Substantially faster (by up to two orders of magnitude) than other software-implemented algorithms for irregular grids [S1, unofficial].
-   **Memory:** Low memory consumption [S1, unofficial].
-   **Accuracy:** High depth-sorting accuracy [S1, unofficial].
-   **Portability:** No hardware dependencies [S1, unofficial].
-   **Limitations:** Performance depends on the "disconnectedness" of the grid; highly disconnected grids may see reduced acceleration benefits [S1, unofficial].

## Background
Lazy Sweep Ray Casting was introduced by Cláudio T. Silva and Joseph S. B. Mitchell in April 1997 [S1, unofficial]. It is based on the sweep-plane paradigm and was designed to address the inefficiencies of existing ray casting methods for irregular grids [S1, unofficial]. Theoretical results, including lower and upper bounds on complexity, were established alongside practical implementations [S1, unofficial].

## Sources
[S1] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
