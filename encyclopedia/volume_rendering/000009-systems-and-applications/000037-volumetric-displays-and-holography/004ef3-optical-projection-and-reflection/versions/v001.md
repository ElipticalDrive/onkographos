# Optical Projection and Reflection

Also called: Pepper's Ghost, Perspective Projection Backdrop, Pepper's Cone

## Overview
Optical Projection and Reflection techniques generate volumetric or pseudo-volumetric images by reflecting light from a 2D display source onto semi-reflective surfaces, creating the illusion of floating 3D objects in free space. This family solves the problem of presenting 3D content with natural occlusion cues without requiring physical particles or scanning beams in the viewing volume. It differs from scanning methods (like [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md)) by using static optical elements to create persistent images, and from holographic methods (like [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md)) by relying on geometric reflection rather than wavefront interference.

## Shared principles
The core mechanism involves a display source emitting light that strikes a translucent or semi-reflective surface. The surface reflects a portion of the light toward the viewer while transmitting the remainder. By stacking multiple such surfaces or using specific geometries, distinct image slices are superimposed to create depth.

**Key Mathematical Principles:**

1.  **Contrast and Occlusion:** The perception of a floating volume relies on high contrast between the projected image and the real background. If $I_{img}$ is the intensity of the projected image and $I_{bg}$ is the intensity of the real background, the contrast $C$ at the boundary is defined as:
    $$ C = \frac{I_{img} - I_{bg}}{I_{img} + I_{bg}} $$
    High contrast (bright image, dark background) is essential for the "floating" effect [S1].

2.  **Perspective Projection:** To simulate virtual images located behind the physical drawing volume, the display content must be projected based on the viewer's position. For an observer at point $E = (x_0, y_0, z_0)$ and an image point $X = (x, y, z)$, the perspective projection onto a plane $P$ (perpendicular to the line $EO$, where $O$ is the origin) is governed by the matrix $S$:
    $$ S = \begin{bmatrix} -r^2 & r^2 x_0 & 0 & 0 \\ -r x_0 z_0 & -r y z_0 & r \rho^2 & 0 \\ 0 & 0 & 0 & r \rho \\ -\rho x_0 & -\rho y_0 & -\rho z_0 & r \rho \end{bmatrix} $$
    where:
    $$ r = \frac{1}{\sqrt{x_0^2 + y_0^2 + z_0^2}} $$
    $$ \rho = \frac{1}{\sqrt{x_0^2 + y_0^2}} $$
    [S1]

3.  **Layer Properties:** In multi-layer systems (e.g., SliceView), each plane $i$ has a thickness $t_i$ and optical properties defined by reflection coefficient $R_i$ and transmission coefficient $T_i$. The thickness $t_i$ is a critical parameter for balancing brightness and depth perception, though exact optimization formulas are not specified in the sources [S2].

## Subtopics

### [SliceView](7774-sliceview.md)
SliceView extends the classical Pepper's Ghost illusion to multiple stacked translucent planes, cumulatively creating a three-dimensional volume from initially two-dimensional projections. It solves the problem of presenting 3D content with natural occlusion cues by reflecting images from a display source onto semi-reflective surfaces positioned between the viewer and the real background. Unlike stereoscopic displays, SliceView avoids vergence accommodation conflict (VAC) because the virtual object and the real background exist at different optical distances, with the image acting as a reflection rather than a focused 2D plane. It differs from persistence-of-vision displays by using static stacked planes rather than moving parts.

*   **Inputs:** A 2D image or video stream with a pure black background.
*   **Outputs:** A virtual 3D image perceived by the viewer.
*   **Data Structures:**
    *   **Display Source:** LCD, LED, or projector capable of high-contrast images.
    *   **Optical Stack:** $N$ translucent planes at depths $z_i$ with thickness $t_i$ and coefficients $R_i, T_i$.
*   **Steps:**
    1.  **Image Preparation:** Render subject against pure black background.
    2.  **Projection:** Display image on source screen.
    3.  **Reflection:** Light strikes first semi-reflective surface; portion reflects to viewer, rest transmits.
    4.  **Layering:** Process repeats for subsequent planes, each contributing a slice at specific depth.
    5.  **Perception:** Viewer sees superimposed reflections; black background ensures virtual object occludes real background.
*   **Parameters:**
    *   **Layer Thickness ($t_i$):** Optimized for image quality; specific values not given.
    *   **Number of Layers ($N$):** Multiple required; specific count not given.
    *   **Angle of Incidence:** Determined by geometry of planes relative to viewer.
