# Shell Textures

Also called: Concentric Shells

## Overview
Shell textures is a technique for rendering complex surface geometry in real-time using concentric 3D rings or shells around a model, where shells are transparent except at the intersection with the microgeometry [S1, unofficial]. It builds on [Real-time shell space rendering](11590-real-time-shell-space-rendering.md) by applying specific texture sampling strategies to the shell space to simulate volumetric microgeometry (such as fur) rather than just solid volume.

## How it works
**Inputs and Outputs:**
*   **Input:** A base polygonal mesh representing the macro-geometry; a precomputed volumetric texture (e.g., fur volume) containing microgeometry data; spherical harmonic coefficients for precomputed radiance transfer (PRT) [S1, unofficial].
*   **Output:** A rendered image with realistic microgeometry and global lighting effects at real-time frame rates [S1, unofficial].

**Data Structures:**
*   **Shell Volume:** A 3D texture or volume data structure where the radial distance from the surface corresponds to the depth within the microgeometry (e.g., hair length) [S1, unofficial].
*   **PRT Textures:** Textures storing the object’s spherical light transfer function in a spherical/zonal harmonic basis, allowing approximate illumination in arbitrary lighting environments [S1, unofficial].

**Steps:**
1.  **Shell Generation:** For each shell index $i$ from $0$ to $N-1$ (where $N$ is the total number of shells):
    a.  Perturb the vertices of the base model along their surface normals by an offset amount $d_i$. The offset increases linearly with the shell index [S1, unofficial].
    b.  Render the perturbed model into a framebuffer.
    c.  Sample the shell volume texture at the corresponding radial level for each fragment. The texture value determines the opacity and color of the shell at that intersection [S1, unofficial].
    d.  Apply alpha blending to accumulate the transparent shells [S1, unofficial].
2.  **Lighting Reconstruction:** For each rendered point:
    a.  Rotate the precomputed transfer function from its local frame to the world frame using the surface normal and tangent space [S1, unofficial].
    b.  Project the rotated transfer function back into the spherical harmonic (SH) basis [S1, unofficial].
    c.  Project the current lighting environment into the SH basis [S1, unofficial].
    d.  Compute the dot product between the lighting SH coefficients and the transfer SH coefficients to determine the final color [S1, unofficial].

**Parameters:**
*   **Number of Shells ($N$):** 16 shells are usually sufficient to create realistic fur [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Silhouette Gaps:** Shells create gaps near the object silhouette because they overlap when viewed from above but separate at edges [S1, unofficial]. This occurs because shells become parallel to the view direction at the silhouette, making gaps apparent [S1, unofficial].
*   **Remedy:** Add small textures rendered normal to the surface across all edges, faded in as the edge approaches the silhouette [S1, unofficial]. *Note: Some implementations omit this step (fins) for simplicity, accepting the gap artifact [S1, unofficial].*

## Subtopics

### Concentric Shells Fur Rendering
A variant specifically for rendering fur by generating concentric shells around a model. Each shell is rendered with an alpha-blended texture sampled from the fur volume at the corresponding level. The offset between shells creates the impression of geometric fur movement relative to the camera [S1, unofficial]. It uses a vertex shader to perturb vertices along the normal linearly for each shell draw call [S1, unofficial]. It typically uses 16 shells [S1, unofficial]. It suffers from silhouette gaps where shells become parallel to the view direction [S1, unofficial]. Unlike the base Lengyel method, this implementation may omit fins to simplify the model, relying solely on the shell textures [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of shells $N$, as the model is drawn once per shell [S1, unofficial].
*   **Memory Complexity:** Proportional to the size of the shell volume texture and the PRT transfer textures [S1, unofficial].
*   **Performance:** Capable of real-time frame rates for complex microgeometry [S1, unofficial].
*   **Strengths:** Realistic appearance of volumetric microgeometry (fur); supports global lighting effects via PRT [S1, unofficial].
*   **Limitations:** Silhouette gaps if fins are not used; lighting scheme may be limited to ambient/diffuse/specular if full self-shadowing is not implemented [S1, unofficial].

## Background
The technique was developed by John Kloetzli, combining Lengyel’s shell/fin method [2001] with Precomputed Radiance Transfer (PRT) using spherical harmonics [S1, unofficial]. It allows for real-time rendering of microgeometry with global lighting effects. The method is viewed as a real-time implementation of the Kajiya and Kay [1989] fur model, adapted for PRT [S1, unofficial].

## Sources
[S1] A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbcm.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
