# Illustrative and non-photorealistic volume rendering

Also called: Volume illustration, NPR volume rendering

## Overview
Illustrative and non-photorealistic volume rendering (NPR) augments physics-based illumination pipelines with artistic techniques to enhance the perception of structure, shape, orientation, and depth in volumetric data [S2, S3, S4, unofficial]. It solves the problem of ambiguous internal structure in traditional physics-based or classification-based renderings by applying feature enhancement and depth cues derived from local volumetric properties rather than sample values alone [S3, unofficial]. Unlike surface-based NPR where features are well-defined boundaries, volume illustration determines feature areas through continuous analysis of local gradients and properties, requiring less manual tuning of transfer functions than traditional direct volume rendering [S3, unofficial].

## Shared principles
The family shares the goal of enhancing volume visualization through stylized lighting and feature abstraction. All methods operate on fragment-level data, typically utilizing the volume sample gradient as a proxy for surface normal orientation. They decouple illumination from the physical light transport equation, instead using mathematical functions of the view direction, light direction, and scalar field properties to modulate color [S3857]. Common inputs include the normalized gradient $\nabla f_n$, light vectors $L$, and base colors derived from transfer functions. The output is a final color $c$ that combines ambient/gaseous components with directed illumination effects [S3857].

The framework utilizes four types of properties for enhancement:
1. Volume sample location and value [S4, unofficial].
2. Local volumetric properties, such as gradient and minimal change direction [S4, unofficial].
3. View direction [S4, unofficial].
4. Light information [S4, unofficial].

Volumetric illustration differs from surface-based NPR in several important ways. In surface NPR, surfaces (features) are well-defined, whereas with volumes, feature areas within the volume must be determined through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space [S2, S4, unofficial].

## Subtopics

### [Geometric Buffers (G-buffers)](781-geometric-buffers-g-buffers.md)
Geometric Buffers (G-buffers) is a Non-Photorealistic Rendering (NPR) technique that decouples geometric analysis from shading by storing per-pixel geometric properties (e.g., depth, normals) in intermediate 2D images. It solves the problem of efficiently generating illustrative effects (such as line drawings and edge enhancement) for complex 3D volumes by leveraging fast 2D image processing algorithms instead of expensive 3D analytic computations. Unlike sibling methods that may compute shading or illumination directly in 3D space, G-buffers provide hidden surface elimination for free and allow NPR parameters to be tweaked interactively without re-traversing the volume.

### [Base Gaseous Illumination Renderer](794-base-gaseous-illumination-renderer.md)
The base renderer used for volume illustration examples, implementing physics-based illumination of gaseous phenomena. It can be replaced by a Phong illumination and linear accumulation renderer.

### [Orientation-Based Color Modification](798-orientation-based-color-modification.md)
Orientation-Based Color Modification simulates medical illustration techniques by altering the color, saturation, value, and transparency of volume samples based on the angle between the volumetric gradient and the viewer. It addresses the lack of depth cues in translucent volume rendering by using HSV color space to desaturate features facing the viewer and darken/saturate receding surfaces, thereby enhancing spatial understanding.

### [Feature Halos](799-feature-halos.md)
A technique to create halo effects in planes orthogonal to the view vector by making regions just outside features darker and more opaque, obscuring background elements to reinforce depth perception.

### [Context-Preserving Volume Rendering](801-context-preserving-volume-rendering.md)
Context-Preserving Volume Rendering (CPVR) is an illustrative volume rendering technique that selectively reduces opacity in less important data regions to visualize interior structures while preserving exterior context. It solves the problem of context loss inherent in conventional clipping techniques by modulating sample opacity based on shading intensity, gradient magnitude, distance to the eye point, and previously accumulated opacity. Unlike clipping planes which remove context information, CPVR retains it by making homogeneous areas (high lighting intensity) transparent, allowing users to inspect the interior of an object without hiding occluded objects.

