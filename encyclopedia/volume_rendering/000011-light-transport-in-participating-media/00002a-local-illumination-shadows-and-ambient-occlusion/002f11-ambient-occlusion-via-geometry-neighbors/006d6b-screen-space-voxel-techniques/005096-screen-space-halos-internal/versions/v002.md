# Screen-space Halos (Internal)

Also called: Depth-based Halo Generation, Internal Halos, VOM Halo Rendering, Vicinity Occlusion Map Halo Rendering

## Overview
Screen-space Halos (Internal) is a technique to render halos over non-background pixels (internal halos) by analyzing depth discontinuities and using average depth to modulate halo intensity, preventing halo rendering where depth variation is minimal [S1, unofficial]. It sets itself apart from basic halo generation by operating on opaque structures within the volume rather than just the background, and from ambient occlusion methods by providing shape cues through color blending rather than just darkening [S1, unofficial]. It builds on [Local illumination, shadows and ambient occlusion](34-local-illumination-shadows-and-ambient-occlusion.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Depth map $D$, Phong-shaded color buffer $C_{Phong}$, halo color $C_{halo}$, halo scaling factor $factor_{halo}$, vicinity window size $(size_x, size_y)$, and a depth threshold $T$ [S1, unofficial].
*   **Outputs:** Final color buffer $C_{final}$ [S1, unofficial].

**Data Structures:**
*   **Depth Map:** 2D array of depth values. Background pixels have depth $0$ [S1, unofficial].
*   **Average Depth Map:** 2D array storing the average depth of the vicinity for each pixel [S1, unofficial].

**Steps:**
1.  **Compute Average Depth:** For every pixel $[x, y]$, calculate the average depth of its neighbors within a window of size $(2size_x + 1) \times (2size_y + 1)$ [S1, unofficial].
    $$ avgdepth[x, y] = \frac{\sum_{i=x-size_x}^{x+size_x} \sum_{j=y-size_y}^{y+size_y} D[i, j]}{NElems} $$
    where $NElems = (2size_x + 1) \times (2size_y + 1)$ [S1, unofficial].

2.  **Calculate Halo Intensity:**
    *   For **internal halos** (non-background pixels where $D[x, y] > 0$):
        $$ inthalo = (avgdepth[x, y] - D[x, y]) \times factor_{halo} $$
        [S1, unofficial]
    *   For **background pixels** ($D[x, y] = 0$), the intensity is effectively zero or handled separately to avoid artifacts, as the pixel's own depth does not affect the intensity function [S1, unofficial].
    *   **Thresholding:** A threshold based on pixel depth is used to avoid rendering halos at every pixel with minor depth variations. Elements above this threshold contribute to the average depth calculation to decrease halo intensity appropriately [S1, unofficial].

3.  **Blend Colors:** Combine the original Phong color with the halo color based on the calculated intensity [S1, unofficial]:
    $$ C_{final}[x, y] = (1 - inthalo) \times C_{Phong}[x, y] + inthalo \times C_{halo} $$
    [S1, unofficial]

**Parameters:**
*   $factor_{halo}$: User-defined weight to scale halo intensity. Typical values range from $0.39$ to $0.78$ depending on desired perceptibility [S1, unofficial].
*   $C_{halo}$: User-defined halo color. Should not be black, as black halos make it difficult to distinguish between lighting darkening and halo effects [S1, unofficial].
*   $size_x, size_y$: Define the resolution of the vicinity. Default values are not specified, but can be adjusted interactively [S1, unofficial].

**Pitfalls:**
*   **Semi-transparent pixels:** Pixels with semi-transparency often have depth $0$ in the depth map and are not emphasized with halos [S1, unofficial].
*   **Black Halo Color:** Using black for $C_{halo}$ reduces shape cues because it is hard to judge if darkening is due to lighting or the halo [S1, unofficial].
*   **Threshold Selection:** Incorrect thresholding may cause halos to appear where depth variation is minimal or miss significant discontinuities [S1, unofficial].

## Subtopics

### Structure Enhancement via Segmented Halos
A technique to generate local halos around specific segmented structures. It uses the depth of segmented structures to compute the Vicinity Occlusion Map (VOM) while using other elements' depth only for occlusion checks. If a halo pixel is occluded by a closer structure, its color remains untouched [S1, unofficial]. Correct occlusion handling is required so foreground structures (e.g., ribs) do not incorrectly display halos behind them [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the screen space, as the average depth computation is a box filter operation [S1, unofficial].
*   **Memory Complexity:** Requires storage for the depth map, average depth map, and color buffers [S1, unofficial].
*   **Performance:** Real-time frame rates are achieved for large volume models (up to $512 \times 512 \times 512$ voxels) on hardware such as an Intel Core 2 Duo CPU and NVIDIA GeForce GTX 280 GPU [S1, unofficial]. The penalty for generating halos is low compared to the ray-casting process [S1, unofficial].
*   **Strengths:** Real-time computation, interactive parameter adjustment, enhances shape perception of internal structures [S1, unofficial].
*   **Limitations:** Relies on accurate depth maps; semi-transparent pixels may not be handled correctly; requires segmentation for structure-specific halos [S1, unofficial].

## Background
The method was introduced in the context of depth-enhanced Direct Volume Rendering (DVR) to improve the perception of internal structures. It is part of a screen-space approach that computes ambient occlusion and halos in real-time [S1, unofficial]. The technique was tested on volume models up to $512^3$ voxels, demonstrating its efficiency for large datasets [S1, unofficial].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
