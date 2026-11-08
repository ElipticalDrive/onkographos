# Hardware Volume Rendering Pipeline with Pixel Textures

Also called: Dependent Texture Classification, Dependent Texture Volume Rendering, Hardware Volume Rendering Pipeline (Pixel Texture Based), Hardware-Accelerated Volume Rendering Pipeline with Pixel Textures, Object Volume Rendering with Shading via Three-Dimensional Textures, Pixel Texture Classification, Pixel Texture Shading on SGI V Series, SGI Octane2 Hardware Volume Rendering with Pixel Textures, VGH/QN Pipeline, Voltx

## Overview
This method implements direct volume rendering on hardware supporting 3D texture mapping (e.g., SGI V-series, NVIDIA GeForce3) by using pixel textures (dependent textures) to perform classification and shading lookups [S1][S2][S3, unofficial]. It solves the problem of applying complex, multi-dimensional transfer functions and lighting models efficiently by decoupling data storage from shading computation [S1]. Unlike [3D Texture Proxy Geometry](870-3d-proxy-geometry.md), which rasterizes geometry, this method rasterizes slabs of parallel planes and uses hardware texture interpolation to sample pre-computed color/opacity values [S3, unofficial]. It differs from [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md) by relying on fixed-function pixel texture extensions rather than custom fragment shaders [S1].

## How it works
The pipeline processes a rectilinear volume dataset by converting it into 3D texture maps containing pre-computed color and opacity, then compositing these maps back-to-front [S3, unofficial]. It utilizes separate data and shading volumes to allow independent updates of classification and lighting [S1].

### Inputs and Outputs
*   **Input:** Rectilinear volume data (scalar field), classification parameters (gradient threshold, material type), lighting parameters (direction, ambient intensity), and transfer function parameters.
*   **Output:** A 2D image with alpha blending.

### Data Structures
The method relies on three primary 3D textures:

1.  **Data Volume (VGH):**
    *   **Layout:** 3D texture.
    *   **Fields:**
        *   Red Channel: Data Value ($V$), 8-bit.
        *   Green Channel: Gradient Magnitude ($G$), 8-bit.
        *   Blue Channel: Second Derivative ($H$), 8-bit.
    *   **Note:** Each voxel maps to one texel [S1][S2].

2.  **Quantized Normal Volume (QN):**
    *   **Layout:** 3D texture.
    *   **Fields:**
        *   Red Channel: First 8 bits of 16-bit quantized normal.
        *   Green Channel: Second 8 bits of 16-bit quantized normal.
    *   **Encoding:** Normal direction encoded as a 16-bit unsigned short [S1][S2].

3.  **Shade Volume (Alternative to QN):**
    *   **Layout:** 3D texture.
    *   **Fields:**
        *   Red, Green, Blue Channels: Scaled and biased normal components (one per channel).
    *   **Note:** Used when hardware supports per-fragment dependent texture reads or when QN is not used [S1][S2].

4.  **Lookup Table (LUT):**
    *   **Layout:** Software array.
    *   **Size:** Product of the number of quantized gradients and the number of reflecting materials.
    *   **Content:** Pre-computed RGBA values (color and opacity) for each combination of gradient index and material type [S3, unofficial].

5.  **3D Texture Maps (Color/Opacity):**
    *   **Layout:** 3D texture.
    *   **Content:** Pre-computed RGBA values resulting from the sum of ambient and reflecting components. One texel per volume voxel [S3, unofficial].

### Steps

1.  **Voxel Classification (Offline/Triggered):**
    *   Compute the quantized gradient index and material classification (reflecting or ambient) for each voxel based on a client-supplied gradient-magnitude threshold [S3, unofficial].
    *   Triggered when the transfer function or gradient parameters change [S3, unofficial].

2.  **Lookup Table Creation (Offline/Triggered):**
    *   Create a software LUT mapping each combination of quantized gradient index and reflecting material to a pre-computed color and opacity (RGBA) [S3, unofficial].
    *   The color and opacity account for depth integration through a slab of thickness $\Delta$ [S3, unofficial].
    *   Triggered when volume rotation or light position changes [S3, unofficial].

3.  **Texture Map Filling (Offline/Triggered):**
    *   Fill 3D texture maps with pre-computed color values, one texel per volume voxel. The color is the sum of ambient and reflecting components [S3, unofficial].
    *   For large volumes, partition into multiple texture maps due to memory limits [S3, unofficial].
    *   Triggered when any classification or lighting parameter changes [S3, unofficial].

4.  **Rendering Slices (Per-Frame):**
    *   Process texture maps in back-to-front visibility order of volume partitions [S3, unofficial].
    *   Apply each texture map to many parallel planes orthogonal to screen-z. Render each plane as a square [S3, unofficial].
    *   Specify texture coordinates only at the four corners of the square. Interpolate texture coordinates across the square; out-of-range coordinates at corners interpolate to in-range values precisely when the pixel is within the volume [S3, unofficial].

5.  **Pixel Texture Classification (Per-Slice, SGI V-Series):**
    *   Render the slice from the VGH data volume to a scratch/auxiliary buffer [S1].
    *   Use pixel texture extension to generate texture coordinates from color fragments. Scale fragment channels to $[0,1]$ [S1].
    *   Use these as coordinates into a 3D texture (the Color/Opacity map).
    *   Replace color fragments with entries from the 3D texture using nearest-neighbor or linear interpolation [S1].
    *   *Note:* On SGI V-series, pixel texture does not occur directly on a per-fragment basis during rasterization, requiring the scratch buffer [S1].

