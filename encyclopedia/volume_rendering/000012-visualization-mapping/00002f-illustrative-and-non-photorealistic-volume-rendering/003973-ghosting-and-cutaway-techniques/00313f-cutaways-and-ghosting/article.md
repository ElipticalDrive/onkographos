# Cutaways and Ghosting
Also called: Cut-away views, Ghosting, Selective Illustration

## Overview
Cutaways and Ghosting are illustrative volume rendering techniques that selectively suppress or reduce the opacity of occluding background structures to reveal a primary object of interest (Selection) [S1][S2, book]. The method employs a three-object logical model (Selection, Ghost, Background) to create "magic views" that aid mental reconstruction of spatial relationships [S1][S2, book]. It sets itself apart from siblings like [Feature Halos](799-feature-halos.md) by explicitly managing occlusion through opacity reduction or removal rather than edge enhancement, and from [Context-Preserving Volume Rendering](801-context-preserving-volume-rendering.md) by allowing full suppression (cutaway) rather than just context preservation [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset; a user-defined transformation (translation, rotation) for the Selection; a user-defined "Ghosting Degree" parameter $\alpha \in [0, 1]$; and a lighting transfer function that enhances opacity at contours [S1].
*   **Output:** A 2D image where the Selection is fully visible, the Ghost is visible in its displaced position, and the Background is occlusion-culled or ghosted [S1][S2, book].

**Data Structures:**
The volume is conceptually decomposed into three sets:
1.  **Selection ($S$):** The primary object of interest [S1][S2, book].
2.  **Ghost ($G$):** The original position or alternative state of the Selection [S1][S2, book].
3.  **Background ($B$):** All other data not part of $S$ or $G$ [S1][S2, book].

**Rendering Strategy:**
To avoid costly per-voxel importance compositing, the method uses static importance functions defined by the three-object setup [S1].

**Steps:**
1.  **Define Selection and Ghost:** Identify the voxels belonging to the Selection $S$. Define the Ghost $G$ as the spatially transformed version of $S$ (e.g., translated) [S1][S2, book].
2.  **Define Background:** Identify voxels belonging to the Background $B$ (all voxels not in $S$ or $G$) [S1].
3.  **Calculate Ghosting Factor:** Determine the opacity reduction for Background voxels based on the user-defined Ghosting Degree $\alpha$ [S1][S2, book].
4.  **Apply Occlusion Culling/Reduction:**
    *   For **Cutaways** ($\alpha = 1$): Completely suppress Background voxels that occlude the Selection or Ghost [S1][S2, book].
    *   For **Ghosting** ($0 < \alpha < 1$): Reduce the opacity of Background voxels that occlude the Selection or Ghost. The reduction is combined with the opacity-enhancement component of the lighting transfer function to preserve edge cues [S1].
    *   For **No Ghosting** ($\alpha = 0$): No reduction is applied [S1][S2, book].
5.  **Render:** Composite the Selection, Ghost, and modified Background [S1][S2, book].

**Formulas:**
Not given in the sources. The effective opacity of a Background voxel is calculated by combining the user-defined ghosting factor with the opacity-enhancement component of the lighting transfer function [S1]. Regions with high contour enhancement (edges) are less suppressed during ghosting [S1].

**Parameters:**
*   **Ghosting Degree ($\alpha$):** Smoothly controllable from 0 (no ghosting) to 1 (full cutaway) [S1][S2, book].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### Fan Visualization
A visual convention used to indicate a more detailed or alternative depiction of a structure [S2, book]. It consists of a connected pair of shapes (e.g., rectangles or circles) linking the screen-space bounding rectangles of the Ghost and Selection [S2, book]. It is constructed by connecting these bounding boxes in screen space [S2, book].

## Operating characteristics
*   **Time Complexity:** Efficient implementation avoids per-voxel importance compositing by using static importance functions, allowing for faster rendering [S1].
*   **Memory Complexity:** Requires storage for the three-object decomposition masks [G].
*   **Strengths:** Provides clear spatial context and aids mental reconstruction of structures by preserving edge cues (Ghosting) or removing occlusions (Cutaways) [S1][S2, book]. Allows for expressive "magic views" [S2, book].
*   **Limitations:** Relies on user-defined transformations and may not automatically adapt to complex scene layouts without manual intervention [S2, book].
*   **When to use:** When illustrating specific anatomical or mechanical structures where occlusion by surrounding tissue/material hinders understanding [S1][S2, book].

## Background
Originated in the context of interactive volume illustration systems, specifically VolumeShop [S1]. The method was developed to bridge the gap between scientific visualization and traditional illustration techniques, allowing users to create expressive views that aid in the interpretation of complex 3D data [S1][S2, book]. It is part of the broader family of Illustrative Focus+Context approaches [S2, book].

## Sources
[S1] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
[S2] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
