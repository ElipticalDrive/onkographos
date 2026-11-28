# Combining Lightening and Darkening

Also called: Lightening and Darkening Combination, Transfer Function (Brightness and Darkness)

## Overview
This method customizes the appearance of volumetric rendering by applying linear brightness and darkness adjustments to input colors, or by combining separately computed lightened and darkened CIELab colors [S1, unofficial][S2]. It differs from sibling methods like [Windowing](6611-windowing.md) by operating directly on color intensity values rather than thresholding scalar data, and from [Color Composition](13205-color-composition.md) by using a specific linear transform or arithmetic mean of luminance-modified colors rather than general blending [S1, unofficial][S2].

## How it works
**Inputs:**
1.  Input color $C$ (RGB or CIELab).
2.  Brightness coefficient $B$.
3.  Darkness subtrahend $D$.
4.  (Optional) Lightened color $C_L$ and Darkened color $C_D$ in CIELab space.

**Outputs:**
1.  Output color $C_{out}$ (RGB).

**Data Structures:**
Not given in the sources.

**Steps:**

**Method 1: Linear Brightness/Darkness Transform**
1.  Let input color be $C_{in}$ (RGB).
2.  Calculate output color using the linear function:
    $$ C_{out} = B \times (C_{in} - D) $$
    Where $D$ is the Darkness (subtrahend) and $B$ is the Brightness (coefficient) [S2].
3.  If $B > 1$, the resulting RGB values are brighter than the input [S2].

**Method 2: CIELab Lightening and Darkening Combination**
1.  Convert input color $C$ to CIELab space [S1, unofficial].
2.  Compute lightened color $C_L$ by unsharpening luminance in CIELab space [S1, unofficial].
3.  Compute darkened color $C_D$ by unsharpening luminance in CIELab space [S1, unofficial].
    *   Note: For darkening, levels 2 or 3 of the hierarchy are used to avoid unpleasant shadowed regions produced by lower resolution levels [S1, unofficial].
4.  Transform $C_L$ from CIELab to RGB: $RGB(C_L)$ [S1, unofficial].
5.  Transform $C_D$ from CIELab to RGB: $RGB(C_D)$ [S1, unofficial].
6.  Combine the colors:
    $$ C_{sample} = 0.5 \times (RGB(C_L) + RGB(C_D)) $$
    [S1, unofficial]

**Parameters:**
*   Brightness ($B$): Coefficient. Typical values not given in the sources.
*   Darkness ($D$): Subtrahend. Typical values not given in the sources.
*   Hierarchy Level for Darkening: 2 or 3 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Using lower resolution levels for darkening produces extra, unpleasant shadowed regions [S1, unofficial].
*   Opacity of the sample remains untouched during the lightening/darkening process [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method is described as one of the very basic methods to customize the look of volumetric rendering using transfer functions [S2]. It is implemented in software such as LightningChart .NET [S2]. The CIELab-based combination approach is part of research into enhanced perception in volume visualization, specifically for emphasizing local features by modifying luminance while maintaining saturation [S1, unofficial].

## Subtopics

### Linear Brightness/Darkness Transform
A simple linear adjustment of RGB values using a coefficient and subtrahend. Choose this for basic intensity scaling. [S2]

### CIELab Lightening and Darkening Combination
Combines separately computed lightened and darkened colors in CIELab space. Choose this for perceptually uniform luminance adjustments that preserve saturation. [S1, unofficial]

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] VolumeModel | LightningChart .NET Docs, https://lightningchart.com/net-charts/docs/inDetail/volumetric-rendering/, fetched 2026-10-29