6.  **Shading (Per-Slice, Optional):**
    *   If shading is enabled, render a matching slice from the shading volume (QN or Normal) [S1].
    *   Apply pixel texture lookup using the quantized normal texture to the rendered slice in the scratch buffer [S1].
    *   Modulate (multiply) the classified slice with the shaded slice [S1].
    *   *Note:* On NVIDIA GeForce3, dependent texture reads occur per-fragment, eliminating the need for a scratch buffer for shading [S1].

7.  **Compositing:**
    *   Copy the slice from the scratch buffer and blend it with previously rendered slices in the frame buffer [S1].

### Formulas

**Slab Integration (Ambient Component):**
For a slab thickness $\Delta$, let $A_1$ be opacity per unit distance and $E(p)$ be color emission per unit distance for primary color $p$ (Red, Green, Blue). The absorption coefficient $\alpha$ and the resulting color $C(p)$ and opacity $A$ for the texture map entry are:

$$
\alpha = \ln\left(\frac{1}{1-A_1}\right)
$$

$$
C(p) = E(p) \left( \frac{1-e^{-\alpha \Delta}}{\alpha \Delta} \right)
$$

$$
A = 1-e^{-\alpha \Delta}
$$

Where $C(p)$ is well-defined as $\alpha \to 0$ via power series expansion [S3, unofficial].

**Second Derivative Control (Classification):**
The second derivative axis ($H$) is limited to three "sheets" representing negative, zero, and positive values.
*   Opacity on the center sheet (zero second derivative) is directly controlled by classification widgets.
*   Opacity on the outer sheets (positive/negative second derivatives) is scaled from the center sheet opacity according to a boundary emphasis slider [S2].
*   A scale and bias are applied to maintain the sign of the second derivative for proper interpolation [S2].

### Parameters

*   **Quantization Scheme for Normals:** 16-bit [S1].
*   **Interpolation Method:** Nearest-neighbor is used for pixel texture lookup in the SGI implementation [S1].
*   **Slab Thickness ($\Delta$):** Calculated as the total distance covered by the stack of planes divided by the number of planes [S3, unofficial].
*   **Resolution Limits:** Data Value and Gradient Magnitude axes use full 8-bit resolution (256 entries). Second Derivative axis resolution is limited to 3 control points (sheets) [S2].

### Edge Cases and Pitfalls

*   **Scratch Buffer Size:** When updating the shade volume in hardware (SGI V-series), the quantized normal slices must always be smaller than the scratch buffer’s dimensions [S1].
*   **Hardware Flexibility:** Some hardware implementations do not support generating the shading pixel texture by rendering a point with its corresponding normal to the frame buffer and copying it [S1].
*   **Perspective Viewing Error:** For perspective viewing, effective slab thickness increases with off-axis angle. The implementation ignores this, causing up to 4% error for 600-pixel width/70cm distance, and up to 14% for 1000-pixel width. Mitigation: Render concentric spherical shells instead of planes [S3, unofficial].
*   **Texture Size Limits:** The number of elements along a dimension of the pixel texture can differ from the number of bit planes in the component generating the texture coordinate due to hardware scaling/interpolation. This prevents prohibitively large 3D pixel textures [S2].

## Subtopics

### [Multi-Pass/Multi-Texture Shading](5653-multi-pass-multi-texture-shading.md)
Decouples classification and lighting into separate passes to handle complex lighting models like gradient-based enhancement without significant performance penalties.

## Operating characteristics
*   **Time Complexity:**
    *   Lookup Table Size: Proportional to the product of the number of quantized gradients and the number of reflecting materials. Independent of volume size [S3, unofficial].
    *   Texture Map Filling: Proportional to the size of the volume, involving few integer operations per voxel [S3, unofficial].
*   **Memory Complexity:** Proportional to the volume size for texture maps. Large volumes require partitioning into multiple texture maps due to hardware memory limits [S3, unofficial].
*   **Performance:**
    *   Comparable in speed to Shear-Warp (LL) when comparable imaging options are selected [S3, unofficial].
    *   Produces images comparable to ray-casting when voxels do not cover too many pixels [S3, unofficial].
    *   Does not introduce artifacts on rotation, unlike Shear-Warp [S3, unofficial].
    *   Performance impact of lighting/shading extensions is about a factor of ten compared to ambient-only rendering [S3, unofficial].
*   **Strengths:**
    *   High-quality images at nearly interactive speeds on hardware with 3D texture support [S3, unofficial].
    *   Supports orthographic, perspective, and stereo views [S3, unofficial].
    *   Robust to high-frequency or discontinuous transfer functions [S3, unofficial].
*   **Limitations:**
    *   Requires hardware support for 3D textures and pixel textures (dependent textures) [S1][S3, unofficial].
    *   SGI V-series requires scratch buffers and copy operations, reducing performance compared to per-fragment dependent reads (e.g., GeForce3) [S1].
    *   Perspective error if planar slabs are used instead of spherical shells [S3, unofficial].

## Background
The method was implemented on SGI Reality Engine hardware with 3D texture mapping support [S3, unofficial]. The specific pipeline using VGH/QN volumes was implemented on SGI Octane 2 with V-series graphics cards and NVIDIA GeForce3 series [S1]. The Voltx variant was developed at the University of California, Santa Cruz [S3, unofficial].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S3] Object Volume Rendering with Shading via Three-Dimensional Textures, https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
