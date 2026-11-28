# Aggregate Cull Window (ACW) Reduction

Also called: ACW Reduction, Dynamic ACW Reduction

## Overview
Aggregate Cull Window (ACW) Reduction is a technique to decrease the size of the Aggregate Cull Window (ACW) so that rendering terminates earlier [S1, unofficial]. It sets itself apart from standard spatial culling by incorporating image-based feedback and geometric refinement to dynamically shrink the culling bounds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current Aggregate Cull Window (ACW) bounds, subvolume geometry, framebuffer image data.
*   **Output:** Reduced ACW bounds, visibility status for subvolumes.

**Data Structures:**
*   **ACW:** A bounding volume (typically a frustum or box) representing the region of interest.
*   **Portals:** Rectangular intersections between adjacent boxes, used to approximate the shape of the volume (e.g., a colon) [S1, unofficial].
*   **Framebuffer:** The current image buffer containing rendered pixel data [S1, unofficial].

**Steps:**
1.  **Portal Generation:** Generate more portals inside each box to better capture the shape of the colon [S1, unofficial]. This improves the accuracy of the spatial culling but increases the number of portals to project [S1, unofficial].
2.  **ACW Evaluation:** Project the generated portals to determine the initial ACW size [S1, unofficial].
3.  **Threshold Check:** Determine if the current ACW size is smaller than a predefined threshold [S1, unofficial].
4.  **Image-Based Reduction (Conditional):** If the ACW size is below the threshold:
    *   Access the framebuffer image for each subvolume inside the ACW [S1, unofficial].
    *   Check if the subvolume's image contains rows of opaque pixels [S1, unofficial].
    *   If opaque rows are found, shrink the ACW accordingly [S1, unofficial].
5.  **Dynamic Combination:** Dynamically combine the portal-based and image-based methods for the most effective ACW reduction [S1, unofficial].

**Parameters:**
*   **Threshold:** A specific size value for the ACW. If ACW size < threshold, perform additional pixel checking; otherwise, rely on the standard ACW intersection logic [S1, unofficial].

**Pitfalls:**
*   Generating more portals increases the expense of ACW evaluation because more portals must be projected [S1, unofficial].
*   Accessing the framebuffer is expensive; this operation is only performed when the ACW size is smaller than a certain threshold [S1, unofficial].

## Subtopics

### Adaptive ACW Thresholding
A dynamic combination method to reduce the cost of Aggregate Cull Window (ACW) calculations by only performing additional pixel checking when the ACW size is smaller than a certain threshold [S1, unofficial]. It relies on a threshold parameter: if ACW size < threshold, perform additional pixel checking; otherwise, rely on the standard ACW intersection logic [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Increases with the number of portals due to projection costs [S1, unofficial]. Framebuffer access is expensive and is conditionally executed [S1, unofficial].
*   **Memory Complexity:** Depends on the number of portals and framebuffer size [S1, unofficial].
*   **Strengths:** Enables earlier termination of rendering by more accurately culling non-visible subvolumes [S1, unofficial].
*   **Limitations:** High cost of framebuffer access and portal projection if not managed carefully [S1, unofficial].
*   **When to use:** When interactive rendering of complex, twisted volumes (e.g., colon) is required, and standard culling is insufficient [S1, unofficial].

## Background
Originated in the context of real-time volume rendering, specifically for medical imaging applications like colon visualization [S1, unofficial]. It builds on the Window-Closing Visibility Detection method [21435-window-closing-visibility-detection.md].

## Sources
[S1] Real-Time Volume Rendering, https://cfcs.pku.edu.cn/baoquan/docs/20180622110335231616.pdf, fetched 2026-10-30 (unofficial)
