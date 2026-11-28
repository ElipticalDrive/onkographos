# Multi-view image blending

Also called: Multi-view blending

## Overview
Multi-view image blending reduces artifacts caused by single-view fading, such as ghosting and transparency variation, by combining a set of images taken from various view angles [S1]. It sets itself apart from single-texture approaches by requiring the selection and blending of at least three textures, trading increased rendering cost for improved visual fidelity across viewpoints [S1]. It differs from [Binary Swap Compositing](3264-binary-swap-compositing.md) by operating on discrete pre-rendered textures rather than iterative alpha blending [G]. It builds on [Compositing and Blending](2515-compositing-and-blending.md).

## How it works
**Inputs and outputs:**
*   **Input:** A set of pre-rendered images (textures) captured from different camera viewpoints surrounding the scene or object [S1].
*   **Output:** A single composite image for the current camera position [S1].

**Data structures:**
*   A collection of $N$ textures (images), where $N \ge 3$ [S1].
*   For each pixel in the target image, a set of corresponding sample colors and weights derived from the source textures [S1].

**Steps:**
1.  **View Selection:** For the current camera position, identify the subset of source images that provide the best coverage or lowest incidence angle error for each pixel [S1].
2.  **Sampling:** Fetch color values $c_i$ from the selected textures for each pixel [S1].
3.  **Blending:** Combine the fetched colors $c_i$ using a blending function [S1]. The specific weighting formula is not given in the sources.
4.  **Output:** Write the blended color to the final image buffer [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
*   **Number of textures:** At least 3 are required to effectively reduce artifacts [S1].

**Edge cases and pitfalls:**
*   **Ghosting:** Occurs when blending two half-faded textures is not equivalent to a single opaque texture, causing duplicated features [S1].
*   **Transparency variation:** Artifacts appear when moving around the object due to inconsistent blending of semi-transparent layers [S1].
*   **Performance:** The method is computationally expensive. Fetching 3 textures instead of a single one is approximately 2 to 3 times slower on recent GPUs [S1]. Real-time performance for complex scenes like a whole forest is generally not achievable due to the cost of selecting and blending images [S1].

## Subtopics

### Bidirectional textures / Lightfields
Relying on an even larger set of images corresponds to bidirectional textures or lightfields [S1]. This approach further reduces view-dependent artifacts but introduces a significant memory constraint: the huge amount of image data may not fit into graphics memory [S1].

### Filtering Mask Compositing
Generates the final multi-view frame $F$ by combining $N$ pre-built views $V_i$ using binary filtering masks $M_i$ [S2]. The composition is defined as $F_c = \sum_i (V_{i,c} \cdot M_c)$, where $c \in \{R, G, B\}$, $V_{i,c}$ is the color component of view $i$, and $M_c$ is the binary mask for component $c$ [S2]. The masks are defined as $\{0, 1\}^3$ over the 2D integer grid $\mathbb{Z}^2$ [S2]. The process depends on the display hardware, causing variable computing times depending on the used hardware [S2].

## Operating characteristics
*   **Time Complexity:** Significantly higher than single-texture rendering. Fetching multiple textures (e.g., 3) results in a 2x to 3x slowdown compared to single texture fetches on recent GPUs [S1].
*   **Memory Complexity:** High. Storing multiple high-resolution textures for a scene can exceed available graphics memory, especially when scaling to lightfield representations [S1].
*   **Strengths:** Reduces view-dependent artifacts like ghosting and transparency variation found in single-view methods [S1].
*   **Limitations:** Not suitable for real-time rendering of large-scale scenes (e.g., whole forests) due to the computational cost of image selection and blending [S1].

## Background
The method was discussed in the context of rendering forest scenes in real-time, aiming to improve upon single-view textured tree models [S1]. It builds on earlier work using hierarchical models with multi-layered depth images, though those were not real-time [S1]. The technique addresses specific artifacts like ghosting and transparency variation observed when moving around textured objects [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S2] Simulating Depth Perception in Virtual Microscopy, https://doi.org/10.1109/ISBI.2019.8759160, fetched 2026-10-26
