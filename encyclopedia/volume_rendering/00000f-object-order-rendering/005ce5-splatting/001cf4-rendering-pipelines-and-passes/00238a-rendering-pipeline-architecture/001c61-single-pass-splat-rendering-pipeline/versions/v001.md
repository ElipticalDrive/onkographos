# Single-Pass Splat Rendering Pipeline

Also called: Single-Pass High-Quality Splatting

## Overview
A real-time splat rendering pipeline that combines splat and mesh rendering components to enable high-quality shader effects (reflection, refraction) in a single rendering pass, avoiding multi-pass blending artifacts and reducing memory/workload compared to dense point sets [S1, unofficial]. It differs from sibling methods like the [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md) by replacing screen-space blending with object-space attribute interpolation on regular grid patches, allowing for complex lighting in one pass [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point set representing a surface, pre-processed into patches. Each patch consists of a planar domain, a height field (elevation map), and optional attribute maps (color, normals).
*   **Output:** A rendered image with continuous attributes and high-quality shading effects.

**Data Structures:**
The model is stored completely on graphics hardware. Connectivity is implicit.

1.  **Patch Data Structure:**
    *   **Plane Parameters:** A 3D point (origin), a normal vector, and grid resolution ($N \times M$).
    *   **Height Field:** A rectangular array of height values ($h_{i,j}$).
    *   **Flags:** A special height value indicates that the patch does not include a point above that grid point (allowing arbitrary support shapes) [S1, unofficial].
    *   **Attribute Maps:** Optional texture maps (color, normal) of arbitrary resolution, stored as texture objects [S1, unofficial].
    *   **Transformation:** One $4 \times 4$ transformation matrix per patch [S1, unofficial].

2.  **Vertex Buffer Object (VBO):**
    *   Stores point positions and elevation data for each patch.
    *   Point positions on the patch plane are expressed by a single index value (combined row and column address in the sampling grid) [S1, unofficial].

3.  **Index Buffer:**
    *   A linear list $1, 2, \dots, n$, where $n$ is the number of points in the patch.
    *   No connectivity encoding is needed [S1, unofficial].

4.  **Texture Objects:**
    *   Store additional attribute maps (color, normal) for each patch individually [S1, unofficial].

**Algorithm Steps:**

1.  **Patch Construction (Pre-processing):**
    *   Partition the input 3D point set into patches. Each patch is parameterized as an elevation map over a planar domain [S1, unofficial].
    *   Resample the surface patch on a regular grid using Moving Least Squares (MLS) surface reconstruction [S1, unofficial].
    *   **Overlap Handling:** To prevent gaps due to MLS reconstruction limitations, patches slightly overlap. An input point is included in a patch if its $\epsilon$-neighborhood intersects the point set of the region, where $\epsilon > 0$ [S1, unofficial].
    *   Resample attribute fields (normals, textures) onto the regular grid. Normals can be computed as MLS surface normals or interpolated from input point normals using bilinear interpolation in a Delaunay triangulation of projected points [S1, unofficial].

2.  **Data Upload:**
    *   Upload VBOs (elevation data) and Texture Objects (attributes) to GPU memory once before rendering [S1, unofficial].

3.  **Rendering Loop:**
    *   Iterate over all patches.
    *   For each patch:
        1.  Address the respective texture objects.
        2.  Trigger the corresponding VBO for rendering [S1, unofficial].

4.  **Vertex Shader:**
    *   **Translation:** Translate point positions according to their elevation values. The position $P$ on the patch plane is offset along the plane normal $N$ by the height $h$:
        $$ P_{translated} = P_{grid} + h \cdot N $$
        where $P_{grid}$ is the base grid point on the support plane [S1, unofficial].
    *   **Rotation:** Rotate the translated points using the patch's $4 \times 4$ transformation matrix to place them on the surface of the original model [S1, unofficial].
    *   **Radius Calculation:** Set the radius of each point sprite. The base radius is $\frac{1}{2}\sqrt{2}\mu$, where $\mu$ is the sampling step size. Adjust radii based on the angle between neighboring elevation values to prevent holes [S1, unofficial]. The formula is:
        $$ r = \max(r_{min}, \min(r_{max}, c_1 \cdot \frac{1}{\cos(\theta)} + c_2 \cdot \Delta h)) $$
        where $\theta$ is the angle between normals, $\Delta h$ is the variance between neighboring elevation values, and $c_1, c_2$ are scaling constants. $r_{min}$ and $r_{max}$ establish minimal and maximal radii [S1, unofficial].

5.  **Fragment Shader:**
    *   **Splat Simulation:** Transform point-sprite fragments to simulate splatting, as point sprites are screen-aligned primitives [S1, unofficial].
    *   **Attribute Interpolation:** Determine per-fragment UV coordinates using continuous attribute interpolation. This involves projecting pixel shader fragments back onto the patches to interpolate attributes from nearby points, replacing disadvantageous splat blending [S1, unofficial].
        *   View rays are intersected with the splats for direct evaluation of UV coordinates.
        *   Intersecting the view ray with the plane through neighboring splat centers provides corrected interpolation of UV coordinates [S1, unofficial].
    *   **Illumination:** Perform final illumination by summing contributions from:
        *   Lighting (e.g., Phong shading)
        *   Texture mapping
        *   Reflectance mapping (using cube mapping)
        *   Refraction mapping [S1, unofficial].

**Parameters:**
*   $\epsilon$: Positive value for patch overlap during construction. Typical value: Not given in the sources.
*   $\mu$: Sampling step size of the grid.
*   $c_1, c_2$: Scaling constants for radius adjustment. Typical values: Not given in the sources.
*   $r_{min}, r_{max}$: Minimal and maximal splat radii. Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Gaps between patches:** MLS resampling cannot guarantee no gaps. Patches must overlap slightly using $\epsilon$ to ensure continuous error-free rendering [S1, unofficial].
*   **Holes due to elevation variance:** High variance between neighboring elevation values increases spatial gaps. Radii must be increased in these areas [S1, unofficial].
*   **Screen-aligned primitives:** Point sprites are screen-aligned, so fragment shader transformation is mandatory to simulate splatting correctly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Single rendering pass. This simplifies overall rendering complexity compared to current GPU-based splatting techniques which require at least three rendering passes [S1, unofficial].
*   **Memory Complexity:** The resampled model is stored completely on graphics hardware. Connectivity information is implicit and does not require additional memory [S1, unofficial].
*   **Performance:** Accelerates rendering and simplifies current splatting techniques [S1, unofficial]. Avoids performance-hungry GPU functionality such as multiple render targets and additional blending passes [S1, unofficial].
*   **Strengths:**
    *   Enables high-quality shader effects (reflection, refraction) without densely sampled models [S1, unofficial].
    *   Improves quality of reconstructed attributes (normals, texture coordinates) by interpolating in object space rather than blending in screen space [S1, unofficial].
    *   Continuous attributes on and between splats [S1, unofficial].
*   **Limitations:**
    *   Requires pre-processing to construct patches and resample data [S1, unofficial].
    *   Relies on GPU vertex and fragment shader functionality and point-sprite extension [S1, unofficial].
*   **When to use:** Real-time rendering of point-sampled surfaces where high-quality shading effects are required and performance is critical.
*   **When not to use:** When dynamic topology changes require frequent re-meshing or when the input data is not available in a compressed/patched form [S1, unofficial].

## Background
*   **Origin:** Introduced by Tilo Ochotta, Stefan Hiller, and Dietmar Saupe in September 2006 [S1, unofficial].
*   **Publication:** Published as Konstanzer Schriften in Mathematik und Informatik Nr. 219 [S1, unofficial].
*   **Development:** Builds on previous work on elevation map compression [16] and Moving Least Squares surface reconstruction [2, 1] [S1, unofficial].
*   **Usage:** Used for real-time splat rendering with enhanced shader effects in entertainment applications [S1, unofficial].

## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
