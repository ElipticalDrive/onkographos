# 3D Painting Metaphor for ROI Definition

Also called: Brush stroke interaction, Envelope-based ROI, Painting metaphor

## Overview
This method allows users to define a Region of Interest (ROI) by placing brush strokes directly in the visualization space, creating an envelope that encloses features of interest without requiring complex parameter tweaking [S1, unofficial]. It differs from traditional transfer function specification by replacing 2D histogram curve manipulation with direct spatial marking, eliminating the need for volumetric data representation knowledge [S1, unofficial]. It builds on [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md) by providing the interaction model that generates the metaball parameters [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** User mouse/pen input (stroke placement events) [S1, unofficial].
*   **Output:** An ROI envelope defined by a collection of metaball primitives [S1, unofficial].

**Data Structures:**
The envelope is represented as a collection of metaball primitives [S1, unofficial]. Exact field layouts, sizes, and bit packing are not given in the sources.

**Steps:**
1.  **Mode Selection:** The user selects either 2D or 3D painting mode [S1, unofficial].
2.  **Stroke Initiation:** The user places a brush stroke in the visualization area [S1, unofficial].
3.  **Coordinate Transformation:**
    *   If **2D mode** is active: The stroke position is interpreted as screen-space coordinates $(u, v)$ [S1, unofficial]. The brush tip is circular in screen space [S1, unofficial].
    *   If **3D mode** is active: The stroke position is interpreted as 3D world/volumetric coordinates [S1, unofficial]. The brush tip is spherical in 3D space [S1, unofficial].
4.  **Envelope Expansion:** The user places additional brush strokes to spread the paint, dynamically expanding the envelope [S1, unofficial].
5.  **Rendering/Usage:** The resulting implicit field is evaluated during volume rendering to mask or highlight the ROI [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
*   **Brush Tip Geometry:** Circular for 2D mode, Spherical for 3D mode [S1, unofficial].
*   **Painting Coordinate Space:** Screen space (2D) or World/Volumetric space (3D) [S1, unofficial].
*   **Metaball Threshold:** Not given in the sources [S1, unofficial].

**Pitfalls:**
*   **2D Envelope Misalignment:** 2D envelopes act like X-ray projections, potentially obscuring or misaligning with internal 3D structures when the object rotates, requiring explicit mode switching [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Scaling:** Not given in the sources [S1, unofficial].
*   **Strengths:** Intuitive, eliminates need for transfer function expertise, provides organic shapes [S1, unofficial].
*   **Limitations:** 2D mode suffers from depth ambiguity and acts like an X-ray projection; 3D mode maintains relative position to the object [S1, unofficial].
*   **When to use:** For intuitive volume exploration and ROI definition [S1, unofficial].
*   **When not:** Not given in the sources [S1, unofficial].

## Background
This method was proposed and implemented as part of the ImageVis3D extensions described in the thesis "Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor" [S1, unofficial]. It aims to overcome domain knowledge and interface complexity issues in volume visualization by providing an integrated 3D single-view interaction model with minimum controls [S1, unofficial]. It was chosen to allow users to concentrate on gaining insight into the data rather than tweaking visualization controls [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
