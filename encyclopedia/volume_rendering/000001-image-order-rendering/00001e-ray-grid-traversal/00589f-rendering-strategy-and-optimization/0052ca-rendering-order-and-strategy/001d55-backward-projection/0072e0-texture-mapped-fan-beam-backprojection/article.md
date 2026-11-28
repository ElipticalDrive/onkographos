# Texture-Mapped Fan-Beam Backprojection

Also called: Divergent beam (fan-beam) backprojection, Fan-Beam Texture Reconstruction, Fan-beam algorithm

## Overview
This method extends parallel beam reconstruction to divergent beam geometry by leveraging hardware-accelerated texture mapping and z-depth cueing to handle the $1/r^2$ attenuation term without additional computational cost [S1]. It differs from [Parallel Beam Backprojection](52-name.md) by using a triangular fan geometry instead of a circular strip, while maintaining similar rendering complexity [S1].

## How it works
**Inputs:** Filtered projection data $p(s, \eta)$, where $s$ is the detector coordinate and $\eta$ is the source angle.
**Outputs:** Reconstructed volume image stored in the framebuffer.

**Data Structures:**
*   **Projection Texture:** A 2D texture map where one axis corresponds to detector channels ($s$) and the other to projection angles ($\eta$). The texture is updated or rotated for each source position.
*   **Framebuffer:** Accumulates the backprojected values.

**Steps:**
1.  Let $p$ be the filtered projection data [S1].
2.  For each source angle $\eta_n$:
    a.  Rotate the entire projection texture by $\eta_n$ to align with the current source position [S1].
    b.  Setup z-depth cueing specifically for the divergent beam geometry to encode the $1/r^2$ attenuation factor into the depth buffer or color value [S1].
    c.  Render a texture-mapped fan (triangle) using the rotated texture $p(s_n, \eta_n)$ [S1]. The fan geometry represents the divergent rays originating from the source point.
    d.  Accumulate the rendered result into the framebuffer [S1].
3.  Scale and remap the accumulated framebuffer values to produce the final reconstruction [S1].

**Pitfalls:**
*   The $1/r^2$ attenuation behavior must be correctly handled by the z-depth cueing setup; failure to do so results in incorrect intensity falloff [S1].
*   Ensure the triangular fan geometry accurately represents the divergent beam spread to avoid geometric distortion [S1].

## Subtopics

### Texture-Mapped Cone-Beam Backprojection
This variant extends the fan-beam approach to 3D cone-beam geometry. It follows the same algorithmic structure: rotate the projection, setup depth cueing for the divergent cone, render a texture-mapped cone/fan, accumulate, and scale/remap [S1]. The primary difference is the geometry of the primitive (cone vs. fan) and the handling of the third dimension [S1].

## Operating characteristics
*   **Performance:** Significantly faster than previous software-based algorithms, rivaling specialized hardware [S1].
*   **Hardware:** Tested on Silicon Graphics Onyx with RealityEngine2 graphics systems [S1].
*   **Complexity:** The triangular shape of the fan is no harder to render than the circle used for parallel beam cases [S1].
*   **Cost:** The $1/r^2$ attenuation handling via z-depth cueing incurs no extra rendering time cost [S1].

## Background
Originated from research into using texture-mapping hardware for tomographic image reconstruction [S1]. Initial performance studies were conducted on 2D image reconstructions using a four Raster Manager (RM) Reality Engine2 Onyx with a 150 MHz R4400 CPU [S1]. The method exploits the fact that texture mapping and frame-buffer accumulation are hardware-accelerated on state-of-the-art graphics systems of the time [S1].

## Sources
[S1] Tomographic image reconstruction and rendering with texture-mapping hardware, https://doi.org/10.1117/12.179259, fetched 2026-10-26
