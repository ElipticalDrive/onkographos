# Cutaways

Also called: Cut-away views

## Overview
Cutaways are an illustrative volume rendering technique where occluding objects are removed to reveal the structure of interest [S1, book]. They solve the problem of visual obstruction in complex volumetric scenes by selectively suppressing background elements. Unlike ghosting, which reduces opacity to preserve spatial context, cutaways completely remove the occluding regions [S1, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset containing three logical object sets: Selection (object of interest), Ghost (original position of selection), and Background (occluding environment) [S1, book].
*   **Output:** A rendered image where background voxels are suppressed based on an importance function.

**Data Structures:**
The method relies on a static importance function $I(\mathbf{x})$ defined per voxel $\mathbf{x}$. In the simplified VolumeShop setup, this function is pre-computed and static to avoid costly per-voxel importance compositing [S1, book].

**Steps:**
1.  **Define Importance Function:** Assign an importance value to each voxel. The function determines which object (Selection, Ghost, or Background) is subject to the cutaway [S1, book].
2.  **Apply Cutaway to Background:** For voxels belonging to the Background set, if the importance function indicates the voxel is occluding the Selection or Ghost, its opacity is set to zero (removed) [S1, book].
3.  **Independent Definition:** The cutaway criteria can be defined independently for the Ghost and Selection objects [S1, book].
4.  **Render:** Composite the remaining voxels using standard volume rendering techniques.

**Parameters:**
*   **Importance Function:** Static definition used to skip costly compositing [S1, book].
*   **Cutaway Degree:** The user can control the degree of cutaway, ranging from no cutaway (full background) to full cutaway (complete suppression of occluding structures) [S1, book].

**Edge Cases and Pitfalls:**
*   **Precision:** Static importance functions may lack the nuance of dynamic importance-driven rendering but offer efficiency [S1, book].
*   **Boundaries:** Care must be taken to ensure cutaways do not inadvertently remove parts of the Selection or Ghost [S1, book].

## Subtopics

### Feature-Based Cut-Aways
A technique to remove portions of Regions of Interest (ROIs) using axis-aligned, surface-aligned, or feature-based (segment-aligned) cuts [S2, unofficial]. It identifies obscuring features and applies cuts to remove them without impinging on ROI geometry [S2, unofficial]. Cut types include axis-aligned (global coordinates), surface-aligned (distances from isosurfaces), and feature-based (segment-aligned) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Efficient implementation due to static importance function, avoiding costly per-voxel importance compositing [S1, book].
*   **Memory Complexity:** Requires storage for the static importance function [S1, book].
*   **Strengths:** Efficient removal of occluding objects; clear revelation of structures of interest [S1, book].
*   **Limitations:** Complete removal of background may reduce spatial context compared to ghosting [S1, book].
*   **When to Use:** When the primary goal is to reveal internal structures obscured by external objects, and spatial context preservation is secondary [S1, book].
*   **When Not to Use:** When maintaining a strong sense of the object's position within the larger scene is critical [S1, book].

## Background
Cutaways are frequently found in information visualization and are an important tool employed by illustrators [S1, book]. Viola et al. introduced importance-driven volume rendering, a general framework for determining which object is to be cut using an importance function [S1, book]. VolumeShop's simplified three-object setup allows a static definition of this importance function, enabling efficient implementation [S1, book]. Correa et al. demonstrated the value of using feature data for generating cut-away views of 3-D volumetric data, including axis-aligned, surface-aligned, and feature-based cuts [S2, unofficial].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] Generalized Framework and Algorithms for Illustrative Visualization of Time-Varying Data on Unstructured Meshes, https://digital.library.unt.edu/ark:/67531/metadc838861/?q=illustrative volume rendering, fetched 2026-10-28 (unofficial)
