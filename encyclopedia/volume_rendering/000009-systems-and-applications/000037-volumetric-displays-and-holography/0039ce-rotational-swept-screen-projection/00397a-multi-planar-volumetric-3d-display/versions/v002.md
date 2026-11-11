# Multi-planar volumetric 3D display

Also called: Moving screen volumetric display, Swept-screen volumometric display, Temporal integration volumetric display

## Overview
A multi-planar volumetric 3D display projects a sequence of 2D image patterns onto a screen moving along the Z-axis to form a stack of spatial image planes, creating a true 3D image viewable without glasses [S1]. Unlike conventional 2D screens that rely on psychological depth cues, this method places voxels at their physical spatial positions, emitting light omni-directionally to provide both physiological and psychological depth cues [S1]. It distinguishes itself from sibling methods like the Volumetric Display using Rotating Mirror (VRDM) by utilizing linear or reciprocating screen motion rather than rotational sweep [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume defined by $N$ 2D image frames (slices) corresponding to specific Z-depths, and a screen sweep frequency $f_{sweep}$.
*   **Output:** A continuous 3D volumetric image perceived by the human eye.

**Data Structures:**
*   **Image Sequence:** A list of $N$ 2D frames $\{I_0, I_1, ..., I_{N-1}\}$, where each frame $I_k$ corresponds to the image pattern projected when the screen is at position $z_k$.
*   **Screen State:** Defined by position $z(t)$ and velocity $v(t)$ along the Z-axis.

**Steps:**
1.  **Initialize Sweep:** The screen begins moving along the Z-axis. The motion is typically back-and-forth (reciprocating) or continuous linear, with a frequency $f_{sweep} > 24$ Hz [S1].
2.  **Synchronize Projection:** A high-speed 2D image projector (e.g., DLP-based) updates its output synchronously with the screen's position [S1].
3.  **Project Frames:** During one complete sweep cycle (period $T = 1/f_{sweep}$), the projector displays $N$ distinct 2D image frames [S1].
4.  **Intercept and Stack:** The moving screen intercepts the projected light at different Z-positions. Each frame $I_k$ is projected when the screen is at the corresponding depth $z_k$, effectively "painting" a slice onto the screen surface [S1].
5.  **Temporal Integration:** Due to the persistence of vision and the high sweep frequency, the discrete 2D slices blend into a continuous 3D volume in the viewer's perception [S1].

**Formulas:**
*   **Sweep Frequency:** $f_{sweep} > 24$ Hz [S1].
*   **Frame Rate:** The projector must be capable of displaying $N$ frames within the time $T = 1/f_{sweep}$. Thus, the required projector frame rate $R_{proj}$ is:
    $$ R_{proj} = N \cdot f_{sweep} $$
*   **Z-Position Mapping:** For a sweep range $[Z_{min}, Z_{max}]$, the position $z_k$ for frame $k$ (where $k \in [0, N-1]$) is mapped linearly or according to the screen's kinematic profile.

**Parameters:**
*   **$f_{sweep}$:** Screen sweep frequency. Must be $> 24$ Hz to avoid flicker and ensure proper temporal integration [S1].
*   **$N$:** Number of 2D image frames projected per sweep cycle [S1].
*   **Projector Type:** High-speed 2D image projector (e.g., DLP) is preferred for high frame rates and quality [S1].

**Edge Cases and Pitfalls:**
*   **Flicker:** If $f_{sweep} \le 24$ Hz, the display may flicker, breaking the 3D illusion [S1].
*   **Motion Blur:** Excessive screen velocity relative to projector exposure time can cause blurring of the slices.
*   **Synchronization Error:** Misalignment between screen position and projected frame index results in Z-depth distortion [S1].

## Subtopics

### [Line Patterns and 2D/3D Projection](10357-line-patterns-and-2d-3d-projection.md)
A signal modulation technique that projects variable 1D patterns along the viewing line through the display volume by timing laser pulses. It provides depth context without the occlusion of full 3D shapes, serving as the 3D analog to the 0D dot used on 2D screens.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The system relies on high-speed DLP projection engines to achieve the necessary frame rates [S1].
*   **Strengths:** Displays voxels physically at their spatial positions, emitting light omni-directionally, providing true physiological depth cues [S1].
*   **Limitations:** Requires precise synchronization between screen motion and projection [S1]. Limited by the mechanical speed of the screen and the frame rate of the projector.
*   **When to use:** When true 3D spatial representation with physiological depth cues is required, and glasses-free viewing is necessary [S1].
*   **When not to use:** When high resolution is required without mechanical complexity, or when the scene requires static 3D display without motion.

## Background
The multi-planar volumetric 3D display technique was developed to address the limitations of conventional 2D screens, which lack physical depth cues [S1]. It utilizes high-speed DLP projection engines to facilitate the rapid projection of 2D image patterns onto a moving screen [S1]. The method was highlighted in research focusing on the design of volumetric 3D displays based on high-speed DLP projection engines [S1].

## Sources
[S1] A volumetric 3D display based on a DLP projection engine, https://doi.org/10.1016/j.displa.2012.11.001, fetched 2026-10-27
