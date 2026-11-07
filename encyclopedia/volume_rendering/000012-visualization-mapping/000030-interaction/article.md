# Interaction
## Overview
Interaction in volume rendering refers to the mechanisms by which users manipulate visualization parameters, data subsets, or view geometry to guide the rendering process and explore the dataset [G]. It solves the problem of static, pre-computed images by allowing real-time exploration of complex volumetric data [G]. This family is distinct from [Classification and transfer functions](38-classification-and-transfer-functions.md) (which map data to visual properties) and [Illustrative and non-photorealistic volume rendering](40-illustrative-and-non-photorealistic-volume-rendering.md) (which stylize the output); instead, it focuses on the control loop between the user and the renderer [G]. It differs from [Multi-field, vector and tensor volume visualization](39-multi-field-vector-and-tensor-volume-visualization.md) by addressing the general interaction paradigm rather than specific data types [G].

## Shared principles
Interaction operates via a feedback loop where user input modifies parameters that influence the rendering pipeline [G]. These parameters typically fall into three categories:
1. **Geometric/View Parameters:** Camera position, orientation, and clipping planes (e.g., cropping) [G].
2. **Data/Selection Parameters:** Subsets of the volume to render or modify (e.g., selection, brushing) [G].
3. **Visual/Mapping Parameters:** Transfer function values, deformation fields, or LOD levels [G].

Mathematically, interaction is often modeled as a function $I: \mathcal{U} \rightarrow \mathcal{P}$, where $\mathcal{U}$ is the user input space and $\mathcal{P}$ is the parameter space of the renderer [G]. The renderer then computes the image $V = R(D, P)$, where $D$ is the data and $P$ is the current parameter set [G]. A key challenge in implementation is maintaining consistency across multiple coordinated views (e.g., linking selection in a 2D overview to the 3D volume), which requires a shared state management system [G].

## Subtopics

### [Cropping](1648-cropping.md)
Cropping restricts volume rendering to a specific region of interest by cutting off the volume along planes parallel to its faces. It solves visual clutter and performance overhead by allowing users to see inside a volume without altering data. Unlike [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md), which uses arbitrary-angle planes, cropping is strictly axis-aligned relative to the volume's object coordinates. It differs from [Clipping](1444-clipping.md) by defining a bounded region of visibility via slabs rather than a half-space relative to the camera.

### [3D Cursor](1655-3d-cursor.md)
The 3D Cursor is a direct interaction method that specifies X, Y, and Z coordinates to select or explore features within a volume dataset. It serves as a spatial probe, allowing users to identify internal structures without altering the geometry, distinct from clipping or deformation methods. Unlike 2D cursors, it operates in three dimensions, often visualized as intersecting planes or lines, and can be used for both local selection and action-at-a-distance targeting.

### [Focus+Context Visualization via Deformation](3001-focus-context-visualization-via-deformation.md)
Focus+Context Visualization via Deformation is an interactive technique that magnifies regions of interest within a high-resolution volume dataset while shrinking less important regions to fit within the original volume boundary. It solves the problem of inspecting detailed features on low-resolution displays by leveraging color gradient magnitudes derived from the user-specified transfer function to determine importance. Unlike binary focus/context methods or explicit lens-based techniques, this method provides continuous, varying extents of magnification for entire features without requiring explicit feature specification.

### [Frustum Culling for Hierarchical Navigation](5370-frustum-culling-for-hierarchical-navigation.md)
Frustum Culling for Hierarchical Navigation is an interaction technique that reduces visual clutter and aids data exploration by selectively rendering nodes based on their position relative to the 3D viewing frustum. It sets itself apart from standard geometric culling by simultaneously managing the visual state of coordinated 2D views (overview maps and treemaps), ensuring that culling in 3D triggers specific contextual updates in 2D rather than simple invisibility.

### [Dynamic Focus+Context Framework](5844-dynamic-focus-context-framework.md)
A framework for dynamic focus+context visualization in volume rendering that uses user interaction to guide enhancements of the visualization via animation modules. It sets itself apart from sibling methods by dynamically adapting the view and rendering style based on estimated user engagement rather than relying on pre-defined regions or static deformation fields.

