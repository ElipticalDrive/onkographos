# MAB Compositing with Gaze-Contingent Blending

Also called: MAB compositing, Multi-Anchor Buffer compositing

## Overview
MAB compositing is a gaze-contingent rendering technique that maps framebuffer fragments to a Multi-Anchor Buffer (MAB) to retrieve and blend color values from different resolution levels based on the distance to the user's gaze [S1]. It solves the problem of maintaining high frame rates in immersive VR environments by dynamically adjusting the rendering resolution distribution in real-time, thereby preventing performance fluctuations that cause cybersickness [S1]. Unlike standard fixed-resolution compositing, it blends multiple resolution levels simultaneously using smooth-step transitions to ensure unnoticeable visual artifacts between detail levels [S1].

## How it works
### Inputs and Outputs
*   **Inputs:**
    *   Screen-space coordinates of the fragment $(u, v)$ [S1].
    *   Gaze center coordinates $(u_g, v_g)$ in screen space [S1].
    *   The MAB, which stores color values at multiple resolution levels $L$ [S1].
    *   A 1D lookup texture implementing function $f_a$ [S1].
*   **Outputs:**
    *   Final composited color $C_{final}$ for each pixel in the output framebuffer [S1].

### Data Structures
*   **Multi-Anchor Buffer (MAB):** A buffer structure containing multiple resolution levels $L_0, L_1, \dots, L_n$ [S1]. Each level stores color data corresponding to a specific spatial resolution [S1].
*   **Blending Texture ($f_a$):** A 1D lookup texture where the four color channels (R, G, B, A) map to the blending factors for the MAB's resolution levels [S1]. Each channel's value represents the opacity or weight for a specific resolution level based on the input gaze distance [S1].

### Algorithm Steps

#### Phase 1: MAB Generation (Raycasting)
1.  Render a full-screen primitive covering the entire MAB to generate fragments for all resolution levels [S1].
2.  For each generated MAB fragment, determine its coordinates and resolution level [S1].
3.  Use function $f_c$ to map the MAB coordinates to the resolution level $L$ of the fragment and the corresponding position $(u_{fb}, v_{fb})$ in the output framebuffer [S1].
4.  Calculate the gaze distance $d$ using the framebuffer position $(u_{fb}, v_{fb})$ and the gaze center $(u_g, v_g)$ [S1].
5.  Evaluate function $f_a$ using the distance $d$ and the resolution level $L$ to retrieve the blending factor (opacity) for this level [S1].
6.  If the blending factor is zero, discard the fragment [S1]. Otherwise, store the color value in the MAB at the appropriate resolution level [S1].

#### Phase 2: Final Compositing
1.  Render a full-screen primitive covering the output framebuffer to initiate compositing for each generated fragment [S1].
2.  For each fragment, compute the distance $d$ to the center of gaze based on its screen-space coordinates $(u, v)$ and the gaze center $(u_g, v_g)$ [S1].
3.  Evaluate function $f_a$ using the gaze distance $d$ to output a set of blending factors $\{w_0, w_1, \dots, w_n\}$ for the MAB resolution levels [S1].
4.  Evaluate function $f_b$ using the fragment's screen-space coordinates $(u, v)$ to output a set of corresponding coordinates $\{(u_0, v_0), (u_1, v_1), \dots, (u_n, v_n)\}$ in the MAB [S1].
5.  Initialize the accumulated color $C_{acc} = 0$ [G].
6.  For each resolution level $i$ where the blending factor $w_i > 0$:
    *   Perform a lookup into the MAB at coordinates $(u_i, v_i)$ to obtain the stored color value $C_i$ [S1].
    *   Blend the obtained color values together into a single color for display [S1].
7.  Output $C_{acc}$ as the final pixel color [S1].

### Functions and Formulas
**Gaze Distance:**
$$ d = \sqrt{(u - u_g)^2 + (v - v_g)^2} $$
Where $(u, v)$ is the fragment's screen position and $(u_g, v_g)$ is the gaze center [G].

**Function $f_a$ (Blending Factors):**
Implemented as a 1D lookup texture [S1].
$$ \{w_0, w_1, \dots, w_n\} = f_a(d) $$
The texture maps the distance $d$ to the four color channels, where each channel corresponds to a resolution level's blending factor [S1]. To ensure unnoticeable blending between adjacent resolution levels, a smooth-step function is integrated into the texture values [S1].

**Function $f_b$ (Coordinate Mapping):**
$$ \{(u_0, v_0), \dots, (u_n, v_n)\} = f_b(u, v) $$
Maps screen-space coordinates to the corresponding coordinates in each resolution level of the MAB [S1].

**Function $f_c$ (Inverse Mapping for MAB Generation):**
$$ (L, u_{fb}, v_{fb}) = f_c(u_{mab}, v_{mab}) $$
Takes MAB coordinates and outputs the resolution level $L$ and the corresponding framebuffer position [S1].

### Parameters
*   **$f_a$ Implementation:** A 1D lookup texture similar to transfer functions in direct volume rendering [S1].
*   **Smooth-step:** Added between adjacent resolution levels in $f_a$ to prevent visible seams [S1].
*   **Frame Rate Threshold:** The system adjusts the distribution of resolution levels on the fly to maintain a refresh rate above a set threshold [S1].

### Edge Cases and Pitfalls
*   **Zero Blending Factors:** Fragments in the MAB generation phase with a blending factor of zero must be discarded to avoid unnecessary computation and memory writes [S1].
*   **Seamless Blending:** Failure to use a smooth-step function in $f_a$ results in noticeable artifacts between adjacent resolution levels [S1].
*   **VR Performance Fluctuations:** In VR setups, camera position and viewing direction fluctuations can cause severe performance variations [S1]. The system must adjust resolution distribution in real-time to prevent cybersickness [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of resolution levels and the cost of texture lookups [S1]. The method requires two full-screen passes (MAB generation and compositing) plus multiple MAB lookups per pixel during compositing [S1].
*   **Memory Complexity:** Requires storage for the Multi-Anchor Buffer, which contains multiple resolution levels of the scene [S1].
*   **Strengths:** Provides straightforward real-time adjustment of rendering resolution to maintain high frame rates in VR, reducing cybersickness [S1].
*   **Limitations:** Performance depends on the efficiency of the texture lookups and the complexity of the blending functions [S1]. Changing transfer functions or views can significantly impact performance [S1].
*   **When to use:** Immersive VR applications where maintaining a consistent high frame rate is critical for user comfort, and where gaze-contingent detail is beneficial [S1].
*   **When not to use:** Non-VR applications where gaze tracking is unavailable or where fixed high resolution is required without dynamic adjustment [S1].

## Background
The method was presented in the paper "Embracing Raycasting for Virtual Reality" at WSCG 2022 [S1]. It addresses the challenges of direct volume raycasting in VR, where performance stability is crucial to prevent cybersickness caused by frame rate fluctuations [S1]. The compositing phase maps fragments to the MAB based on gaze distance [S1]. The system separately tracks rendering performance and adjusts the resolution levels' distribution on the fly to guarantee refresh rates above a set threshold [S1].

## Sources
[S1] Embracing Raycasting for Virtual Reality, https://doi.org/10.24132/csrn.3201.16, fetched 2026-10-29
