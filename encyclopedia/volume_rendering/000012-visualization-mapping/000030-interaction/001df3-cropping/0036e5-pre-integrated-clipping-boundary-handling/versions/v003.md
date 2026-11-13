# Pre-integrated Clipping Boundary Handling

Also called: Opaque texture mapping for clip boundaries, boundary layer handling in pre-integrated rendering

## Overview
Pre-integrated Clipping Boundary Handling is a technique to eliminate visual artifacts at clip boundaries when using pre-integrated volume rendering with low sampling rates [S1, unofficial]. It solves the problem where standard pre-integration fails to account for the optical interaction between the clipping surface and the first volume slice, which can cause incorrect transparency or "halo" effects. It sets itself apart from sibling methods like [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md) by actively computing the ray segment contribution between the clip surface and the volume data, rather than simply suppressing or masking pixels.

## How it works
**Inputs and Outputs:**
*   Input: Clipping geometry (proxy polygons), volume texture, viewing ray origin and direction, pre-integration lookup table (LUT).
*   Output: Correctly composited color and alpha for the clipped volume view.

**Data Structures:**
*   **Clipping Geometry:** A set of proxy polygons defining the clip boundary.
*   **Volume Texture:** 3D texture storing scalar field values.
*   **Pre-integration LUT:** 2D texture mapping $(v_1, v_2)$ to $(\alpha, c)$ for a fixed step size.

**Steps:**
1.  **Render Clipping Surface:** Render the clipping geometry (opaque texture) into the color and depth buffers. This establishes the position $P$ of the clip boundary for each pixel.
2.  **Ray-Surface Intersection:** For each pixel, identify the point $P$ on the clipping surface.
3.  **Find First Volume Intersection:** Determine the intersection point $Q$ of the viewing ray passing through $P$ with the first proxy polygon (or volume slice boundary) behind $P$ [S1, unofficial].
4.  **Sample Volume Values:** Look up the scalar field values $v_P$ at point $P$ and $v_Q$ at point $Q$ from the volume texture. Note: $P$ may be outside the volume data range, requiring clamping or extrapolation [G].
5.  **Compute Segment Contribution:** Calculate the optical contribution of the ray segment between $P$ and $Q$. This involves computing the sampling distance $d$ (or $\Delta d$) corresponding to the segment length. The actual geometric length is not strictly required if the pre-integration LUT is parameterized by optical thickness, but consistency requires mapping the geometric distance to the appropriate LUT index based on the gradient or average density [S1, unofficial].
6.  **Composite:** Use the pre-integration LUT with values $v_P$ and $v_Q$ to retrieve the accumulated alpha and color for the segment $P \to Q$. Composite this result with the background (or subsequent volume slices) using standard front-to-back or back-to-front accumulation.

**Formulas:**
Let $v_P$ be the value at the clip surface and $v_Q$ be the value at the first volume sample. The pre-integration lookup function $L(v_1, v_2)$ returns $(\alpha_{seg}, c_{seg})$.
The final color $C$ and alpha $\alpha$ for the pixel are computed by combining the segment contribution with the rest of the ray:
$$ \alpha_{final} = \alpha_{seg} + (1 - \alpha_{seg}) \alpha_{rest} $$
$$ C_{final} = c_{seg} + (1 - \alpha_{seg}) C_{rest} $$
where $\alpha_{rest}$ and $C_{rest}$ are the accumulated alpha and color from the volume behind $Q$.

**Parameters:**
*   **Sampling Rate:** Low sampling rates exacerbate artifacts, making this method critical [S1, unofficial].
*   **Clip Geometry Resolution:** The proxy polygons must be fine enough to approximate the true boundary.

**Edge Cases and Pitfalls:**
*   **Interleaved Rendering:** An alternative optical model using a boundary layer of finite thickness requires interleaved rendering of volume and clip surface, which is more complex and costly [S1, unofficial].
*   **Feature Visibility:** The primary goal of clipping is often to show relationships between features (e.g., distances between isosurfaces). If the opaque texture is not handled carefully, it may obscure these relationships [S1, unofficial].
*   **Depth Buffer Overhead:** Rendering the depth buffer multiple times yields minimal overhead on current GPUs [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Adds one pass for the clipping surface and one fragment program execution per pixel for the boundary segment. The overhead is minimal on modern GPUs [S1, unofficial].
*   **Memory Complexity:** Requires storage for the clipping geometry and the pre-integration LUT. No additional large buffers are needed beyond standard depth/color buffers.
*   **Strengths:** Eliminates boundary artifacts without complex interleaved rendering; works well with low sampling rates.
*   **Limitations:** Requires accurate ray-surface intersection; may obscure features if the clip surface is opaque and not blended correctly.
*   **When to Use:** When using pre-integrated volume rendering and low sampling rates, and when accurate boundary representation is critical.
*   **When Not to Use:** When high sampling rates are used (artifacts are less visible) or when the clip boundary is not planar/convex and proxy geometry is difficult to construct.

## Background
Originated in the context of interactive volume rendering interfaces, specifically in the "Focus and Context Interface" work by I. K. et al. [S1, unofficial]. It addresses the limitations of early pre-integration techniques which did not account for the optical properties of clip boundaries.

## Sources
[S1] PDF A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