*   **Edge Cases:**
    *   **VAC:** Avoided between virtual object and real background.
    *   **Black Background:** Non-black backgrounds reduce contrast and illusion.
    *   **Light Leakage:** Deeper layers may leak through upper layers; mitigated by optimizing $t_i$ and optical properties.
*   **Link:** [SliceView](7774-sliceview.md)

### Perspective Projection Backdrop
A technique to simulate virtual images (located behind the physical drawing volume) in freespace volumetric displays by projecting content onto a background image plane that moves and rotates to match the viewer's position, creating motion parallax cues consistent with a distant object [S1].

*   **Definition:** Simulates virtual images in freespace displays using time-varying perspective projection [S1].
*   **Steps:**
    1.  Define observer position $E = (x_0, y_0, z_0)$ [S1].
    2.  Define 3D image point $X = (x, y, z)$ [S1].
    3.  Define background image plane $P$, typically perpendicular to line $EO$ where $O$ is origin [S1].
    4.  Calculate intersection of line $EX$ with plane $P$ to determine pixel coordinate [S1].
*   **Link:** No separate article.

### Pepper's Cone Display
A simple 360° display based on a conical semi-reflective surface that reflects an anamorphic image into a real undistorted aerial image. It has no moving parts [S2].

*   **Definition:** 360° display using conical semi-reflective surface [S2].
*   **Steps:**
    1.  Display an anamorphic image [S2].
    2.  Reflect it off a 360° symmetrical conical semi-reflective surface [S2].
    3.  Surface creates a real undistorted aerial image [S2].
    4.  As viewer turns device, image changes to offer 360° interactive view [S2].
*   **Pitfalls:** Constrained to a limited number of viewers because only one point of view is displayed at a time [S2].
*   **Link:** No separate article.

### Mechanically Driven Mirrors
Mechanically driven mirrors are beam deflectors that are essentially lossless but suffer from slow response times. Two types are galvanometer driven and piezoelectrically driven [S3, unofficial].

*   **Definition:** Lossless beam deflectors with slow response [S3, unofficial].
*   **Parameters:**
    *   **Lowest Natural Frequency (Piezoelectric):** ~12 KHz [S3, unofficial].
    *   **Lowest Natural Frequency (Galvanometer):** ~3 KHz [S3, unofficial].
*   **Pitfalls:**
    *   **Hysteresis:** Galvanometer drives suffer hysteresis up to 4%. Must approach commanded angle from same direction, doubling random access time [S3, unofficial].
    *   **Piezoelectric:** Circumvents hysteresis via active position sensing [S3, unofficial].
    *   **Oscillation:** High response drives are highly underdamped, causing decaying oscillations that enlarge spot size and decrease boundary definition. Additional damping reduces response [S3, unofficial].
*   **Numbers:**
    *   **Max Spot Capability (30/sec refresh):** Piezoelectric: 400 spots; Galvanometer: 100 spots [S3, unofficial].
*   **Link:** No separate article.

## Background
Optical projection and reflection techniques have roots in classical theatrical illusions, specifically Pepper's Ghost, which uses a large sheet of glass to reflect a hidden stage area into the main stage [S1, S2]. Modern volumetric applications have adapted these principles for digital displays.

*   **SliceView:** Developed as an extension of Pepper's Ghost to multiple planes to create true volumetric depth cues [S2].
*   **Perspective Projection Backdrop:** Introduced in 2021 to simulate virtual images in optical trap displays (OTDs) by employing time-varying perspective projection [S1].
*   **Pepper's Cone:** Proposed as a low-cost, efficient 360° display solution using conical surfaces [S2].
*   **Mechanically Driven Mirrors:** Documented in early volumetric display studies (1970s) as a method for beam deflection, noting trade-offs between speed, hysteresis, and damping [S3, unofficial].

These methods are used in research prototypes for volumetric displays, holography, and augmented reality interfaces where natural occlusion and parallax are desired without the complexity of scanning beams or particle traps.

## Sources
[S1] Simulating virtual images in optical trap displays, https://doi.org/10.1038/s41598-021-86495-6, fetched 2026-10-29
[S2] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
[S3] DTIC AD0753438: 3D Volumetric Display Study, https://archive.org/details/DTIC_AD0753438, fetched 2026-10-26 (unofficial)
