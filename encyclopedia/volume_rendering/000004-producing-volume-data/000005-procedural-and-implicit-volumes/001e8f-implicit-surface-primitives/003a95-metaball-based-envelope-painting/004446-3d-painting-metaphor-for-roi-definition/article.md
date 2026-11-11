# 3D Painting Metaphor for ROI Definition

Also called: Brush stroke interaction, Envelope-based ROI, Painting metaphor

## Overview
This method allows users to define a Region of Interest (ROI) by placing brush strokes directly in the visualization space, creating an implicit envelope that encloses features of interest without requiring complex parameter tweaking [S1, unofficial]. It differs from traditional transfer function specification by replacing 2D histogram curve manipulation with direct spatial marking, eliminating the need for volumetric data representation knowledge [S1, unofficial]. It builds on [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md) by providing the interaction model that generates the metaball parameters.

## How it works
**Inputs and Outputs:**
*   **Input:** User mouse/pen input (position, click/drag events); current camera/view state; target volumetric data bounding box.
*   **Output:** A set of metaball primitives (centers, radii, weights) defining the ROI envelope.

**Data Structures:**
The envelope is represented as a collection of metaball primitives. Each primitive $P_i$ is defined by:
*   Center position $\mathbf{c}_i$ (3D vector).
*   Radius $r_i$ (scalar).
*   Weight $w_i$ (scalar, typically 1.0 for uniform influence).

**Steps:**
1.  **Mode Selection:** The user selects either 2D or 3D painting mode [S1, unofficial].
2.  **Stroke Initiation:** The user places a brush stroke in the visualization area [S1, unofficial].
3.  **Coordinate Transformation:**
    *   If **2D mode** is active: The stroke position is interpreted as screen-space coordinates $(u, v)$. The system projects a ray from the camera through this point into the 3D scene. The metaball center $\mathbf{c}_i$ is placed at the intersection with the object surface or a fixed depth plane [S1, unofficial]. The brush tip is circular in screen space [S1, unofficial].
    *   If **3D mode** is active: The stroke position is interpreted as 3D world/volumetric coordinates. The system uses ray casting or depth buffer information to place the metaball center $\mathbf{c}_i$ at the specific 3D location within the volume [S1, unofficial]. The brush tip is spherical in 3D space [S1, unofficial].
4.  **Envelope Expansion:** The user places additional strokes. Each new stroke adds a new metaball primitive to the collection. The metaballs merge based on proximity and radius, creating a continuous organic envelope [S1, unofficial].
5.  **Rendering/Usage:** The resulting implicit field is evaluated during volume rendering to mask or highlight the ROI [S1, unofficial].

**Parameters:**
*   **Brush Tip Geometry:** Circular for 2D mode, Spherical for 3D mode [S1, unofficial].
*   **Painting Coordinate Space:** Screen space (2D) or World/Volumetric space (3D) [S1, unofficial].
*   **Metaball Threshold:** Not given in the sources.

**Pitfalls:**
*   **2D Envelope Misalignment:** 2D envelopes act like X-ray projections, potentially obscuring or misaligning with internal 3D structures when the object rotates [S1, unofficial]. This requires explicit mode switching for accurate internal feature selection [S1, unofficial].

## Subtopics

### 2D Painting Mode
*   **What it is:** Painting in screen-space coordinates where the envelope remains fixed to the screen regardless of volumetric object orientation [S1, unofficial].
*   **Differentiation:** Acts like an X-ray projection; fast for global surface features but lacks depth specificity [S1, unofficial].
*   **When to choose:** When selecting features that are aligned with the current view plane or for quick global masking [S1, unofficial].
*   **Link:** [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md)

### 3D Painting Mode
*   **What it is:** Painting in 3D volumetric space where the envelope maintains its relative position and orientation to the object as the camera moves [S1, unofficial].
*   **Differentiation:** Provides true spatial selection within the volume, independent of camera angle [S1, unofficial].
*   **When to choose:** When selecting internal structures or features that require precise 3D localization [S1, unofficial].
*   **Link:** [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Linear with respect to the number of brush strokes (metaballs) [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance depends on the number of metaballs and the efficiency of the implicit field evaluation during rendering [S1, unofficial].
*   **Strengths:** Intuitive, eliminates need for transfer function expertise, provides organic shapes [S1, unofficial].
*   **Limitations:** 2D mode suffers from depth ambiguity; 3D mode requires accurate depth estimation [S1, unofficial].
*   **When to use:** For intuitive volume exploration and ROI definition [S1, unofficial].
*   **When not:** When precise parametric control over the ROI shape is required beyond organic merging [S1, unofficial].

## Background
This method was proposed and implemented as part of the ImageVis3D extensions described in the thesis "Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor" [S1, unofficial]. It aims to overcome domain knowledge and interface complexity issues in volume visualization by providing an integrated 3D single-view interaction model with minimum controls [S1, unofficial]. It was chosen to allow users to concentrate on gaining insight into the data rather than tweaking visualization controls [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