### [Volume Illustration](810-volume-illustration.md)
Volume illustration is a non-photorealistic volume rendering method that augments a physics-based illumination pipeline with artistic techniques to enhance the perception of structure, shape, orientation, and depth in volumetric data. It solves the problem of ambiguous internal structure in traditional physics-based or classification-based renderings by applying feature enhancement and depth cues derived from local volumetric properties rather than sample values alone. Unlike surface-based NPR where features are well-defined boundaries, volume illustration determines feature areas through continuous analysis of local gradients and properties, requiring less manual tuning of transfer functions than traditional direct volume rendering.

### [Sketch Rendering via Silhouette Transparency](816-sketch-rendering-via-silhouette-transparency.md)
A technique to create a sketch-like appearance by making non-volumetric silhouettes (areas not near silhouette edges) transparent and using a specific sketch color (e.g., white or black), with shadows calculated based on original volume opacity.

### [Exact Projection Full Opacity](1882-exact-projection-full-opacity.md)
Exact Projection Full Opacity is a non-photorealistic volume rendering technique that simplifies the visual output by identifying a single primary front object and optionally one significant background object, blending them based on density and distance thresholds rather than performing full front-to-back accumulation. It differs from standard front-to-back compositing by skipping the accumulation of intermediate samples, storing only the first significant object (primary) and potentially one background object, then blending them directly to create a stylized, illustrative result.

### [Illumination and Tone Shading Models](3857-illumination-and-tone-shading-models.md)
Illumination and Tone Shading Models provide non-photorealistic lighting cues to enhance the visual clarity and artistic style of volume renderings. Unlike photometric models that simulate physical light transport, these methods map geometric or scalar properties to pre-defined color variations, prioritizing interpretability and aesthetic control over physical accuracy. They are distinct from [Geometric Buffers](781-geometric-buffers-g-buffers.md) by focusing on color modulation rather than depth/normal encoding, and from [Base Gaseous Illumination Renderer](794-base-gaseous-illumination-renderer.md) by incorporating directed shading components.

### [Interactive Volume Illustration (Hatching)](5272-interactive-volume-illustration-hatching.md)
An interactive object-space technique for illustrating volumetric data by simulating free-hand line art (hatching) using hardware-assisted volume and line rendering, with data-driven seed point selection to minimize memory and generation time.

### [Line and Edge Detection](5341-line-and-edge-detection.md)
Line and Edge Detection is a family of Non-Photorealistic Rendering (NPR) techniques designed to detect and render lines within volume data to communicate structure, shape, and boundaries. It solves the problem of visualizing complex geometry and internal structures in volumetric datasets where traditional photorealistic rendering may obscure topological details. What sets this family apart from its siblings is its specific focus on line-based abstraction rather than surface shading or contour outlines; it distinguishes itself from [Face-based volume rendering with edge highlights](2453-face-based-volume-rendering-with-edge-highlights.md) by generating lines that represent intrinsic surface properties (like curvature) rather than just silhouette edges, and from [Outline and Contour Generation](4137-outline-and-contour-generation.md) by often utilizing vector-oriented or texture-based approaches that are viewpoint-independent or analytically derived, rather than purely image-space depth-difference extraction.

### [Stipple Volume Rendering](6321-stipple-volume-rendering.md)
An interactive direct volume illustration system that simulates traditional stipple drawing by generating point lists based on volume characteristics (gradient magnitude/direction) and applying feature enhancements (silhouette curves, varying point sizes, resolution adjustments) to convey tone, texture, and shape.

### [Multi-object volume rendering with ghosting](6421-multi-object-volume-rendering-with-ghosting.md)
A method to visualize inter-penetrating volumetric objects by decomposing a scalar field into three distinct logical objects: selection, ghost, and background, allowing independent control of appearance and spatial transformation.

### Technical Illustration Effects
Efficient non-photorealistic rendering of 3D polygonal meshes in OpenGL for non-living subjects, using specific conventions different from scientific illustration [S1, unofficial]. Supported effects include line weighted depth cueing, fast silhouettes, shading using cool-to-warm lighting methods, light and highlight motion, and a non-parametric method of representing metal [S1, unofficial].

