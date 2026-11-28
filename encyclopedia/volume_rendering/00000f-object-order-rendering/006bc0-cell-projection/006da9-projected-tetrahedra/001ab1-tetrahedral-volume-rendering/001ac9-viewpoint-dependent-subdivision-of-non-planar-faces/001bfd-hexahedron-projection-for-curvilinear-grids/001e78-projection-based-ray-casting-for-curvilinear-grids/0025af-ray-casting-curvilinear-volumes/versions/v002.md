# Ray-casting Curvilinear Volumes

Also called: Direct ray-casting of curvilinear volumes

## Overview
Ray-casting Curvilinear Volumes is a direct volume rendering technique for data sampled on curvilinear grids (six-sided cells with non-planar, non-orthogonal faces) that casts rays directly through the volume without interpolating to a rectilinear grid first [S1]. It solves the problem of rendering deformed grids while avoiding the accumulation of interpolation errors inherent in resampling methods [S1]. Unlike interpolation-based approaches, this method accommodates irregular sampling densities but is generally slower due to the lack of simplifying assumptions available for regular grids [S1].

## How it works
**Inputs and Outputs:**
- Input: A curvilinear volume dataset consisting of six-sided cells (hexahedra) with associated scalar values at grid nodes.
- Output: A 2D image with accumulated color and opacity values per pixel.

**Data Structures:**
- **Element Representation:** Each curvilinear element (six-sided cell) is represented by its twelve bounding triangles to ensure planarity for intersection calculations [S1].
- **Acceleration Structures:** A scan line list and an active element list are maintained to reduce the number of intersection calculations performed for each ray [S1].

**Steps:**
1. **Initialization:** Construct the acceleration structures (scan line list and active element list) based on the bounding boxes or projections of the curvilinear elements [S1].
2. **Ray Casting:** For each pixel in the image plane, cast a ray into the volume [S1].
3. **Intersection Testing:** Determine intersections between the ray and the curvilinear elements.
    - The ray is tested against the twelve triangles bounding each element [S1].
    - Assuming the element is convex, if the ray intersects the element, two intersection points (entry and exit) are generated [S1].
    - Intersection calculations use the standard technique for triangle primitives [S1].
4. **Data Sampling:** Sample the scalar data values along the ray segment inside the element. Since the grid is curvilinear, interpolation is performed within the element based on the ray's position relative to the element's local coordinates or node positions [S1].
5. **Shading and Accumulation:** Apply shading methods similar to those used for rectilinear volumes to the element [S1]. Accumulate the color and opacity contributions into the pixel buffer using standard volume rendering integration (e.g., front-to-back compositing) [S1].
6. **Termination:** Repeat for all rays until the image is complete.

**Parameters:**
- Typical values for acceleration structure parameters are not given in the sources.
- Shading parameters (transfer functions) are not specified beyond "similar to rectilinear volumes" [S1].

**Edge Cases and Pitfalls:**
- **Non-planar Faces:** Curvilinear faces may be non-planar and non-orthogonal, requiring decomposition into triangles for intersection [S1].
- **Convexity Assumption:** The method assumes convexity to guarantee exactly two intersections per element if an intersection occurs [S1]. Non-convex elements may require more complex handling.
- **Performance:** The potentially large number of elements necessitates efficient data structures to avoid excessive intersection calculations [S1].

## Subtopics

### [Analytic Rendering of Curvilinear Volume Data](22982-analytic-rendering-of-curvilinear-volume-data.md)
Analytic rendering of curvilinear volume data is a direct volume rendering technique for data sampled on curvilinear grids (six-sided cells with non-planar faces) that represents each voxel as a cubic triparametric Bezier solid. It solves the problem of rendering deformed grids without interpolating to a rectilinear grid or approximating voxels with triangles/tetrahedra, ensuring $C^1$ continuity of both geometry and data values across voxel boundaries. Unlike footprint-based methods that require recomputing footprints for varying voxel shapes, this method renders each voxel analytically directly into the accumulating image using a 3D forward difference operator.

### Gridigration
Gridigration is a volume rendering technique for 3D scalar fields defined on curvilinear grids that renders by casting rays along grid lines, where each ray deposits a scalar result on a vertex of the grid's face using a deposition function applied to intersected data points [S2, unofficial]. It casts rays along grid lines, applies a deposition function to the data points the ray intersects to compute a scalar result, and deposits the resulting scalar on a vertex of the grid's face [S2, unofficial]. The resulting scalar field on a surface embedded in three dimensions is visualized using standard techniques [S2, unofficial]. Interpretation of results can be difficult [S2, unofficial]. The technique is $O(n)$ where $n$ is the number of grid nodes [S2, unofficial]. Gridigration is much faster than traditional ray casting volume rendering on identical workstation hardware and somewhat faster than traditional ray casting on a massively parallel supercomputer [S2, unofficial]. It is used for quick look surveys of scalar fields to find features of interest [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The method is generally slower than interpolation-based methods because it cannot make the simplifying assumptions in traversing regular grids [S1]. Gridigration, a variant, is $O(n)$ where $n$ is the number of grid nodes [S2, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Gridigration is much faster than traditional ray casting volume rendering on identical workstation hardware [S2, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Less prone to accumulation of interpolation errors; easily accommodates irregular sampling densities [S1].
- **Limitations:** Generally slower than interpolation-based methods; interpretation of results can be difficult for variants like Gridigration [S1][S2, unofficial].
- **When to use:** When avoiding interpolation errors is critical or when dealing with irregular sampling densities [S1].
- **When not to use:** When speed is the primary concern and the grid can be reasonably interpolated to a rectilinear grid [S1].

## Background
Direct volume rendering creates an image of sampled volume data by mapping from the data values directly to pixel contributions, without the creation of intermediate geometric representations [S1]. Early methods assumed rectilinear grid data [S1]. This method extends ray casting methods for rectilinear volumes to curvilinear volumes [S1]. The concept of direct rendering on curvilinear grids was discussed in the context of comparing it with interpolation-based approaches [S1]. Gridigration was introduced as a very fast volume renderer for 3D scalar fields defined on curvilinear grids [S2, unofficial].

## Sources
[S1] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S2] Gridigrator:A Very Fast Volume Rendererfor 3D Scalar Fields Defined on Curvilinear Grids, http://alglobus.net/NASAwork/papers/RNR-92-001/gridigrator.html, fetched 2026-10-30 (unofficial)
