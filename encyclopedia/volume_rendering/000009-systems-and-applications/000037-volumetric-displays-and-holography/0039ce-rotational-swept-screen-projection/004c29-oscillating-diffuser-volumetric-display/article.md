# Oscillating Diffuser Volumetric Display

Also called: Voxon volumetric display, moving projection surface display, oscillating diffuser display

## Overview
An oscillating diffuser volumetric display is a 360-degree volumetric display that uses a vertically oscillating diffuser screen synchronized with a projection system to create a full volumetric object via retinal persistence [S1]. It is distinguished from rotational swept-screen systems by its linear vertical motion rather than rotation, and from holographic defuser displays by its reliance on simple diffusive scattering rather than light-field generation for occlusion [S1]. While it allows for multi-user viewing from various angles, it is limited to "ghosted content" because the large field of view prevents true occlusion between different parts of the image [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D virtual scene or animation; a synchronization signal from the diffuser actuator or projector.
*   **Output:** A projected 2D image stream on the oscillating diffuser, perceived by the viewer as a 3D volume.

**Data Structures:**
*   **Frame Buffer:** A standard 24-bit RGB image. In specific implementations (e.g., FlexiVol), multiple binary slices are packed into this single frame [S2].
*   **Synchronization Signal:** A periodic signal (e.g., sine wave) driving the actuator, sampled to determine phase [S2].

**Steps:**
1.  **Oscillation:** A diffuser screen is driven to oscillate vertically at a high frequency [S1]. The frequency is chosen such that the human eye cannot perceive the motion of the screen itself, resulting in the screen appearing generally transparent [S1].
2.  **Synchronization:** The projection system updates its image synchronously with the diffuser's position. Synchronization can be achieved by:
    *   Using the actuator signal to trigger the renderer: The renderer samples the actuation signal phase before each frame, extrapolates the diffuser position for the upcoming slices, renders the content, and sends it [S2].
    *   Using the projector as the master clock: The actuator is driven at the same frequency as the projector frame rate, requiring drift correction [S2].
3.  **Projection:** The projector displays 2D slices onto the moving diffuser. As the diffuser moves, the persistence of vision blends these discrete slices into a continuous 3D volume [S1].
4.  **Perception:** The viewer perceives a floating 3D object. Because the diffuser scatters light in a large field of view, multiple viewers can see the image from different angles [S1].

**Parameters:**
*   **Oscillation Frequency:** High enough to prevent flicker and screen visibility [S1]. Not given in the sources.
*   **Frame Rate:** Must match the oscillation frequency or be an integer multiple [S2].

**Pitfalls:**
*   **Occlusion:** True volumetric occlusion is not possible because the image is diffused over a large field of view. The display is limited to "ghosted content" where objects do not block each other [S1].
*   **Synchronization Drift:** If the actuator and projector are not perfectly phase-locked, drift occurs, causing Z-depth distortion or blurring [S2].

## Subtopics

### FlexiVol Rendering Pipeline
A rendering method for a swept volumetric display using an elastic diffuser [S2]. It differs from standard implementations by packing multiple binary slices into a single color frame to increase data throughput or simplify transmission [S2].
*   **Definition:** A virtual scene is sliced along planes, and these slices are projected at high speed onto the oscillating diffuser [S2].
*   **Steps:**
    1.  Slice the virtual scene along planes [S2].
    2.  Combine several slices into a single color image [S2].
    3.  Send the combined color image to the projector via HDMI or Display Port [S2].
    4.  Project the slices at high speed onto the oscillating diffuser [S2].
*   **Parameters:**
    *   Number of slices packed per frame: 24 [S2].
    *   Color depth per slice: Binary (1-bit per channel implied by 24 slices packed into 24-bit RGB) [S2].
*   **Layout:** A regular 24-bit RGB image where 24 binary slices are packed [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of slices rendered per frame. Synchronization overhead depends on the sampling rate of the actuator signal [S2].
*   **Memory Complexity:** Low, as it only requires storing the current frame buffer and scene data [S2].
*   **Performance:** Efficient for displaying animated objects or figures [S1]. The high oscillating frequency precludes the diffuser from being seen, leading to general transparency [S1].
*   **Strengths:** 360-degree viewing; multi-user capability; simple optical setup compared to holographic methods [S1].
*   **Limitations:** No true occlusion between objects; limited to "ghosted content" [S1].
*   **When to use:** When multi-user 360-degree visibility is required and true occlusion is not critical [S1].
*   **When not to use:** When true volumetric occlusion is required [S1].

## Background
The concept was developed to allow efficient 360-degree display of volumetric objects, particularly for animated figures, enabling group viewing [S1]. The commercial product Voxon is a notable example of this technology [S1]. Research by Jones et al. proposed using a holographic rotating diffuser to address the occlusion limitation, generating 288 images on a 360° horizontal field of view [S1]. More recent research has explored elastic diffusers (e.g., elastane, latex, silicone) to improve mechanical and optical properties [S2].

## Sources
[S1] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
[S2] Direct Interaction with a Reach-through Volumetric Display that uses an Elastic Diffuser, https://doi.org/10.1145/3772363.3799374, fetched 2026-10-29
