# Upsampling and Super-Resolution

Also called: 3D Image Resize and Crop, Cubic Upsampling via Tiling, Cloud Upscaling, Upsampling Layer, Epipolar Overlap Visualization, Pre-processing for Double Resolution

## Overview
Upsampling and Super-Resolution methods in volume rendering address the problem of increasing the spatial resolution of volumetric data or rendered images to recover fine details, correct discrete positioning errors, or optimize rendering performance. This family is distinguished from sibling multiresolution techniques (which often focus on downsampling or LOD selection for culling) by its primary goal of generating higher-fidelity representations from lower-resolution inputs, either through geometric remapping, interpolation, or learning-based refinement.

## Shared principles
The core mechanism across these methods involves expanding the spatial domain of a volume or image. Common mathematical operations include:
1.  **Scaling Factors:** Defining an integer or float scale factor $s$ such that $N_{new} = s \times N_{prev}$ or $N_{new} / N_{prev} = s$ [S1, unofficial][S5].
2.  **Interpolation:** Using bilinear or bicubic kernels to estimate values at new grid points [S1, S3, unofficial].
3.  **Geometric Remapping:** Mapping existing voxel coordinates $(i, j, k)$ to new coordinates in a finer grid, often to align with sub-pixel precision [S5].
4.  **Normalization:** Scaling values to a standard range (e.g., $[0, 1]$) by dividing by the global maximum to ensure consistent input for subsequent processing [S1, unofficial].

## Subtopics

### [3D Image Resize and Crop](2703-3d-image-resize-and-crop.md)
This method prepares 3D volumetric data for super-resolution tasks by generating paired high-resolution and low-resolution inputs [2703-3d-image-resize-and-crop]. It solves the problem of spatial misalignment and aspect ratio mismatch in 3D datasets by centering the volume along the Z-axis and squaring the X-Y plane via zero-padding [2703-3d-image-resize-and-crop]. It differs from [3D Fast Bilateral Solver](6478-3d-fast-bilateral-solver.md) by operating in the spatial domain using direct interpolation and normalization rather than solving optimization problems for edge-preserving smoothing [2703-3d-image-resize-and-crop].

### [3D Fast Bilateral Solver](6478-3d-fast-bilateral-solver.md)
A 3D extension to the Fast Bilateral Solver [12] used to refine annotated similarity volumes [6478-3d-fast-bilateral-solver]. It increases the resolution of similarity maps and adapts them to the underlying intensity volume to improve visual quality [6478-3d-fast-bilateral-solver].

### Cubic Upsampling via Tiling
A method for increasing resolution by tiling [S1, unofficial].
- **Step:** Calculate the integer scale factor as $N_{new} / N_{prev}$ [S1, unofficial]. Reshape the input 1D values array into a 3D cube of shape $(N_{prev}, N_{prev}, N_{prev})$ [S1, unofficial]. Tile the 3D cube along dimension 2 (z-axis) by the scale factor [S1, unofficial]. Tile the result along dimension 1 (y-axis) by the scale factor [S1, unofficial]. Tile the result along dimension 0 (x-axis) by the scale factor [S1, unofficial]. Reshape the tiled 3D array back into a 1D array [S1, unofficial].

### Cloud Upscaling
Rendering clouds at a lower resolution and upscaling to screen resolution, leveraging their naturally blurry and fluffy appearance [S2, unofficial]. This approach exploits the perceptual tolerance for blur in volumetric cloud rendering to reduce computational cost while maintaining visual fidelity [S2, unofficial].

### Upsampling Layer
A layer that expands the spatial dimensions $x$ and $y$ of an input volume $U$ of size $x \times y \times p$ by factors $s_x$ and $s_y$ respectively to generate a volume $V$ of size $(s_x x) \times (s_y y) \times p$ [S3, unofficial]. It contains no trainable parameters and is used in the decoder to reconstruct original size [S3, unofficial].
- **Parameters:** Scale Factors: $s_x=2, s_y=2$. Filter Size: $2 \times 2$. Stride: 2. Applied in decoder layers [S3, unofficial].

### Epipolar Overlap Visualization
A visualization technique involving upsampled valid masks [S4, unofficial].
- **Step:** Extract `valid` mask from sampling data [S4, unofficial]. Upsample `valid` mask to match context image resolution using `repeat` with $r_h=ds, r_w=ds$ [S4, unofficial]. Blend valid mask with context images: `valid = valid * (green + context_images[index]) / 2` [S4, unofficial]. Concatenate context images and valid masks [S4, unofficial].
- **Parameter:** `ds`: Downscale factor for epipolar transformer, typically integer > 1 [S4, unofficial].

### Pre-processing for Double Resolution
A method to increase resolution by mapping full-resolution voxels to a double-resolution volume and filling gaps to provide more details and accurate voxel positions [S5]. Double resolution allows existing voxels to be located at the middle point between two voxels of the full-resolution volume, correcting for discrete domain positioning errors where non-corresponding pixels might map to the same voxel [S5].
- **Step:** Map the voxel at location $(i, j, k)$ of the full-resolution volume to location $(2i-1, 2j-1, 2k-1)$ of the double-resolution volume [S5]. Inspect other voxels to determine if they are part of the volume or transparent by registering depth maps of references into the double-resolution volume [S5]. Iteratively refine the constructed volume utilizing brightness information of the reference views [S5]. Properly select the colour of each voxel to render a smooth and realistic image [S5].

## Background
The concept of upsampling in volumetric data is rooted in the need to mitigate aliasing and improve the perceptual quality of synthesized views. Early work in colour volumetric compression highlighted that discrete domain positioning errors lead to poor subjective quality if the representing volume is not fine enough, motivating double-resolution pre-processing [S5]. In the context of deep learning for volume representation, upsampling layers are standard components of autoencoder decoders to reconstruct high-resolution features from compressed latent spaces [S3, unofficial]. More recently, specific implementations for 3D image resizing and cropping have been developed to handle aspect ratio mismatches in datasets used for neural rendering [S1, unofficial]. Cloud rendering techniques have also adopted upscaling as a performance optimization, relying on the inherent blur of volumetric clouds to mask interpolation artifacts [S2, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S3] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S4] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
[S5] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
