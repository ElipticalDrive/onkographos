# Spherical Tessellation for Normal Quantization
Also called: Voltx Normal Quantization Tessellation

## Overview
Spherical Tessellation for Normal Quantization is a geometric method used in the Voltx direct volume rendering system to quantize normal vectors and generate triangle meshes for rendering concentric spherical shells [S1, unofficial]. It addresses the problem of perspective viewing slab thickness variations, which cause shading artifacts when using planar slabs [S1, unofficial]. Unlike planar tessellation, this method generates triangles suitable for spherical surfaces to compensate for these variations, particularly at the periphery of the image [S1, unofficial].

## How it works
The method defines a specific tessellation of the unit sphere to serve as the basis for normal-vector quantization [S1, unofficial]. While the parent method (Data-Dependent Normal Vector Quantizer) optimizes the quantizer indices via simulated annealing, this method provides the geometric structure (the triangles) onto which those normals are mapped or from which they are derived for rendering [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A unit sphere domain.
*   **Output:** A set of triangles forming a tessellation of the sphere.

**Data Structures:**
*   **Triangle Mesh:** A list of triangles, where each triangle is defined by three vertices on the unit sphere. The vertices represent the quantized normal directions.
*   **Lookup Table:** A mapping from quantized normal indices to their corresponding 3D vector coordinates, used for efficiency during rendering [S1, unofficial].

**Steps:**
1.  **Sphere Tessellation:** Generate a tessellation of the unit sphere. The specific topology is not detailed in the provided sources, but it is distinct from octahedral or dodecahedral subdivisions [S1, S2, unofficial].
2.  **Normal Quantization:** Use the vertices of the tessellation triangles as the discrete set of normal vectors. These vectors are stored in a lookup table for fast access during the rendering phase [S1, unofficial].
3.  **Shell Generation:** Use the tessellation triangles to generate concentric spherical shells. These shells replace planar slabs in the rendering pipeline [S1, unofficial].
4.  **Rendering:** Apply the 3D texture map (containing color and opacity) to the stack of spherical shells. Composite the shells to form the final image [S1, unofficial].

**Formulas:**
*   **Slab Thickness Compensation:** The method compensates for the variation in effective slab thickness due to perspective. The error from ignoring this effect is at most 4% on the periphery for a 600-pixel window width at 70 cm viewing distance, but can increase to 14% for a 1000-pixel window width [S1, unofficial].
*   **Gradient Magnitude Interpretation:** The gradient magnitude is interpreted in the context of the field-data value and material classification parameters, not in isolation [S1, unofficial].

**Parameters:**
*   **Viewing Geometry:** Window width and viewing distance affect the magnitude of the error that the tessellation compensates for [S1, unofficial].
*   **Material Classification:** Parameters defining the range of field values for material boundaries (e.g., bone density > 110) influence the gradient-based shading criterion [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Peripheral Error:** For large window widths (e.g., 1000 pixels), the error from planar approximation increases significantly (up to 14%), making spherical tessellation more critical [S1, unofficial].
*   **Gradient Isolation:** Interpreting gradient magnitude in isolation leads to incorrect material classification. It must be combined with field-data values [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storage for the tessellation triangles and a lookup table for quantized normals [S1, unofficial].
*   **Performance:** Provides efficiency through quantized normal vectors and lookup tables [S1, unofficial].
*   **Strengths:** Compensates for perspective viewing slab thickness variations, reducing shading artifacts at image peripheries [S1, unofficial].
*   **Limitations:** The specific tessellation topology is not detailed, making implementation from this text alone difficult without the parent method's context or external sources [S1, unofficial].

## Background
The method was introduced in the Voltx direct volume rendering system [S1, unofficial]. It is part of a pipeline that converts rectilinear data sets into 3D texture maps, uses quantized normals for shading, and renders via a stack of parallel planes or spherical shells [S1, unofficial]. The approach compares favorably to shear-warp factorization [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
