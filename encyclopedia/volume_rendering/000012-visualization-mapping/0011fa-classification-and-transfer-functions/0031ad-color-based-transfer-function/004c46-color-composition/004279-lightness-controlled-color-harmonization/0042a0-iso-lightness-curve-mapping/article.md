# Iso-lightness Curve Mapping
## Overview
Iso-lightness curve mapping is a technique that uses iso-lightness curves derived from the CIE LAB color space, mapped onto HSV hue slices, to determine the correct saturation/lightness trade-off when shifting hues in volume visualization [S1]. It addresses the limitation of hue-only harmonization by preserving existing lightness contrasts and salience, which are critical for highlighting and contrast enhancement [S1]. Unlike standard HSV manipulation, this method accounts for the non-perceptual nature of HSV by grounding color adjustments in the perceptually uniform CIE LAB space [S1].

## How it works
The method maps transfer function parameters from HSV to CIE LAB to ensure that hue changes do not inadvertently alter the perceived lightness of the volume data [S1].

**Inputs and Outputs:**
*   **Input:** A source HSV triple $(H_s, S_s, L_s)$ representing a color in the transfer function; a target hue $H_t$ for harmonization [S1].
*   **Output:** A target HSV triple $(H_t, S_t, L_t)$ that maintains the source lightness $L_s$ while adopting the target hue [S1].

**Data Structures:**
*   **Iso-lightness Curves:** A set of pre-computed trajectories in CIE LAB space [S1]. Each curve corresponds to a specific hue slice in HSV space [S1]. For a given hue in HSV, the trajectory in CIE LAB space maintains constant lightness while varying chroma [S1].
*   **Monitor Profile:** Required for accurate sRGB conversion, specifically the monitor's white point [S1].

**Steps:**
1.  **Derive Iso-lightness Curves:**
    *   Convert the monitor's sRGB color space to CIE XYZ using the monitor's specific white point [S1].
    *   Convert CIE XYZ to CIE LAB [S1].
    *   For a given hue in HSV, identify the trajectory in CIE LAB space that maintains constant lightness while varying chroma [S1]. These trajectories form the iso-lightness curves [S1]. Note that these curves are monitor-specific [S1].
2.  **Map Source Color to LAB:**
    *   Convert the source HSV triple to sRGB, then to CIE XYZ using the monitor's white point, and finally to CIE LAB $(L_s, a_s, b_s)$ [S1].
3.  **Locate Target Color:**
    *   Determine the target hue $H_t$ [S1].
    *   In the CIE LAB space, locate the point that has the target hue and the source lightness $L_s$ [S1].
    *   This involves finding the intersection of the iso-lightness curve corresponding to the target hue with the lightness plane defined by $L_s$ [S1].
4.  **Handle Saturation/Lightness Trade-off:**
    *   If the target hue's iso-lightness curve does not allow the desired saturation at the source lightness, the method identifies the closest valid point on the curve [S1].
    *   The resulting LAB triple is converted back to HSV to yield the final harmonized color [G].

**Parameters:**
*   **Monitor White Point:** Essential for the sRGB to XYZ conversion [S1].
*   **Tolerance Levels:** CIE LAB is perceptually uniform within certain tolerance levels [S1].

**Edge Cases and Pitfalls:**
*   **Monitor Specificity:** The iso-lightness curves are monitor-specific [S1].
*   **Impossible Lightness:** It is not always possible to increase lightness without sacrificing saturation [S1].
*   **HSV Non-Perceptual Nature:** HSV is not a perceptual color space [S1].

## Subtopics

### Equi-lightness color scales for highlighting
This variant uses equi-lightness curves to design color scales where vividness (chroma) changes from high to low while keeping lightness constant [S1]. By applying such a scale, specific features in volume data can be highlighted, drawing the observer's attention without altering the lightness-based contrast of other features [S1]. This leverages the "pop-out" effect of vivid colors while maintaining the structural integrity provided by lightness [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Preserves lightness contrasts, which are crucial for highlighting, contrast enhancement, and pop-out in volume visualization [S1]. Allows for harmonic colorization without losing salience [S1].
*   **Limitations:** Monitor-specific, requiring accurate white point calibration [S1]. CIE LAB is only perceptually uniform within certain tolerance levels [S1].
*   **When to use:** When harmonic colorization is desired but lightness-based features (like highlighting) must be preserved [S1].
*   **When not to use:** When monitor-specific calibration is unavailable [S1].

## Background
The method was introduced by Wang and Mueller in 2008 as part of their work on harmonic colormaps for volume visualization [S1]. It addresses a significant shortcoming of previous color harmonization frameworks, which did not preserve lightness [S1]. The technique builds on the observation that HSV is intuitive but inappropriate for lightness preservation, while CIE LAB is perceptually uniform [S1].

## Sources
[S1] Harmonic Colormaps for Volume Visualization, https://doi.org/10.2312/vg/vg-pbg08/033-039, fetched 2026-10-26
