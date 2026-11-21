# Conical MImP

Also called: Conical Maximum Importance Projection, Countersink MImP, MImP (Conical), Maximum Importance Projection

## Overview
Conical MImP is a maximum importance projection technique that corrects the spatial ambiguity of cylindrical MImP by using a conical clipping frustum instead of a cylindrical one [S2, unofficial]. While standard MImP renders only the object with the highest importance along a ray and hides all others, the cylindrical variant creates a "cut-away" view where the focus object appears to float in front of the context object without depth cues [S2, unofficial]. Conical MImP scales the footprint of the focus object during the translational sweep towards the viewer, creating a countersink geometry that reveals the spatial relationship between the focus and context objects [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset with associated importance values for each voxel; a viewing direction; a focus object identified by having the highest importance along specific rays.
*   **Output:** A 2D image where the focus object is rendered densely, and context objects are rendered with high sparseness (transparency), adjusted by the conical clipping geometry.

**Data Structures:**
*   **Depth Footprint Image:** A 2D image where each pixel $(u, v)$ stores the depth $z$ of the last-hit sample of the focus object along the viewing ray. If the focus object does not intersect the ray, the pixel is marked as empty or background.
*   **Distance Field:** A 2D image derived from the depth footprint, storing the chamfer distance $d_i$ for each pixel $i$ from the boundary of the focus object's footprint.
*   **Ray Start Points Buffer:** A 2D image storing the modified starting depth $e_i$ for each ray.

**Steps:**

1.  **Compute Depth Footprint:**
    For each pixel in the image plane, traverse the ray through the volume. Identify the sample with the highest importance value. Record the depth $z_{last}$ of this sample as the footprint depth for that pixel. Let $e_{max}$ be the maximum depth value found in the entire footprint image.

2.  **Compute 2D Chamfer Distance Transform:**
    Process the binary mask of the footprint (pixels where the focus object exists) using a 2D chamfer distance transform [S2, unofficial]. This produces a distance field where each pixel $i$ contains a distance value $d_i$ representing its distance to the nearest edge of the footprint.

3.  **Calculate Ray Starting Points (Countersink Geometry):**
    Define a slope factor $s_c$ for the countersink frustum. Calculate the new starting depth $e_i$ for each pixel $i$ in the footprint using the formula:
    $$ e_i = e_{max} - d_i \cdot s_c $$
    Here, $e_{max}$ is the highest depth value of the footprint, $d_i$ is the distance value at pixel $i$, and $s_c$ is the slope of the countersink [S2, unofficial].
    *   For pixels inside the footprint where $d_i = 0$ (center of the object), $e_i = e_{max}$. The ray starts at the back of the object.
    *   For pixels near the edge where $d_i > 0$, $e_i < e_{max}$. The ray starts earlier, closer to the viewer, effectively "slicing" into the context object to reveal depth.

4.  **Modify Gradient Vectors:**
    To correctly simulate the cut-out effect and shading, modify the gradient vector at entry points contributing to the countersink effect [S2, unofficial].
    *   The $z$-component of the gradient is set to a constant value equal to the slope of the countersink frustum ($s_c$).
    *   The $x$ and $y$ components are estimated from the gradient information of the 2D distance field [S2, unofficial].

5.  **Ray Traversal and Compositing:**
    For each pixel, start the ray traversal at the calculated $e_i$.
    *   If the pixel is outside the footprint, use standard cylindrical MImP logic or standard volume rendering for context objects.
    *   If the pixel is inside the footprint, traverse from $e_i$ to the end of the volume.
    *   Apply importance compositing: Determine the object with the highest importance along the ray segment. Render this object densely. Render all other objects along the ray with the highest level of sparseness (fully transparent) [S2, unofficial].

**Parameters:**
*   $s_c$ (Countersink Slope): Controls the steepness of the conical clipping. A higher $s_c$ creates a steeper cone, revealing more of the context object's depth. Typical values are not specified in the sources [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation of $e_i$ depends on the accuracy of the chamfer distance transform and the depth buffer. [G] Ensure floating-point precision is sufficient to handle small $d_i$ values near the footprint boundary to avoid jagged artifacts.
*   **Empty Footprint:** If a ray does not intersect the focus object, the footprint is empty. The algorithm should fall back to standard rendering or cylindrical MImP for that pixel.
*   **Gradient Estimation:** The estimation of $x$ and $y$ gradient components from the 2D distance field is an approximation. [G] In practice, this may require smoothing or filtering to avoid noisy normals at the countersink boundary.

## Subtopics

### Cylindrical MImP
The baseline method where the clipping geometry is a translational sweep (cylinder) with the footprint as the cross-section. It clips the space in front of the most important object but fails to show spatial arrangement [S2, unofficial]. See [Streaming MIP Implementation (Basic)](1803-streaming-mip-implementation-basic.md).

### Countersink Geometry
The specific conical shape used in Conical MImP, defined by scaling the footprint during the sweep. It corrects the visual artifact of the cylindrical approach [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The dominant cost is the 2D chamfer distance transform on the footprint image, which is $O(N)$ where $N$ is the number of pixels in the footprint. Ray traversal remains $O(R)$ where $R$ is the number of rays.
*   **Memory Complexity:** Requires storage for the depth footprint image, the distance field, and the ray start points buffer. All are $O(N)$ for an image of size $N$.
*   **Strengths:** Correctly visualizes the spatial relationship between focus and context objects, resolving the "floating" artifact of cylindrical MImP [S2, unofficial].
*   **Limitations:** Requires pre-computation of the footprint and distance field. The choice of slope factor $s_c$ is subjective and affects the visual result [S2, unofficial].
*   **When to use:** When the spatial arrangement of occluded structures is critical for interpretation, such as in medical visualization where the relationship between a tumor and surrounding vessels must be clear.
*   **When not to use:** When performance is critical and the footprint computation is too expensive, or when the focus object is small and the countersink effect is negligible.

## Background
Conical MImP was introduced by Ivan Viola and M. Eduard Gröller as part of their work on importance-driven volume rendering [S1, unofficial][S2, unofficial]. It addresses a specific limitation of Maximum Importance Projection (MImP), which selects the object with the highest importance along a ray and renders it densely while hiding others [S2, unofficial]. The authors noted that standard MImP, like Maximum Intensity Projection (MIP), does not clearly show spatial arrangement, making it difficult to perceive depth relationships [S2, unofficial]. The conical variant was developed to alleviate this by modifying the clipping geometry to a countersink frustum [S2, unofficial].

## Sources
[S1] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S2] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
