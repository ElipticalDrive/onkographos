# Outline and Contour Generation


## Overview
Outline and Contour Generation is a family of illustrative volume rendering techniques that generate view-dependent lines or strokes to emphasize object boundaries, silhouettes, and topological features. It solves the problem of ambiguous spatial structure in scalar volumes by providing explicit geometric cues that distinguish objects from background and highlight internal features. This family is distinguished from its siblings by its focus on generating explicit line primitives (either analytically or via pixel-based extraction) to demarcate surfaces, whereas [Line Illustration Detection](5341-line-and-edge-detection.md) focuses on flow-aligned hatching, and [Face-based volume rendering with edge highlights](2453-face-based-volume-rendering-with-edge-highlights.md) focuses on shading-based edge enhancement rather than explicit line rendering.

## Shared principles
Not given in the sources.

## Subtopics

### [Object Contour NPR](787-object-contour-npr.md)
Generates view-dependent lines that demarcate areas where an object's surface turns away from the viewer, effectively emphasizing physical boundaries without relying on predefined classification functions or voxel intensity values. Unlike hatch-based techniques which are often pre-rendered to texture buffers, this method produces line-strips rendered directly to the screen, allowing for interactive, view-dependent illustration.

### [Lasso Contour Rendering](1855-lasso-contour-rendering.md)
Lasso Contour Rendering is an interactive, pixel-based outline generation method that allows users to manually define regions of interest within a volume using a freehand "lasso" selection tool. Unlike automatic contour methods that detect edges based on scalar field gradients, this approach relies on user intent to highlight specific structures. It is distinguished by its integration into a real-time rendering pipeline where the contour is rendered dynamically only when the user is actively drawing, leveraging offscreen render passes to composite the selection overlay.

### [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md)
Curvature-Controlled Style Contours is a line and edge detection method for illustrative volume rendering that regulates contour thickness based on the normal curvature along the view direction. It solves the problem of uncontrolled thickness variation inherent in standard dot-product contours, which appear too thick on flat surfaces and too thin on fine structures. It sets itself apart from siblings like [Object Contour NPR](787-object-contour-npr.md) and [Silhouette Rendering](7959-silhouette-rendering.md) by explicitly incorporating second-order geometric information (curvature) to maintain constant visual thickness, whereas those methods typically rely on first-order normal-view angle relationships or topological features.

### [Silhouette Rendering](7959-silhouette-rendering.md)
Silhouette rendering is an object-space line rendering technique used to indicate context objects (such as bones) or to discriminate object classes based on specific features (e.g., lymph nodes touching critical structures). It sets itself apart from sibling methods by explicitly separating the geometric extraction of edges from their rasterization, allowing for independent stylization and hidden line removal via a dedicated stroke pipeline.

### [Topological Landscape](8481-topological-landscape.md)
Topological landscape is a visualization technique that maps the topology of a scalar field's contour tree onto a 3D terrain model. It solves the problem of interpreting complex topological events (merges and splits) by leveraging human intuition for reading physical terrain, where peaks represent contour merges and valleys represent contour splits. It sets itself apart from sibling methods like [Object Contour NPR](787-object-contour-npr.md) and [Lasso Contour Rendering](1855-lasso-contour-rendering.md) by abandoning direct geometric rendering of contours in favor of an abstract, height-map-based representation of the data's topological structure.

### Image Space Outline Generation
#### definition
- [S1, unofficial] A pixel-based method to generate simple outlines by extracting edges from depth or normal maps, observing that depth differences are minimal between adjacent pixels of a single object and large between objects, and normals vary smoothly across a single surface but show discontinuity between surfaces.
#### step
- [S1, unofficial] Extract edges from a depth map or normal map using an edge detection mechanism, such as the Sobel filter.
- [S1, unofficial] Combine detected depth and normal edges to yield perceptually significant silhouettes.
- [S1, unofficial] For further processing (e.g., modifying line qualities), calculate analytical curves from the detected edge map, as detected edges only exist as lists of pixels.
#### comparison
- [S1, unofficial] Depth edges reflect C0 discontinuities, while normal edges reflect C1 discontinuities; combining both yields all the perceptually significant silhouettes in the source model.

### Ghosted Views with Contours
#### step
- [S2, unofficial] Pre-segment the most interesting internal structures (e.g., small internal organs).
- [S2, unofficial] Identify context information (e.g., the body) that obscures the interesting structures.
- [S2, unofficial] In areas of occlusion, reduce the visual representation of the context object to contours.

## Background
The concept of using outlines to enhance visual clarity in computer graphics has been surveyed extensively, with Hertzmann providing a useful overview of image space outline generation algorithms [Her99, cited in S1]. The foundational idea relies on the observation that depth and normal maps contain discontinuities at object boundaries that can be extracted using standard edge detection filters like Sobel [S1, unofficial]. In the context of volume rendering, these techniques have been adapted to handle volumetric data, with applications ranging from medical visualization (e.g., CT-angiography, ghosted views of internal organs) to scientific data exploration (e.g., hurricane moisture visualization) [S2, unofficial]. The field has evolved from simple pixel-based edge extraction to more sophisticated methods involving curvature control, interactive lasso selection, and topological landscape mapping [S1, S2, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
