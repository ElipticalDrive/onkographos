# Multi-view image blending

Also called: Multi-view blending

## Overview
Multi-view image blending reduces artifacts caused by single-view fading, such as ghosting and transparency variation, by combining a set of images taken from various view angles [S1]. It sets itself apart from single-texture approaches by requiring the selection and blending of at least three textures, trading increased rendering cost for improved visual fidelity across viewpoints [S1].

## How it works
**Inputs and outputs:**
*   **Input:** A set of pre-rendered images (textures) captured from different camera viewpoints surrounding the scene or object.
*   **Output:** A single composite image for the current camera position.

**Data structures:**
*   A collection of $N$ textures (images), where $N \ge 3$.
*   For each pixel in the target image, a set of corresponding sample colors and weights derived from the source textures.

**Steps:**
1.  **View Selection:** For the current camera position, identify the subset of source images that provide the best coverage or lowest incidence angle error for each pixel.
2.  **Sampling:** Fetch color values $c_i$ from the selected textures for each pixel.
3.  **Blending:** Combine the fetched colors $c_i$ using a blending function. While the specific weighting formula is not detailed in the sources, the process involves combining at least 3 textures [S1].
4.  **Output:** Write the blended color to the final image buffer.

**Parameters:**
*   **Number of textures:** At least 3 are required to effectively reduce artifacts [S1].

**Edge cases and pitfalls:**
*   **Ghosting:** Occurs when blending two half-faded textures is not equivalent to a single opaque texture, causing duplicated features [S1].
*   **Transparency variation:** Artifacts appear when moving around the object due to inconsistent blending of semi-transparent layers [S1].
*   **Performance:** The method is computationally expensive. Fetching 3 textures instead of a single one is approximately 2 to 3 times slower on recent GPUs [S1]. Real-time performance for complex scenes like a whole forest is generally not achievable due to the cost of selecting and blending images [S1].

## Subtopics

### Bidirectional textures / Lightfields
Relying on an even larger set of images corresponds to bidirectional textures or lightfields [S1]. This approach further reduces view-dependent artifacts but introduces a significant memory constraint: the huge amount of image data may not fit into graphics memory [S1].

## Operating characteristics
*   **Time Complexity:** Significantly higher than single-texture rendering. Fetching multiple textures (e.g., 3) results in a 2x to 3x slowdown compared to single texture fetches on recent GPUs [S1].
*   **Memory Complexity:** High. Storing multiple high-resolution textures for a scene can exceed available graphics memory, especially when scaling to lightfield representations [S1].
*   **Strengths:** Reduces view-dependent artifacts like ghosting and transparency variation found in single-view methods [S1].
*   **Limitations:** Not suitable for real-time rendering of large-scale scenes (e.g., whole forests) due to the computational cost of image selection and blending [S1].

## Background
The method was discussed in the context of rendering forest scenes in real-time, aiming to improve upon single-view textured tree models [S1]. It builds on earlier work using hierarchical models with multi-layered depth images, though those were not real-time [S1]. The technique addresses specific artifacts like ghosting and transparency variation observed when moving around textured objects [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
