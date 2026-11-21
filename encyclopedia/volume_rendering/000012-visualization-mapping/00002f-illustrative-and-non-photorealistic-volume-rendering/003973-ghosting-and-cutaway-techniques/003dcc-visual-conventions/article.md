# Visual Conventions

Also called: Boxes, Arrows, Fans

## Overview
Visual Conventions are auxiliary geometric primitives overlaid on volume renderings to clarify the logical relationship between a **Selection** (the object of interest) and its **Ghost** (the original position) within the context of [Ghosting and Cutaway Techniques](8497-ghosting-and-cutaway-techniques.md). Unlike [Feature Halos](799-feature-halos.md) which highlight boundaries, or [Context-Preserving Volume Rendering](801-context-preserving-volume-rendering.md) which maintains spatial coherence through opacity, Visual Conventions explicitly depict translation, orientation, or alternative states using distinct shapes: Boxes for orientation, Arrows for movement, and Fans for alternative depictions.

## How it works
The method operates on the three-object logical model defined in the parent family: **Selection**, **Ghost**, and **Background** [S1]. Visual Conventions are rendered as static geometric overlays, avoiding costly per-voxel importance compositing [S1].

### Inputs and Outputs
- **Inputs**: 3D bounding boxes or centroids of the Selection and Ghost objects; camera view matrix for screen-space projection.
- **Outputs**: Rasterized geometric primitives (boxes, arrows, fans) composited over the volume rendering.

### Data Structures
No complex volumetric data structures are required for the conventions themselves. The method relies on:
- `Selection_BBox`: Axis-aligned or oriented bounding box of the selected object in world space.
- `Ghost_BBox`: Axis-aligned or oriented bounding box of the original object in world space.
- `Camera`: View-projection matrix to transform world coordinates to screen coordinates.

### Steps
1. **Project Bounding Boxes**: Transform the vertices of `Selection_BBox` and `Ghost_BBox` from world space to screen space using the current camera view-projection matrix.
2. **Compute Screen-Space Bounds**: Determine the 2D bounding rectangles for both the Selection and Ghost in screen space.
3. **Render Conventions**:
   - **Boxes**: Render the 3D `Selection_BBox` as wireframe or semi-transparent solid geometry. [S1]
   - **Arrows**: Construct a curved path from the Ghost position to the Selection position. [S1]
   - **Fans**: Connect the screen-space bounding rectangle of the Ghost to that of the Selection. [S1]
4. **Compositing**: Render these primitives with appropriate depth testing and blending to overlay the volume rendering.

### Formulas and Parameters

#### Arrow Curvature Control
To prevent degenerate short arrows when the Selection and Ghost project to nearby screen positions, the curvature of the arrow is controlled by the screen-space depth difference [S1].

Let $d_{screen}$ be the difference in depth (z-buffer value) between the projected centers of the Selection and Ghost in screen space.
The curvature $C$ of the arrow is a function of $d_{screen}$:
$$ C = f(d_{screen}) $$
Where a larger depth difference results in a straighter arrow, and a smaller difference increases curvature, creating the "bent" appearance typical in illustrations [S1].

#### Fan Construction
A Fan is constructed by connecting the screen-space bounding rectangles of the Ghost and Selection [S1].
Let $R_G$ be the screen-space bounding rectangle of the Ghost.
Let $R_S$ be the screen-space bounding rectangle of the Selection.
The Fan geometry consists of the union of $R_G$ and $R_S$ and the connecting edges between their corresponding corners.

### Edge Cases and Pitfalls
- **Short Arrows**: If the Selection and Ghost are close in screen space, a straight arrow may be visually insignificant or misleading. The depth-difference curvature control mitigates this [S1].
- **Occlusion**: Boxes can occlude important details of the volume data. They are recommended for design/interaction phases but may be distracting during final presentation [S1].
- **Ambiguity**: Fans indicate alternative depictions or detailed views. The user must ensure the visual distinction between the "detailed" and "alternative" states is clear to avoid confusion [S1].

## Subtopics

### [Marker Placement](16841-marker-placement.md)
Marker Placement is an illustrative volume rendering technique that overlays cone-shaped geometric primitives into the 3D data space to aid feature location and facilitate discussion during parameter adjustments. It distinguishes itself from sibling methods like [Boxes](9605-visual-conventions.md) by using directional, non-occluding shapes that can be arbitrarily oriented to indicate specific vectors or points of interest, rather than providing bounding context.

## Operating characteristics
- **Performance**: Rendering is efficient as it uses static geometric primitives rather than per-voxel computation. The cost is dominated by the rasterization of the overlay geometry, which is negligible compared to volume raycasting.
- **Strengths**: Provides clear semantic cues about object movement (Arrows), orientation (Boxes), and alternative states (Fans). Enhances interpretability in complex scenes with multiple objects.
- **Limitations**: Boxes can clutter the view and occlude data. Arrows and Fans rely on screen-space projection, which may distort if the camera angle changes drastically without updating the conventions.
- **When to use**: Ideal for interactive design tools, educational visualizations, or presentations where explaining the relationship between object states is critical.
- **When not to use**: In final high-fidelity presentations where visual clutter must be minimized, or in real-time applications with strict frame-rate budgets where even simple geometry adds overhead.

## Background
Visual Conventions were introduced in the VolumeShop system, an interactive environment for direct volume illustration [S1]. The approach draws inspiration from traditional illustrators' techniques to clarify the role of objects in complex scenes. The system allows users to manipulate objects and visualize their movement or alternative states using these conventions, enhancing the communicative power of volume rendering [S1].

## Sources
[S1] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
