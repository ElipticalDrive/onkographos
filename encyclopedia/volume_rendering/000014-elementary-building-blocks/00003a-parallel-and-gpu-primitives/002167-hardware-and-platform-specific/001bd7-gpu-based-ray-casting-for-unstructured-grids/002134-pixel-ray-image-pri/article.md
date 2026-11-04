# Pixel Ray Image (PRI)

Also called: PRI Spline Construction, PRI knot generation, Ray-cell intersection knot listing

## Overview
The Pixel Ray Image (PRI) is a semi-regular data structure for volume rendering of unstructured grids that replaces the irregular mesh with a pixelized projection plane and 1D linear splines defined on sampling rays [S1, unofficial]. It solves the difficulty of mapping irregular unstructured grid data efficiently to GPU hardware by leveraging coherent sample placement and 2D texture storage, allowing efficient rendering utilizing shader programming and a slice-based rendering scheme similar to hardware-accelerated rendering for regular grids [S1, unofficial]. Unlike direct volume rendering approaches that must overcome cell ordering and per-cell processing bottlenecks, PRI pre-computes ray-cell intersections into a texture-friendly format [S1, unofficial].

## How it works
**Inputs:** Unstructured grid data (e.g., tetrahedral mesh), camera parameters (position, up vector, view direction), sampling plane resolution.
**Outputs:** A packed 2D texture representation of the PRI, ready for GPU rendering.

**Data Structures:**
1.  **Sampling Plane:** A 2D image plane with finite dimensions, defined in space. It has a 2D coordinate system with the origin at the lower left corner. The plane is pixelized with a regular grid of pixel centers at integer offsets from the origin.
2.  **Knot List (per pixel):** A list of knots for each sampling ray. Each knot is a pair $(s, t)$, where $s$ is the scalar value and $t$ is the distance from the sampling plane.
3.  **Linear Spline:** A 1D scalar function represented by consecutive knots $(s_k, t_k)$ and $(s_{k+1}, t_{k+1})$ where $t_k < t_{k+1}$. The scalar field is assumed to change linearly over the segment.

**Steps:**

1.  **Define Sampling Plane:**
    Define a sampling plane using camera parameters:
    *   A position outside the unstructured grid.
    *   An orientation with an up vector.
    *   A view direction pointing towards the grid.
    *   Define a 2D coordinate system on the plane with the origin at the lower left corner.
    *   Pixelize the plane with a regular grid of pixel centers at integer offsets from the origin.
    *   *Alternative Definition:* Define a tight axis-aligned bounding box around the volume, choose one box face as the sampling plane, and use the associated axial direction as the sampling direction [S1, unofficial].

2.  **Cast Sampling Rays:**
    For each pixel on the sampling plane, cast a sampling ray into the unstructured grid.

3.  **Find Intersections (Knot Generation):**
    *   If the ray intersects the grid, it enters and exits at boundary faces.
    *   If the scalar field is assumed to be linear within grid cells, locate knots at the intersections between the sampling ray and cell faces.
    *   For each intersection, compute the distance $t$ from the originating pixel to the intersection location.
    *   Reconstruct the scalar field value $s$ across the cell face at the intersection.
    *   Insert the pair $(s, t)$ into the knot list for the spline associated with that pixel.
    *   *Alternative:* If the scalar field is not linear within cells, samples can be located uniformly along the ray, and the scalar field is reconstructed using an appropriate filter [S1, unofficial].

4.  **Handle Non-Convex Grids (Boundary Knots):**
    For non-convex unstructured grids, a sampling ray may enter and exit the grid more than once. To define regions outside the volume:
    *   If a knot $(s_b, t_b)$ lies on the boundary of the grid, insert an additional knot with value $(0, t_b)$ into the knot list.
    *   This creates consecutive pairs of knots with a multiplicity of two to bound portions of the ray outside the grid [S1, unofficial].

5.  **Spline Simplification (Data Reduction):**
    *   Reduce the knot count on a per-spline basis using a linear spline simplification algorithm (e.g., greedy algorithm) [S1, unofficial].
    *   This step identifies spans of nearly linear scalar values and removes superfluous knots.
    *   *Note:* The sources mention this step exists and is used to reduce sample count, but do not provide the specific algorithm details for the simplification itself [S1, unofficial].

6.  **Pack into Texture:**
    *   Pack the entire collection of splines onto 2D textures (see [PRI Texture Storage](2493-pri-texture-storage.md)).
    *   Upload the textures to GPU memory prior to rendering [S1, unofficial].

