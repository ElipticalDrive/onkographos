# Outline and Contour Generation
Also called: Outline Representation
## Overview
Outline and Contour Generation is a family of non-photorealistic volume rendering techniques that enhance perceptual clarity by explicitly rendering edges, silhouettes, or topological boundaries as line primitives or stylized strokes [G]. Unlike standard volume rendering which relies on opacity and color to imply form, these methods detect geometric or scalar discontinuities and render them as distinct visual cues to define object boundaries, internal structures, or topological relationships [G]. This family is distinguished from its siblings by its focus on explicit edge extraction and rendering strategies, ranging from object-space silhouette extraction to image-space depth discontinuity detection and topological landscape mapping [G].
## Shared principles
The family shares the fundamental goal of enhancing perceptual clarity through non-photorealistic edge representation [G]. Common mechanisms include identifying boundaries based on local volumetric properties or image-space properties, and converting detected edges into line primitives or stylized strokes [G].
## Subtopics

### [Object Contour NPR](787-object-contour-npr.md)
Generates view-dependent line primitives demarcating areas where an object's surface turns away from the viewer, relying on gradient magnitude and viewing angle for real-time tracing. Choose when direct screen rendering of contours is needed over texture-buffer recording.

### [Image Space Outline Generation](788-image-space-outline-generation.md)
A post-processing technique detecting screen-space depth or normal discontinuities to define boundaries without analytical geometry processing. Choose for faster implementation and lower computational overhead.

### [Lasso Contour Rendering](1855-lasso-contour-rendering.md)
An interactive technique allowing users to manually define regions of interest to generate contour overlays, isolating complex structures lacking sharp boundaries. Choose when user-driven contour generation is required.

### [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md)
Regulates contour thickness based on normal curvature along the view direction to prevent uncontrolled variation in thickness inherent in standard dot-product silhouette methods. Choose when consistent visual weight across varying surface geometries is needed.

### [Silhouette Rendering](7959-silhouette-rendering.md)
An object-space technique extracting significant edges (silhouettes and feature lines) from geometry, assembling them into strokes, and rendering them with stylization to enhance perceptual clarity. Choose when operating on pre-extracted geometric primitives for independent stylization.

### [Topological Landscape](8481-topological-landscape.md)
Maps the connectivity of a scalar field’s level sets onto a 3D terrain model using the contour tree, leveraging human intuition for reading physical terrain to interpret complex topological relationships. Choose when interpreting complex topological relationships (merges/splits) is the goal.

### [Suggestive Contour Extraction](13101-suggestive-contour-extraction.md)
Identifies surface locations where radial curvature is zero but increasing towards the viewer, complementing standard object contours to reveal hidden or occluded geometry. Choose when revealing hidden geometry is necessary.

### [Bounding Wall Contour Overlay](16354-bounding-wall-contour-overlay.md)
Displays line contours of a reference scalar field on the vertical bounding walls of a 3D volume to provide spatial context without occluding primary data. Choose when spatial context for deformed interfaces is needed without visual clutter.

### [Contour Correspondence via Object Set Overlap](17217-contour-correspondence-via-object-set-overlap.md)
Determines contour correspondence in time-varying fields by verifying overlap of associated upper and lower object sets, avoiding higher-dimensional interpolations. Choose when tracking contour evolution (merging, splitting, moving) in time-varying scalar fields is required.

### [Tube Outline Effect](17620-tube-outline-effect.md)
Constructs a tubular contour around volumetric features to clarify borders and assist in feature identification relative to the view direction. Choose when explicit 3-D tube geometry from projected silhouettes is needed.

### [Outline Drawing](22208-outline-drawing.md)
Renders a dataset by displaying only the geometric boundaries of its extent, typically as a wireframe or bounding box, rather than the volumetric data itself. Choose for lightweight visualization primitives for rapid spatial orientation.

### [Probabilistic Silhouette Search](22281-probabilistic-silhouette-search.md)
A spatial coherence-based heuristic for detecting and tracing silhouette edges on polygonal meshes. Choose when performance bottleneck of brute-force silhouette detection needs to be solved.

### [Hidden Line Removal](22782-hidden-line-removal.md)
A rendering technique used in wireframe representations to hide line segments that are occluded by solid objects in the scene. Choose when operating on explicit geometric primitives to hide occluded lines.

### Ghosted Views with Contours
Pre-segments the most interesting internal structures and identifies context information that obscures them. In areas of occlusion, the visual representation of the context object is reduced to contours to provide a clear view of the interesting internal organ [S2, unofficial]. Choose when clarifying internal structures obscured by context is needed.

### 2D Stroke to 3D Line Picking
Locates the intended location of a 2D stroke in 3D space by finding the 3D line that is a projection of the 2D stroke and fits contours in the data best, determined by the dot product of the normal vector of the extruded stroke and the gradient in the data [S3]. This method only works at contours of objects and segments objects whose contour best fits the stroke, which may not be the most visible objects, whereas WYSIWYP is applicable to any position in the rendering [S3]. Choose when explicitly selecting and highlighting regions of interest via 2D strokes is required.

## Background
The family of outline and contour generation techniques has evolved to address the limitations of photorealistic volume rendering in conveying shape, boundaries, and topological information [G]. Early work in illustrative visualization for medicine and science highlighted the utility of expressive visualizations, such as ghosted views with contours, to clarify internal structures obscured by context [S2, unofficial]. The development of non-photorealistic rendering (NPR) techniques allowed for the explicit rendering of edges and silhouettes to enhance perceptual clarity [G]. Interactive methods like lasso contour rendering and 2D stroke to 3D line picking were developed to allow users to explicitly select and highlight regions of interest [G][S3].
## Sources
[S1] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S3] WYSIWYP: What You See Is What You Pick, https://doi.org/10.1109/tvcg.2012.292, fetched 2026-11-05
