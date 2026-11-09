# Outline and Contour Generation


## Overview
Outline and Contour Generation is a family of illustrative volume rendering techniques that emphasize object boundaries, silhouettes, and topological features using line primitives or stylized edges rather than photorealistic shading. It solves the problem of visual clutter and ambiguity in complex volumetric data by providing clear structural context and separating distinct objects. This family is distinguished from its siblings by its focus on explicit geometric or image-space edge extraction to define boundaries, whereas [Line and Edge Detection](5341-line-and-edge-detection.md) is the broader category encompassing these methods, and [Face-based volume rendering with edge highlights](2453-face-based-volume-rendering-with-edge-highlights.md) focuses on highlighting mesh edges rather than generating standalone contour lines.

## Shared principles
The family shares the fundamental goal of enhancing perceptual clarity through non-photorealistic edge representation. Common mechanisms include:
1.  **Edge Detection:** Identifying boundaries based on geometric properties (normals, curvature, topology) or image-space properties (depth, normal discontinuities).
2.  **Line Rendering:** Converting detected edges into line primitives (strips, segments) or stylized strokes.
3.  **Compositing:** Integrating these lines with the underlying volume or surface rendering, often using depth buffering for hidden line removal.

## Subtopics

### [Object Contour NPR](787-object-contour-npr.md)
Generates view-dependent lines demarcating surface turns away from the viewer using particle tracing and 4D gradient regression.

### [Lasso Contour Rendering](1855-lasso-contour-rendering.md)
An interactive method allowing users to manually define regions of interest via freehand selection, rendered dynamically in real-time.

### [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md)
Regulates contour thickness based on normal curvature to maintain constant visual appearance, addressing the thickness variation issues of standard dot-product contours.

### [Silhouette Rendering](7959-silhouette-rendering.md)
Separates geometric edge extraction from rasterization, allowing for independent stylization and hidden line removal of object silhouettes.

### [Topological Landscape](8481-topological-landscape.md)
Maps the contour tree of a scalar field onto a 3D terrain model to visualize topological events (merges/splits) as peaks and valleys.

### [Boundary Enhancement Volume Style](10863-boundary-enhancement-volume-style.md)
A post-processing/compositing style that highlights feature bounds and reduces noise using local gradient analysis, rather than generating explicit geometric contours.

### Image Space Outline Generation
- **Definition**: A pixel-based method to generate simple outlines by extracting edges from depth or normal maps, observing that depth differences are minimal between adjacent pixels of a single object and large between objects, and normals vary smoothly across a single surface but show discontinuity between surfaces [S1, unofficial].
- **Steps**:
    1. Extract edges from a depth map or normal map using an edge detection mechanism, such as the Sobel filter [S1, unofficial].
    2. Combine detected depth and normal edges to yield perceptually significant silhouettes [S1, unofficial].
    3. For further processing (e.g., modifying line qualities), calculate analytical curves from the detected edge map, as detected edges only exist as lists of pixels [S1, unofficial].
- **Comparison**: Depth edges reflect C0 discontinuities, while normal edges reflect C1 discontinuities; combining both yields all perceptually significant silhouettes [S1, unofficial].

### Ghosted Views with Contours
- **Steps**:
    1. Pre-segment the most interesting internal structures (e.g., small internal organs) [S2, unofficial].
    2. Identify context information (e.g., the body) that obscures the interesting structures [S2, unofficial].
    3. In areas of occlusion, reduce the visual representation of the context object to contours [S2, unofficial].

## Background
The concept of using outlines to enhance visual clarity in computer graphics has roots in early non-photorealistic rendering research. Hertzmann provided a survey of image space outline generation algorithms, recommending the use of depth and normal maps for edge extraction [S1, unofficial]. Schlechtweg and Raab described flexible line models for expressive visualization [S1, unofficial]. In the context of volume rendering, illustrative techniques have been applied to medical and scientific data to improve the understanding of spatial arrangements, such as in CT-angiography and ghosted views where context objects are reduced to contours to reveal internal structures [S2, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
