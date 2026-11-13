# Ray Marching Jittering

Also called: Dithering Offset, Jittered Ray Marching, Randomized Step Size Ray Marching

## Overview
Ray marching jittering is a technique to reduce aliasing artifacts (specifically banding) in volume rendering by introducing random variation to the ray marching step sizes [S2, unofficial]. It addresses the visual defect of clearly defined intensity bands caused by fixed step sizes, while preserving the sharpness of volumetric effects like god rays [S2, unofficial]. It builds on [Ray Marching Dither](3269-ray-marching-dither.md) by applying perturbation to the step magnitude rather than just UV coordinates.

## How it works
**Inputs and Outputs:**
*   Input: Current ray position $t$, fixed step size $s$, dithering strength parameter $d$ (typically $>0$), and a source of randomness (e.g., hash of texture coordinates).
*   Output: A jittered step size or offset used to advance the ray.

**Data Structures:**
*   No specific complex data structures are required; the method operates on scalar values per ray or per sample.

**Steps:**
1.  **Calculate Jitter Offset:** Determine a random offset based on the dithering strength and a hash of the current pixel or texture coordinates. If dithering is disabled ($d \le 0$), the offset is zero [S1, unofficial].
    $$ \text{jitter} = \begin{cases} \text{hash}(v_{\text{tex\_coord}}) \cdot s \cdot d & \text{if } d > 0 \\ 0.0 & \text{otherwise} \end{cases} $$
    Where $\text{hash}(\cdot)$ returns a value in $[0, 1)$ [S1, unofficial].
2.  **Determine Step Size:** The ray marches a random distance between $0.5s$ and $1.5s$ [S2, unofficial]. This is often implemented by adding the jitter offset to the base step or selecting a random multiplier in $[0.5, 1.5]$.
3.  **Advance Ray:** Update the ray position $t$ using the jittered step. For example, if using an offset approach:
    $$ t_{\text{new}} = t_{\text{old}} + s + \text{jitter} $$
    Or, as described in the source, setting a start time for the next segment:
    $$ t_{\text{start}} = t_{\text{min}} + \text{jitter} $$
    [S1, unofficial].

**Parameters:**
*   $d$ (dithering strength): Controls the magnitude of the jitter. Must be $>0$ to activate [S1, unofficial].
*   Step size multiplier range: $[0.5, 1.5]$ [S2, unofficial].

**Pitfalls:**
*   **Banding:** Fixed step sizes introduce aliasing in the form of clearly defined bands of intensity in the volume region [S2, unofficial].
*   **Sharpness:** Unlike some other anti-aliasing methods, jittering is chosen specifically because it keeps god rays sharp and focused while removing banding [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step addition. The cost is negligible compared to the volume lookup.
*   **Memory Complexity:** $O(1)$ additional memory per ray/sample.
*   **Strengths:** Effectively removes banding artifacts in participating media without blurring sharp volumetric features like god rays [S2, unofficial].
*   **Limitations:** Does not address other forms of aliasing (e.g., spatial undersampling of the volume geometry itself). Relies on the randomness source being sufficiently uniform.
*   **When to use:** When rendering participating media with fixed step sizes and observing banding artifacts.
*   **When not to use:** When strict deterministic reproducibility is required without accounting for the random seed, or when other aliasing sources dominate.

## Background
The technique was documented in the context of rendering underwater scenes and participating media, where long render times and visual artifacts like banding were significant issues [S2, unofficial]. It was applied to both photon gathering and lookup stages to improve visual quality [S2, unofficial].

## Sources
[S1] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S2] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
