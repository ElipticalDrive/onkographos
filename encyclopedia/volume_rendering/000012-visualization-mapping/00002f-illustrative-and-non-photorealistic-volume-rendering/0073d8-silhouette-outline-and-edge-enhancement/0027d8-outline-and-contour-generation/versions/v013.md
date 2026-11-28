# Outline and Contour Generation


## Overview
Outline and Contour Generation is a family of non-photorealistic volume rendering techniques that enhance perceptual clarity by extracting and stylizing object boundaries, silhouettes, or topological features [S1, unofficial][S2]. It solves the problem of ambiguous shape perception in volumetric data by providing explicit edge cues that distinguish object classes and indicate spatial relationships [S1, unofficial][S2]. Unlike standard photorealistic rendering which relies on shading and opacity, this family operates in object space or screen space to generate line primitives that are independent of the volume's internal intensity values, allowing for faster parameter tuning and distinct stylistic control [S2, unofficial].

## Shared principles
The family shares the fundamental goal of enhancing perceptual clarity through non-photorealistic edge representation [G]. Common mechanisms include:
1. **Edge Detection:** Identifying boundaries based on local volumetric properties (gradients, Laplacian of Gaussian) or image-space properties (depth, normal discontinuities) [S2][S3, S4, unofficial].
2. **Line Rendering:** Converting detected edges into line primitives (strips, segments) or stylized strokes [S2][S3, S4, unofficial].

## Subtopics

### [Object Contour NPR](787-object-contour-npr.md)
Object Contour NPR generates view-dependent line primitives that demarcate areas where an object's surface turns away from the viewer, conveying shape through boundary emphasis. Unlike hatches, which are often pre-rendered to texture buffers, these contours are traced and rendered directly to the screen in real-time. It differs from direct volume rendering by relying solely on gradient magnitude and viewing angle rather than voxel intensity values, allowing for faster parameter tuning. See [Object Contour NPR](787-object-contour-npr.md).

### [Image Space Outline Generation](788-image-space-outline-generation.md)
Image Space Outline Generation is a post-processing technique that renders non-photorealistic outlines by detecting discontinuities in screen-space depth or normal buffers. It solves the problem of defining object boundaries in volume rendering without the computational overhead of analytical geometry processing. Unlike sibling methods that compute contours from volumetric gradients or topological features, this method relies on simple pixel-wise comparisons of existing rasterization buffers, making it significantly faster and easier to implement. See [Image Space Outline Generation](788-image-space-outline-generation.md).

### [Lasso Contour Rendering](1855-lasso-contour-rendering.md)
Lasso Contour Rendering is an interactive non-photorealistic volume rendering technique that allows users to manually define and visualize specific volumetric features using a free-form "lasso" selection tool. It solves the problem of isolating complex, continuous volumetric structures that lack sharp boundaries by enabling user-driven contour generation. It differs from its siblings, such as [Object Contour NPR](787-object-contour-npr.md) and [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md), by relying on explicit user interaction to define the region of interest rather than automatic geometric or scalar-based detection. See [Lasso Contour Rendering](1855-lasso-contour-rendering.md).

### [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md)
Curvature-Controlled Style Contours is a volume rendering technique that regulates contour thickness based on the normal curvature along the view direction to prevent the uncontrolled variation in thickness inherent in standard dot-product silhouette methods. While standard methods ($v \cdot n \approx 0$) produce contours that are too thick on flat surfaces and too thin on fine structures, this method uses curvature to maintain consistent visual weight across varying surface geometries. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md) by integrating curvature-aware edge detection with style transfer functions. See [Curvature-Controlled Style Contours](5350-curvature-controlled-style-contours.md).

### [Silhouette Rendering](7959-silhouette-rendering.md)
Silhouette rendering is a non-photorealistic technique that extracts and stylizes object boundaries to enhance perceptual clarity, distinguish object classes, and indicate contextual relationships. It sets itself apart from image-space alternatives by operating in object space, allowing for precise geometric edge extraction and independent stroke stylization before compositing. Unlike [Image Space Outline Generation](788-image-space-outline-generation.md), which relies on post-processing rendered images, this method constructs explicit line primitives from mesh data. See [Silhouette Rendering](7959-silhouette-rendering.md).

### [Topological Landscape](8481-topological-landscape.md)
Topological Landscape is a visualization technique that maps the connectivity of a scalar field’s level sets onto a 3D terrain model using the contour tree. It solves the problem of interpreting complex topological relationships (merges and splits of contours) by leveraging human intuition for reading physical terrain. It sets itself apart from standard contour rendering by transforming abstract graph nodes into a spatial height map where peaks and valleys explicitly encode topological events, differing from [Contour Tree](9312-contour-tree.md) by providing a geometric metaphor rather than just a hierarchical graph structure. See [Topological Landscape](8481-topological-landscape.md).

### [Boundary Enhancement Volume Style](10863-boundary-enhancement-volume-style.md)
Boundary Enhancement Volume Style is a non-photorealistic rendering technique designed to highlight feature boundaries within volumetric data while simultaneously reducing visual noise. It operates by detecting local intensity gradients or structural discontinuities to generate edge-like highlights, distinct from standard photorealistic shading which focuses on surface illumination. Unlike sibling methods such as [Object Contour NPR](787-object-contour-npr.md) which may rely on geometric mesh extraction, this method operates directly in the volume space to preserve internal structure visibility while emphasizing edges. See [Boundary Enhancement Volume Style](10863-boundary-enhancement-volume-style.md).

### [Suggestive Contour Extraction](13101-suggestive-contour-extraction.md)
Suggestive contour extraction identifies surface locations where radial curvature is zero but increasing towards the viewer, complementing standard object contours to enhance shape perception in non-photorealistic volume rendering. Unlike standard contours which mark visible boundaries, suggestive contours reveal hidden or occluded geometry, such as the rims of indentations, by detecting where the surface is about to curve towards the camera. See [Suggestive Contour Extraction](13101-suggestive-contour-extraction.md).

### Ghosted Views with Contours
Ghosted Views with Contours is a technique that reduces the visual representation of context objects to contours in areas of occlusion to provide a clear view of interesting internal structures [S1, unofficial]. The process involves pre-segmenting the most interesting internal structures (e.g., small internal organs) and identifying context information (e.g., the body) that obscures them [S1, unofficial]. In areas where the context object occludes the internal structure, the visual representation of the context is reduced to contours [S1, unofficial]. This allows for an unobstructed view of the segmented features from each viewpoint [S1, unofficial].

## Background
The concept of using illustrative techniques for volume visualization has been applied in various domains, including medicine and science [S1, unofficial]. Early applications included cut-away visualizations for CT-angiography to visualize peripheral arteries, where bones and skin contours were visualized to understand spatial arrangement [S1, unofficial]. The technique of ghosted views with contours was demonstrated using a scalar volumetric dataset of a Leopard gecko, where the small internal structure (yellow) was pre-segmented and the body was reduced to contours in areas of occlusion to clearly view the internal organ [S1, unofficial].

## Sources
[S1] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
