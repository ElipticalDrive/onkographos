# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive clipping technique that renders a semi-transparent plane mesh and normal arrow, dynamically updating raycasting uniforms to slice the volume and reveal internal structures [S1, unofficial]. It solves the problem where full opacity is obtained before reaching internal structures, making them difficult to see [S1, unofficial]. It sets itself apart from standard volume rendering by restricting the integration domain to a user-defined subset, which can be geometric or volumetric, rather than the entire volume extent [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (typically stored as a `Float32Array` of normalized values), camera parameters, and user-controlled plane parameters (height, rotation, flip state).
*   **Output:** A 2D image where the volume is rendered only up to the cutting plane, with the plane itself rendered as a semi-transparent mesh with a normal arrow.

**Data Structures:**
The volume data is loaded from a binary format (e.g., `uint16Array`) where the first three elements define dimensions:
*   `width`: `uint16Array[0]`
*   `height`: `uint16Array[1]`
*   `depth`: `uint16Array[2]`
*   `slice`: `width * height`
*   `size`: `slice * depth`
*   `voxels`: A `Float32Array` of length `size`, where each value is `uint16 / 4095.0` [S1, unofficial].

The cutting plane is represented by:
*   `planeMesh`: A geometric primitive positioned in world space.
*   `normalArrow`: An `ArrowHelper` visualizing the plane's normal vector, positioned at the plane's center [S1, unofficial].

**Steps:**
1.  **Load Volume:** Parse the input array to extract dimensions and normalize voxel values to [0, 1] [S1, unofficial].
2.  **Initialize Plane:** Create a plane mesh and a normal arrow. The arrow initially points along the Z-axis (0, 0, 1) [S1, unofficial].
3.  **Update Plane Position:** Map the normalized `planeHeight` parameter (0 to 1) to the world Z position. The formula is:
    $$ z_{world} = 200 - 400 \times planeHeight $$
    [S1, unofficial]
4.  **Update Plane Rotation:** Convert rotation angles `planeRotX` and `planeRotY` from degrees to radians. Apply a 180° ($\pi$ radians) offset to `planeRotY` if the `renderAbove` flag is true [S1, unofficial].
5.  **Update Raycasting Uniforms:** Pass the plane's world position and normal vector to the raycasting shader as uniforms. The raycasting loop must clip samples against this plane, stopping integration at the intersection point [S1, unofficial].
6.  **Render Plane and Arrow:** Render the semi-transparent plane mesh and the normal arrow. The arrow's depth testing is disabled (`depthTest = false`, `depthWrite = false`) to ensure it always renders on top [S1, unofficial].

**Parameters:**
*   `planeHeight`: Normalized float [0, 1] mapping to Z-axis position [S1, unofficial].
*   `planeRotX`: Rotation angle in degrees around the X-axis [S1, unofficial].
*   `planeRotY`: Rotation angle in degrees around the Y-axis [S1, unofficial].
*   `renderAbove`: Boolean flag that flips the plane 180° around Y when true [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The normalization factor `4095.0` assumes 12-bit input data. If the input data has a different bit depth, the normalization will be incorrect [S1, unofficial].
*   **Boundary Conditions:** The plane position formula assumes a specific world coordinate range. If the volume or camera coordinates differ, the mapping may need adjustment [S1, unofficial].
*   **Arrow Visibility:** The arrow's depth test disabling ensures visibility but may cause z-fighting with other transparent objects if not managed carefully [S1, unofficial].

## Subtopics

### [Region of Interest (ROI) Rendering](10737-region-of-interest-roi-rendering.md)
Region of Interest (ROI) rendering is a technique to visualize internal features by defining a specific region and rendering only that portion of the dataset. It solves the problem where full opacity is obtained before reaching internal structures, making them difficult to see. It sets itself apart from standard volume rendering by restricting the integration domain to a user-defined subset, which can be geometric or volumetric, rather than the entire volume extent.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray up to the cutting plane. The cutting plane reduces the average number of samples per ray compared to full volume rendering, potentially improving performance [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the volume size for storing the voxel data. The plane mesh and arrow add negligible overhead [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales with the resolution of the volume and the screen. The cutting plane can improve performance by reducing the integration length [S1, unofficial].
*   **Strengths:** Interactive exploration of internal structures; simple implementation; reduces visual clutter from overlying opaque structures [S1, unofficial].
*   **Limitations:** Only slices along a single plane; may not be sufficient for complex internal structures that require multiple cuts or volumetric clipping [S1, unofficial].
*   **When to use:** When interactive exploration of internal structures is required, and a single planar cut is sufficient to reveal the desired features [S1, unofficial].
*   **When not to use:** When the internal structures are complex and require multiple cuts or volumetric clipping to be fully visible [S1, unofficial].

## Background
Cutting Plane Integration is an interactive clipping technique used in volume rendering visualization projects [S1, unofficial]. It is commonly used in medical imaging and scientific visualization to explore internal structures of 3D datasets [S1, unofficial]. The technique is implemented in the SubairProject1/Volume-Rendering-Visualization-Project repository [S1, unofficial].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
