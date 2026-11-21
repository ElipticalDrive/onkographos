# Holographic Stereogram

Also called: HPO Stereogram, Horizontal-Parallax-Only Stereogram, Stereogram

## Overview
A Holographic Stereogram is a type of computer-generated hologram composed of a series of discrete 2D perspective views of an object scene [S1]. It produces a view-dependent image that presents the corresponding perspective view in each horizontally displaced direction, functioning similarly to lenticular or parallax barrier displays [S1]. It restricts vertical parallax to achieve this horizontal-only view dependency [S1].

## How it works
**Inputs:** A 3D scene model or point cloud; camera parameters (field of view, resolution) [G].
**Outputs:** A 2D fringe pattern (hologram) suitable for display on an SLM or printed medium [G].
**Data Structures:** Not given in the sources.
**Steps:**
1. **View Generation:** Generate a sequence of perspective views by moving a virtual camera laterally in discrete steps across the scene [S1].
2. **Fringe Combination:** Combine the generated views to generate the holographic fringe for display [S1]. This involves mapping each view to a specific horizontal segment or column of the hologram plane [G].
3. **Display:** Output the resulting fringe pattern to a spatial light modulator (SLM) or print medium [S1].
**Formulas:** Not given in the sources.
**Parameters:**
*   **Number of Views ($N$):** Must be large enough to create a high-quality image free from sampling artifacts [S1]. The exact number $N$ depends on the target horizontal field of view (FOV), with larger FOVs requiring more views and increasing computational complexity [S2].
*   **Camera Step Size:** Determines the angular separation between views. Smaller steps improve smoothness but increase computation [G].
**Edge Cases and Pitfalls:**
*   **Sampling Artifacts:** A small number of perspective views leads to visible artifacts and poor image quality [S1].
*   **Computational Load:** The need for a large number of views limits computation speed, although specialized hardware can achieve nearly interactive rates [S1].

## Subtopics

### View Extraction
A technique used to acquire sub-holograms that simulate a pinhole camera view [S2]. A window with the resolution of the vertical axis of the hologram moves along the horizontal axis to extract different views [S2]. The apertures can be chosen densely enough to overlap, as increasing the number of views improves final rendering quality [S2]. However, larger targeted FOVs require more views and increased computational complexity [S2].

## Operating characteristics
*   **Complexity:** Computationally expensive due to the need for a large number of perspective views [S1]. Complexity scales with the number of views required for the target FOV [S2].
*   **Performance:** Allows computation at nearly interactive rates when implemented on specialized hardware [S1].
*   **Strengths:** Suitable for horizontal parallax applications [S1].
*   **Limitations:** Limited to horizontal parallax; requires many views for high quality, impacting speed [S1].

## Background
The holographic stereogram approach was introduced as a method to produce 3D images with resolution finer than can be utilized by the human visual system, avoiding the need for supercomputers in some implementations [S1]. It is based on the principle of combining discrete 2D perspective views to form a holographic fringe [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
