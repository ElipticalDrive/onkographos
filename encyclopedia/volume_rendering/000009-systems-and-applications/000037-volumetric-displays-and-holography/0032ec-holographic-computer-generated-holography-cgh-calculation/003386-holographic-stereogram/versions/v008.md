# Holographic Stereogram

Also called: HPO Stereogram, Horizontal-Parallax-Only Stereogram, Stereogram

## Overview
A Holographic Stereogram is a type of computer-generated hologram composed of a series of discrete 2D perspective views of an object scene [S1]. It produces a view-dependent image that presents the corresponding perspective view in each horizontally displaced direction, functioning similarly to lenticular or parallax barrier displays [S1]. Unlike full-parallax methods that compute complex interference patterns for every pixel, this method separates the problem into rendering standard 2D images and combining them into a fringe pattern, allowing for computation at nearly interactive rates on specialized hardware [S1].

## How it works
The method generates a hologram by synthesizing a sequence of perspective views and combining them into a displayable fringe pattern.

**Inputs and Outputs:**
*   **Input:** A 3D scene description (e.g., point cloud or mesh), camera parameters (position, orientation, field of view), and hologram display specifications (pixel pitch, resolution).
*   **Output:** A 2D fringe pattern (intensity or phase map) suitable for display on a Spatial Light Modulator (SLM) or printed holographic film.

**Data Structures:**
1.  **View Buffer:** A 2D array of images, where each element corresponds to a specific horizontal viewpoint. The vertical resolution matches the hologram's vertical pixel count.
2.  **Fringe Pattern:** A 2D array representing the final hologram plane. For HPO, this is typically a 1D fringe structure repeated vertically or a 2D array where horizontal variations encode the parallax.

**Steps:**
1.  **Generate Perspective Views:** Render a sequence of 2D images of the object scene. The virtual camera is moved laterally (along the horizontal axis) in discrete steps. The number of steps determines the horizontal resolution of the parallax [S1].
2.  **Combine into Fringe:** Combine the generated view images to generate a fringe for display. In an HPO stereogram, the horizontal axis of the hologram is divided into segments, each corresponding to a specific view. The vertical axis contains the full image content for that view [S1].

**Formulas:**
Let $N_v$ be the number of views. Let $I_k(x, y)$ be the $k$-th rendered view image, where $k \in \{0, \dots, N_v-1\}$. The hologram plane $H(x, y)$ is constructed by mapping the horizontal coordinate $x$ to the view index $k$.
$$ k = \left\lfloor \frac{x}{W_v} \right\rfloor $$
where $W_v$ is the width of each view segment in the hologram plane. The intensity at $(x, y)$ is:
$$ H(x, y) = I_k(x \mod W_v, y) $$
Note: This simple mapping assumes ideal sampling. In practice, interpolation or diffraction-based encoding may be applied to the fringe to reduce aliasing [S1].

**Parameters:**
*   **Number of Views ($N_v$):** Must be large enough to create a high-quality image free from sampling artifacts. A higher $N_v$ improves smoothness but increases computational load [S1].
*   **Field of View (FOV):** Determines the range of lateral camera movement. Larger FOVs require more views [S2].

**Edge Cases and Pitfalls:**
*   **Sampling Artifacts:** If the number of perspective views is insufficient, the reconstructed image will exhibit visible discontinuities or aliasing artifacts [S1].
*   **Computational Complexity:** The need for a large number of views limits computation speed if implemented on general-purpose hardware [S1].
*   **Aperture Diffraction:** When extracting views from a pre-existing hologram (e.g., for light field displays), sharp rectangular apertures can cause diffraction artifacts. Apodization is recommended to suppress side-lobe leakage [S2].

## Subtopics

### View Extraction
This variant involves extracting different views from an existing hologram rather than generating them from a 3D scene. A window with the resolution of the vertical axis of the hologram moves along the horizontal axis to acquire sub-holograms that simulate a pinhole camera view [S2]. The apertures extracted can be chosen densely enough to overlap, as increasing the number of views improves final rendering quality [S2]. However, the targeted Field of View (FOV) is crucial; bigger FOVs require more views and increased computational complexity [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of views and the resolution of each view. The method allows for computation at nearly interactive rates when implemented on specialized hardware [S1].
*   **Memory Complexity:** Proportional to the number of views times the resolution of each view.
*   **Strengths:** High computational efficiency compared to full diffraction-based CGH methods; suitable for real-time or near-real-time applications on specialized hardware [S1].
*   **Limitations:** Limited to horizontal parallax only (HPO); requires a large number of views to avoid sampling artifacts, which can limit speed on general hardware [S1].
*   **When to Use:** When horizontal parallax is sufficient for the application and computational resources are limited or specialized hardware is available.
*   **When Not to Use:** When full 3D parallax (vertical and horizontal) is required, or when high-quality full-parallax holography is needed without specialized hardware.

## Background
The holographic stereogram approach was introduced as a way to achieve interactive 3D holographic displays by breaking away from the traditional simulation of optical holographic interference [S1]. It was developed to address the computational burden of full CGH methods, which generally employ supercomputers and use simple images to achieve interactive display [S1]. The approach produces an image with resolution that is finer than can be utilized by the human visual system, but at a fraction of the computational cost [S1]. New techniques continue to improve image quality and computational ease of stereograms [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