### [Depth-based Clipping for Focus and Context Volume Rendering](7485-depth-based-clipping-for-focus-and-context-volume-rendering.md)
A technique to decompose a volume into a high-quality 'focus' region and a lower-quality 'context' region using depth-based clipping and early-z rejection, implemented in four rendering passes to maximize fragment shader discard efficiency.

### [Selection and Manipulation](7919-selection-and-manipulation.md)
This family covers techniques for identifying specific subsets of volumetric data or rendering primitives for modification, deletion, or annotation, and for manipulating the spatial relationship between the selection and the scene. It solves the problem of precise user intent translation in 3D space, where traditional 2D screen-space interactions must be mapped to complex 3D structures. It is distinguished from [Cropping](1648-cropping.md) by its focus on discrete object or primitive-level targeting rather than global volume partitioning, and from [3D Cursor](1655-3d-cursor.md) by its broader scope including occlusion-based and state-based selection modes.

### [Hybrid Rendering for Stress Tensor Fields](8909-hybrid-rendering-for-stress-tensor-fields.md)
A visualization approach combining volume rendering with glyphs and tensorlines to separate focus and context regions in stress tensor datasets. Context is visualized by volume rendering of scalar invariants, while focus regions (selected via user interaction) display explicit geometrical tensor representations.

### User Interaction-Based Illustration
#### definition
- [S1, unofficial] A system that monitors user interaction with virtual text to infer viewer intent and generate supporting graphics dynamically.
#### step
- [S1, unofficial] Monitor user interaction with virtual text to infer intent.
- [S1, unofficial] Generate supporting graphics dynamically based on user interaction according to a set of heuristics.
- [S1, unofficial] Render surface models and display with OpenGL (extendable to voxel models).

### Brushing and Linking Interface
#### definition
- [S2] A technique allowing users to select a subset of nodes from the current LOD for operations like join or split, with dynamic linking across three views (rendering window, overview map, treemap).
#### step
- [S2] Brushing in the overview map allows joining or splitting all nodes at particular levels in the tree hierarchy.
- [S2] Brushing in the treemap allows changing resolutions of blocks in a local vicinity according to error values. Users click a node or specify a rectangular region to select multiple nodes simultaneously.
- [S2] Brushing in the rendering window allows direct interaction with the data set. Users identify 3D blocks of interest and link selection forward to other views.
- [S2] Provide sliders in the rendering window to specify brush coverage in data space as a 1D point, a 2D plane, or a 3D box.
- [S2] Allow brushing by specifying coverage in color space using two sliders in range [0, 1] to set minimum and maximum color values. Select all nodes in the current LOD within this color (and error) range.
- [S2] Use color space brushing to select nodes from a certain error range in a fuzzy manner, allowing resolution changes without degrading image quality too much.

### Hybrid Spatial Rendering
#### definition
- [S3, book] A rendering approach combining volume rendering with glyphs and tensorlines, steered by a mask volume created from diagram view selections.
#### step
- [S3, book] Create and update a mask volume through user selections (brushing) in the diagram views.
- [S3, book] Evaluate the mask volume to map selected features to geometrical tensor representations (glyphs, tensorlines) for focus regions.
- [S3, book] Visualize context regions using volume rendering.

## Background
The concept of interaction in volume rendering evolved from early direct volume visualization (DVV) systems in the late 1980s and early 1990s, which primarily focused on rendering algorithms like ray casting and texture mapping [G]. Early interaction was limited to camera navigation and simple transfer function adjustment [G]. As datasets grew larger and more complex, the need for advanced interaction techniques such as cropping, focus+context, and hierarchical navigation became apparent [G]. The integration of interaction with non-photorealistic illustration (NPR) and multi-view coordinated interfaces (like brushing and linking) has been a significant area of research, particularly in medical and scientific visualization [G]. Key developments include the introduction of direct selection methods for splat-based rendering and the use of deformation fields for focus+context visualization [G].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Hierarchical Navigation Interface: Leveraging Multiple Coordinated ..., https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
[S3] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
