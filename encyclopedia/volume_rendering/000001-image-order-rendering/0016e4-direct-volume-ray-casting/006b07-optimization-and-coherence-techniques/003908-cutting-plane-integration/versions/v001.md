# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive clipping technique that renders a semi-transparent plane mesh and normal arrow to dynamically slice a volume, revealing internal structures [S1, unofficial]. It differs from standard ray casting by modifying the ray integration bounds in real-time based on a user-controlled geometric plane, rather than integrating through the entire volume extent [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (typically stored as a `Float32Array` of normalized values), camera parameters, and user controls for plane position (`planeHeight`), rotation (`planeRotX`, `planeRotY`), and orientation (`renderAbove`) [S1, unofficial].
*   **Output:** A 2D image where the volume is rendered up to the cutting plane, with the plane itself rendered as a semi-transparent geometric object [S1, unofficial].

**Data Structures:**
The volume data is loaded from a binary format where the first three `uint16` values define dimensions, and subsequent values are normalized voxel intensities [S1, unofficial]:
*   `width`: `uint16` at index 0.
*   `height`: `uint16` at index 1.
*   `depth`: `uint16` at index 2.
*   `voxels`: `Float32Array` of size `width * height * depth`, where each value is `uint16 / 4095.0` [S1, unofficial].

**Algorithm Steps:**
1.  **Plane Position Calculation:** Map the normalized user input `planeHeight` ($h \in [0, 1]$) to a world Z-coordinate ($z_p$). The plane moves between $Z=200$ (at $h=0$) and $Z=-200$ (at $h=1$) [S1, unofficial]:
    $$ z_p = 200 - 400 \cdot h $$
2.  **Plane Orientation:** Calculate the rotation matrix for the plane. The Y-rotation ($\theta_y$) is derived from `planeRotY` in degrees, converted to radians. If `renderAbove` is true, an offset of $\pi$ is added to flip the plane 180° around the Y-axis [S1, unofficial]:
    $$ \theta_y = \text{radians}(\text{planeRotY}) + (\text{renderAbove} ? \pi : 0) $$
    The X-rotation ($\theta_x$) is derived directly from `planeRotX` [S1, unofficial].
3.  **Ray Casting with Clipping:** For each pixel, generate a ray. Clip the ray against the cutting plane defined by position $z_p$ and orientation $(\theta_x, \theta_y)$. The integration stops at the intersection point with the plane [S1, unofficial].
4.  **Rendering the Plane:** Render the cutting plane as a semi-transparent mesh and a normal arrow (origin at center, direction along local Z-axis, length 35, color red) to visualize the cut surface [S1, unofficial]. The arrow disables depth testing to ensure it remains visible [S1, unofficial].

**Parameters:**
*   `planeHeight`: Normalized float $[0, 1]$. Controls Z-position [S1, unofficial].
*   `planeRotX`, `planeRotY`: Floats in degrees. Control rotation around X and Y axes [S1, unofficial].
*   `renderAbove`: Boolean. Flips plane orientation by $\pi$ around Y [S1, unofficial].

**Edge Cases:**
*   **Precision:** Voxel values are normalized from `uint16` (0-4095) to floats. Loss of precision occurs if the original data had higher bit-depth [S1, unofficial].
*   **Boundaries:** The plane position is clamped to the range $[-200, 200]$ in world space by the linear mapping [S1, unofficial].

## Subtopics

### Region of Interest (ROI) Rendering
A technique to visualize internal features by defining a specific region and rendering only that portion of the dataset [S2, book]. It differs from simple cutting planes by supporting arbitrary geometric definitions (e.g., six orthographic planes, closed objects, or binary masks) rather than a single infinite plane [S2, book]. For image-order ray casting, the ray is clipped against all geometric definitions as a preprocessing step, and the ray function is evaluated only along segments within the ROI [S2, book]. For object-order methods, samples are tested against the ROI before contribution [S2, book]. Hardware clipping planes may be used to support this [S2, book].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the average ray length within the volume. The clipping operation adds a constant overhead per ray [S1, unofficial][S2, book].
*   **Memory Complexity:** Proportional to the volume size ($W \times H \times D$) for storing voxel data [S1, unofficial].
*   **Strengths:** Interactive exploration of internal structures; simple implementation using standard graphics primitives [S1, unofficial].
*   **Limitations:** Limited to planar cuts (in the base method); performance degrades with very large volumes if not optimized with spatial acceleration structures [S1, unofficial].
*   **When to use:** Interactive medical or scientific visualization where users need to slice through data to see internal features [S1, unofficial][S2, book].
*   **When not to use:** When non-planar or complex ROI definitions are required (use ROI Rendering variants) or when full-volume integration is needed [S2, book].

## Background
The technique is implemented in educational and open-source projects such as the SubairProject1 Volume Rendering Visualization Project, which demonstrates interactive clipping using WebGL/Three.js [S1, unofficial]. The concept of using clipping planes for region-of-interest visualization is documented in standard computer graphics literature, such as the VTK Book, which discusses geometric and volumetric techniques for isolating internal features [S2, book].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
