# SliceView

Also called: Pepper's Ghost volumetric extension, Pepper's ghost display, Pepper's ghost effect, holographic display

## Overview
SliceView is a volumetric display technique that extends the classical Pepper's Ghost optical illusion to multiple stacked translucent planes, cumulatively creating a three-dimensional volume from initially two-dimensional projections [S2]. It solves the problem of presenting 3D content with natural occlusion cues by reflecting images from a display source onto semi-reflective surfaces positioned between the viewer and the real background [S1]. Unlike stereoscopic displays, SliceView avoids vergence accommodation conflict (VAC) because the virtual object and the real background exist at different optical distances, with the image acting as a reflection rather than a focused 2D plane in the traditional sense [S1]. It differs from persistence-of-vision displays by using static stacked planes rather than moving parts or oscillating surfaces [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D image or video stream intended for projection, typically surrounded by a black background to define the object's silhouette [S1, S2].
*   **Output:** A virtual 3D image perceived by the viewer, appearing to float in space and occluding the real background behind the display [S1, S2].

**Data Structures:**
The system relies on a physical arrangement of optical components rather than a complex data structure. The core data is the pixel buffer of the display source.
*   **Display Source:** A screen (LCD, LED, or projector) capable of displaying high-contrast images with black backgrounds.
*   **Optical Stack:** A series of $N$ translucent or semi-reflective planes.
    *   **Plane Properties:** Each plane $i$ has a specific thickness $t_i$ and optical properties (reflection coefficient $R_i$, transmission coefficient $T_i$) [S2].
    *   **Geometry:** Planes are positioned at specific depths $z_i$ relative to the viewer and the display source.

**Steps:**
1.  **Image Preparation:** Generate a 2D image frame where the subject is rendered against a pure black background. This ensures that only the subject reflects light, while the background remains transparent or non-emissive [S1, S2].
2.  **Projection/Display:** Display the prepared image on the source screen. In the basic Pepper's Ghost configuration, the screen is positioned below or behind a semi-reflective surface [S1]. In the SliceView extension, the image is projected or displayed such that it reflects off multiple planes [S2].
3.  **Reflection:** The light from the display source strikes the first semi-reflective surface (or translucent plane) positioned at an angle. The surface reflects a portion of the light towards the viewer while transmitting the rest [S1, S2].
4.  **Layering:** For multiple planes, the process repeats. Light reflects off the first plane, then the second, and so on. Each plane contributes a "slice" of the volume at a specific depth [S2].
5.  **Perception:** The viewer sees the superimposed reflections from all planes. The black background of the source image ensures that the virtual object appears to occlude the real world behind it, creating a strong depth cue [S1].

**Formulas:**
*   **Occlusion Cue:** The perception of volume is driven by the occlusion between the bright image and the real background. If $I_{img}$ is the intensity of the projected image and $I_{bg}$ is the intensity of the real background, the contrast $C$ at the boundary is:
    $$ C = \frac{I_{img} - I_{bg}}{I_{img} + I_{bg}} $$
    High contrast (bright image, dark background) enhances the floating effect [S1].
*   **Layer Thickness Optimization:** The thickness of each layer $t_i$ is evaluated alongside light reflection and transmission properties to optimize image quality. Exact formulas for optimal $t_i$ are not specified in the sources, but it is noted that this parameter is critical for balancing brightness and depth perception [S2].

**Parameters:**
*   **Layer Thickness ($t_i$):** Not given in the sources as a specific value, but evaluated to optimize image quality [S2].
*   **Number of Layers ($N$):** Not specified, but "multiple planes" are required to create volume [S2].
*   **Angle of Incidence:** The translucent material is placed at a certain angle to reflect the image towards the user [S2].

**Edge Cases and Pitfalls:**
*   **Vergence Accommodation Conflict (VAC):** While stereoscopic displays suffer from VAC (conflict between eye focus and convergence), SliceView avoids this because the virtual object and real background are at different optical distances, and the image is a reflection [S1]. However, the sources note that "no VAC occur between the virtual object and the real background" specifically in this configuration [S1].
*   **Black Background Requirement:** If the background of the source image is not pure black, the transmitted light will illuminate the real background, reducing the contrast and the "floating" illusion [S1].
*   **Light Leakage:** In multi-layer systems, light from deeper layers may leak through upper layers, potentially washing out the image. This is mitigated by optimizing layer thickness and optical properties [S2].

## Subtopics

### Transparent Retro-Reflective Surface Display
A 360° volumetric display variant that uses a transparent projection surface made of a sparse cube corner distribution [S1]. It derives from the Pepper's Ghost configuration but replaces the semi-reflective surface with a transparent retro-reflective surface to project a real image [S1].
*   **Definition:** A 360° volumetric display based on a transparent projection surface made of sparse cube corner distribution [S1].
*   **Steps:**
    1.  Project a real image onto a transparent retro-reflective surface (made of sparse cube corners) [S1].
    2.  The surface reflects the image towards the viewer while remaining transparent [S1].
    3.  The image seen through the semi-reflective surface (or directly if configured as aerial) seems to float if surrounded by a black background [S1].
    4.  Occlusion between the bright image and the real background creates volume perception [S1].
*   **Layout:** The transparent projection surface is made of a sparse cube corner distribution [S1].
*   **Comparison:** Compared to classical Pepper's Ghost design, this solution avoids the presence of a display panel above the device and presents a brighter image [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The rendering is essentially 2D image generation, which is standard for display technologies.
*   **Memory Complexity:** Not given in the sources. Depends on the resolution of the display source.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The volume size scales with the number of stacked planes and their physical separation [S2].
*   **Strengths:**
    *   Avoids vergence accommodation conflict (VAC) [S1].
    *   Provides natural occlusion cues [S1].
    *   Simple and cheap compared to complex volumetric solutions [S1].
    *   The transparent retro-reflective variant offers 360° viewing and brighter images without a top display panel [S1].
*   **Limitations:**
    *   Limited to static or temporally multiplexed multi-view perception (video with moving scene) [S1].
    *   Requires a black background for the source image [S1].
    *   Image quality depends on the optical properties (reflection/transmission) and thickness of the layers [S2].
*   **When to use:** When a simple, low-cost volumetric effect is needed for public displays, retro-gaming, or data visualization where 360° viewing or natural occlusion is beneficial [S1, S2].
*   **When not to use:** When high-resolution, full-parallax, or dynamic focus cues are required, or when the environment is too bright to maintain contrast with the projected image [S1].

## Background
The Pepper's Ghost effect is a well-known optical illusion used in theater and entertainment for over a century [S1]. It produces a virtual image from a display by reflection on a semi-reflective surface [S1]. SliceView extends this concept to multiple planes to create a volumetric display [S2]. The transparent retro-reflective surface variant was developed in a laboratory and presented to the public during a symposium, with the optical design detailed in [S1]. SliceView was implemented as a layer-based painting tool, retro-gaming platform, text overlaying media player for subtitles, browser-based 3D scene renderer, and volumetric data visualizer [S2]. User studies compared SliceView with commercial lightfield displays in tasks requiring observation of information at multiple depths [S2].

## Sources
[S1] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
[S2] Volumetric Reach-through Displays for Direct Manipulation of 3D Content, https://doi.org/10.1145/3380867.3427410, fetched 2026-10-26