**Formulas:**
*   **Knot Definition:** Each knot is defined by the value pair $(s,t)$, where $s$ is a scalar value and $t$ is the knot's distance to the sampling plane [S1, unofficial].
*   **Linear Segment:** A segment of a spline is defined by two consecutive knots $(s_k,t_k)$ and $(s_{k+1},t_{k+1})$, where $t_k < t_{k+1}$, and the scalar field is assumed to change linearly over the segment [S1, unofficial].
*   **Approximation Error (MSE):** Used to evaluate the quality of the PRI representation $P$ against the original grid $V$:
    $$MSE(V, P) = \frac{\sum_{\forall t \in V} \varepsilon_P(t)^2 |t|}{\sum_{\forall t \in V} |t|}$$
    where $t$ is a tetrahedron in $V$, $|t|$ is the volume of tetrahedron $t$, and $\varepsilon_P(t) = \max_{\forall v \in t} |S(v) - S_P(v)|$. $S(v)$ is the scalar value at vertex $v$ in the original grid, and $S_P(v)$ is the value reconstructed from PRI $P$ by trilinear interpolation [S1, unofficial].

**Parameters:**
*   **Pixel Resolution:** The sampling plane is pixelized. In reported tests, the resolution was $256 \times 256$ [S1, unofficial].
*   **Resolution Choice:** The pixel resolution of the sampling plane must be chosen so that multiple rays pierce every cell face to ensure sufficient resampling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Convex Grids:** Rays may enter and exit the grid multiple times. This is handled by inserting boundary knots with value 0 to mark regions outside the volume, creating consecutive pairs of knots with multiplicity two [S1, unofficial].
*   **Insufficient Resampling:** If the pixel resolution is too low, not multiple rays will pierce every cell face, leading to insufficient resampling of the grid [S1, unofficial].

## Subtopics

### [PRI Texture Storage](2493-pri-texture-storage.md)
PRI Texture Storage is a GPU-friendly data layout for the Pixel Ray Image (PRI) structure, designed to map irregular unstructured grid data onto coherent 2D texture memory. It replaces the irregular connectivity of unstructured grids with two packed 2D textures—an index texture and a knots texture—allowing efficient ray traversal and interpolation on hardware. It sets itself apart from generic unstructured grid rendering by leveraging pre-determined pixel traversal to pack splines contiguously, minimizing texture fetch overhead and enabling fast "outside volume" culling.

## Operating characteristics
*   **Time Complexity:** The construction time depends on the number of tetrahedra and the complexity of the grid. Reported construction times using depth peeling range from 4.39 seconds to 86.04 seconds for datasets with 12,936 to 215,040 tetrahedra, respectively [S1, unofficial].
*   **Memory Complexity:** The PRI is stored in packed 2D textures. The number of knots per spline varies by dataset, with reported averages ranging from 27 to 85 knots per spline for a $256 \times 256$ resolution [S1, unofficial].
*   **Performance:** The PRI allows for efficient rendering utilizing shader programming and a slice-based rendering scheme similar to hardware-accelerated rendering approaches [S1, unofficial].
*   **Strengths:** Flexible sample placement, efficient mapping to graphics hardware, leverages hardware performance trends, fewer samples needed than regular resampling, scalar coherency can be used to further reduce sample count, easily rendered with a slicing approach using only 2D textures [S1, unofficial].
*   **Limitations:** Requires preprocessing steps (resampling, simplification, packing) before rendering. The construction time can be significant for large datasets [S1, unofficial].
*   **When to Use:** When rendering unstructured grid data on GPU hardware where efficient mapping and hardware acceleration are desired, and when the grid is not excessively complex or non-convex to the point of causing excessive knot multiplicity [S1, unofficial].
*   **When Not to Use:** When real-time construction is required without preprocessing, or when the grid is very simple and regular grid rendering is sufficient [S1, unofficial].

## Background
The Pixel Ray Image (PRI) was introduced in the paper "An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering" [S1, unofficial]. It addresses the limitations of earlier algorithms like Projected Tetrahedra (PT) [ST90], which required sorting tetrahedrons and rasterizing them one at a time, and improvements by Roettger [RKE00] that interpolated scalars and used pre-integrated transfer functions. PRI represents a shift towards image-based approaches that allow for flexible sample placement and efficient mapping to graphics hardware [S1, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