### Intent-Based Illustration Generation
A system that attempts to predict what kind of illustration a viewer would like to see by using an intent-based approach borrowing from AI knowledge bases [S1, unofficial]. Rules that describe communicative intent are defined, and stylistic decisions that reflect particular goals are used to generate the actual illustrations [S1, unofficial].

### Residual-based Visualization
A visualization technique to emphasize differences in information content between two rendered fields by storing voxel-wise color differences [S5, unofficial]. It calculates the $l_2$-norm of opacity-weighted color values for each voxel to store in the residual map, which is then rendered interactively from arbitrary perspectives [S5, unofficial].

### Procedural Pattern Coloring
Procedural coloring for butterfly wings using polar coordinates in wing space to generate veins, margins, bands, and apex patterns [S6, unofficial]. It converts wing coordinates to polar, calculates base field color, vein patterns using sine waves, and mixes colors for veins, margins, and bands [S6, unofficial].

### Residual-based Difference Visualization
A visualization mode to reveal structural differences between a reference volume (rendered with $T_r$) and an optimized volume (rendered with $T_o$) by rendering the residual field between them [S5, unofficial]. It applies transfer functions to both fields, computes the residual by subtracting per-voxel colors and computing the $l_2$-norm of the RGB$\alpha$ difference vector, and uses RGB colors pre-multiplied with $\alpha$ to ensure transparent regions have less contribution [S5, unofficial].

### Level Set Visualization
Visualizes level sets of large CT scans (e.g., $1024^3$) using slicemaps, which are small enough to fit in GPU memory and can be created by one slicing pass on the GPU [S7, unofficial]. It estimates normals to compensate for the lack of binary information and reflect surface appearance, keeping several level sets at the same time and blending them together for interactive display [S7, unofficial]. This avoids texture transfer stalls associated with slicing and the infeasibility of Marching Cubes at $1024^3$ resolution [S7, unofficial].

### Cutaways and Ghosting
Cutaways remove occluding objects to reveal structures of interest, using a simplified three-object setup (Background, Ghost, Selection) with static importance functions to skip costly importance compositing [S8]. Ghosting selectively reduces opacity of occluding regions to preserve edges and spatial location impression, controllable from no ghosting to full cutaway [S8]. The degree of ghosting is smoothly controllable from no ghosting (opacity not reduced) to full cutaway view (occluding structures completely suppressed) [S8].

## Background
The term "illustration" in the context of volume rendering was introduced by Ebert in 2000 [S3, unofficial]. The field is primarily motivated by the attempt to create renditions that consider the perceptual capabilities of humans [S3, unofficial]. Visualization primarily motivated by this attempt includes boundary enhancement based on gradient approximation [S3, unofficial] and curvature-based transfer functions [S3, unofficial].

The approach augments a physics-based rendering process with non-photorealistic rendering (NPR) techniques to enhance the expressiveness of the visualization [S2, S4, unofficial]. NPR draws inspiration from fields such as art and technical illustration to develop automatic methods to synthesize images with an illustrated look from geometric surface models [S2, unofficial]. While NPR research had effectively addressed surface shape and 2D data, it had virtually ignored the rendering of volume models until this work [S2, unofficial].

Early systems like SIV (Scientific Illustrator for Volumes) separated a 3D volume into segments using seeded region growing with thresholding, assigning interest labels to abstract groups of voxels, which were then rendered using NPR techniques [S1, unofficial]. SIV used Saito and Takahashi's G-buffer idea extensively [S1, unofficial].

In 2003, GPU raycasting was introduced by Krüger, making it feasible from a computational standpoint to apply illustrative visualization techniques in practice, overcoming previous performance constraints [S3, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] Non-Photorealistic Rendering of Volume Models, https://www.cs.umbc.edu/~rheingan/pubs/volillus00.pdf, fetched 2026-10-27 (unofficial)
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
[S5] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
[S6] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S7] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
[S8] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
