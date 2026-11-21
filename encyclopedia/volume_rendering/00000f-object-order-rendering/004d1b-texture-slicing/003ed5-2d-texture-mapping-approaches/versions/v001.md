# 2D Texture Mapping Approaches

Also called: 2D Texture-Based Volume Rendering, Palette Texture Volume Rendering, Cabral Texture Mapping

## Overview
2D Texture Mapping Approaches are object-order volume rendering techniques that decompose a 3D scalar field into a set of 2D slices, storing each slice as a 2D texture, and compositing them back-to-front using hardware alpha blending [S2, S3, S4, S7, unofficial][S19][S20, unofficial]. This family solves the problem of rendering volumetric data on hardware that lacks 3D texture support or to exploit the high performance of 2D texture mapping units [S2, S7, unofficial][S14, S15, S19]. Unlike view-aligned 3D texture methods which require trilinear interpolation and per-frame geometry regeneration, this method uses object-aligned slices and bilinear interpolation, trading mathematical accuracy for compatibility with older graphics hardware and reduced geometry generation overhead [S2, S4, S7, S9, S20, unofficial].

## Shared principles
The 2D Texture Mapping Approaches family shares the following mechanism, terms, and mathematical foundations:

1.  **Axis-Aligned Decomposition:** The volume is decomposed into three stacks of 2D textures, one for each major axis (X, Y, Z). Each stack contains textures representing slices perpendicular to that axis [S2, unofficial][S14, S15, S16, S17, S19][S20, unofficial].
2.  **Texel Format:** The standard texel format is RGBA. The RGB components store the emission coefficient $I_{emit}$ (color), and the Alpha component stores the inverse absorption coefficient $(1 - \vartheta_k)$ [S4, unofficial]. Alternatively, GL_LUMINANCE may store opacity only, requiring multiplication by tissue color in the shader [S20, unofficial].
3.  **View Selection:** The rendering axis is selected based on the viewing direction. The component of the view vector with the largest absolute value determines the axis [S4, unofficial]. The chosen stack's normal must be within $\pm 45^\circ$ of the viewing direction to avoid seeing through slice edges [S7, unofficial][S10, S14, S15, S16, S17].
4.  **Bilinear Interpolation:** Since 2D textures only support bilinear interpolation, the third dimension is not interpolated, leading to potential aliasing and lack of spatial smoothness, especially when zooming [S3, S18, unofficial][S19][S20, unofficial].
5.  **Back-to-Front Compositing:** Slices are sorted back-to-front relative to the camera and composited using the standard alpha blending equation:
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    where $C_{src} = I_{emit}$ and $A_{src} = (1 - \vartheta_k)$ [S4, unofficial]. This implements the discrete volume rendering integral when slices are processed in correct depth order [S3, unofficial].

## Subtopics

### [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md)
Renders volumetric data by decomposing the 3D scalar field into axis-aligned 2D slices stored as 2D textures. It is chosen for compatibility with older graphics hardware lacking 3D texture support. It trades mathematical accuracy (bilinear vs. trilinear interpolation) for performance and compatibility.

### [Palette Texture Volume Rendering](6502-palette-texture-volume-rendering.md)
An acceleration technique for mobile devices with limited memory. It maps 8-bit indexed data to RGBA values via a palette, reducing texture memory footprint to approximately one-fourth of the original size. It solves the problem of slow interactive rendering on low-end hardware by combining adaptive screen-resolution rendering with compressed palette textures.

### [Cabral Texture Mapping](7257-cabral-texture-mapping.md)
Not given in the sources.

## Background
Not given in the sources.

## Sources
