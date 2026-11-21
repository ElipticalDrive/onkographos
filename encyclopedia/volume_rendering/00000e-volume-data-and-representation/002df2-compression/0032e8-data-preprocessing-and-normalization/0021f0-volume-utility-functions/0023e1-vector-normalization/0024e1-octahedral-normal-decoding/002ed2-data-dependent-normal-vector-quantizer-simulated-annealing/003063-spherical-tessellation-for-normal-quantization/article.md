# Spherical Tessellation for Normal Quantization
Also called: Voltx Normal Quantization Tessellation

## Overview
Spherical Tessellation for Normal Quantization is a geometric method in the Voltx direct volume rendering system that quantizes normal vectors and generates triangle meshes for rendering concentric spherical shells [S1, unofficial]. It solves the problem of perspective viewing slab thickness variations that cause shading artifacts when using planar slabs [S1, unofficial]. Unlike planar tessellation, it produces triangles tailored for spherical surfaces to compensate for peripheral thickness variations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A unit sphere domain.
*   **Output:** A set of triangles forming a tessellation of the sphere [S1, unofficial].

**Data Structures:**
*   Exact field layouts, sizes, and bit packing are Not given in the sources.
*   **Triangle Mesh:** A list of triangles where each triangle is defined by three vertices on the unit sphere [S1, unofficial].
*   **Lookup Table:** A mapping from quantized normal indices to their corresponding 3D vector coordinates, used for efficiency during rendering [S1, unofficial].

**Steps:**
1.  **Sphere Tessellation:** Generate a tessellation of the unit sphere [S1, unofficial].
2.  **Normal Quantization:** Use the vertices of the tessellation triangles as the discrete set of normal vectors [S1, unofficial].
3.  **Shell Generation:** Use the tessellation triangles to generate concentric spherical shells to replace planar slabs [S1, unofficial].
4.  **Rendering:** Apply a 3D texture map containing color and opacity to the stack of spherical shells and composite them to form the final image [S1, unofficial].

**Formulas:**
*   **Perspective Error:** Ignoring slab thickness variation causes at most 4% error on the periphery for a 600-pixel window width at 70 cm viewing distance, increasing to 14% for a 1000-pixel window width [S1, unofficial].
*   **Gradient Interpretation:** Gradient magnitude $M$ is evaluated in the context of the field-data value $V$ and material classification parameters $P$, rather than in isolation [S1, unofficial].

**Parameters:**
*   **Viewing Geometry:** Window width and viewing distance determine the magnitude of the perspective error [S1, unofficial].
*   **Material Classification:** Parameters defining the range of field values for material boundaries (e.g., bone density > 110) influence the gradient-based shading criterion [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Peripheral Error:** Large window widths significantly increase planar approximation error, making spherical tessellation critical [S1, unofficial].
*   **Gradient Isolation:** Interpreting gradient magnitude in isolation leads to incorrect material classification; it must be evaluated alongside field-data values [S1, unofficial].

# Pseudocode not provided in the sources.

## Subtopics

### [Dodecahedron Subdivision Normal Quantization](6588-dodecahedron-subdivision-normal-quantization.md)
A static method for generating normal vector directions by tessellating a unit sphere using a subdivided regular dodecahedron. It solves directional aliasing in volume rendering. It is distinguished from its parent by its specific geometric construction using pentagonal faces, and from [Gradient Quantization via Polyhedron Tesselation](#gradient-quantization-via-polyhedron-tesselation) by its fixed 240-vector output rather than a variable count based on refinement levels. Choose it when a fixed, high-quality static quantizer is preferred over data-dependent optimization.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Requires storage for the tessellation triangles and a lookup table for quantized normals [S1, unofficial].
*   **Performance:** Provides efficiency through quantized normal vectors and lookup tables [S1, unofficial].
*   **Strengths:** Compensates for perspective viewing slab thickness variations, reducing shading artifacts at image peripheries [S1, unofficial].
*   **Limitations:** The specific tessellation topology is not detailed in the sources [S1, unofficial].

## Background
The method was introduced in the Voltx direct volume rendering system [S1, unofficial]. It is part of a pipeline that converts rectilinear data sets into 3D texture maps, uses quantized normals for shading, and renders via a stack of parallel planes or spherical shells [S1, unofficial]. The approach compares favorably to shear-warp factorization [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
