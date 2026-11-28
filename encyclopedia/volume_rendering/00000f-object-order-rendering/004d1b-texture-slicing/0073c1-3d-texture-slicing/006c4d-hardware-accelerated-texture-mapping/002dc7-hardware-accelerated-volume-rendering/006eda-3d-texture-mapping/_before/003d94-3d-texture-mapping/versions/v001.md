# 3D Texture Mapping

Also called: Hardware-Accelerated Volume Rendering via 3D Texturing, 3D Pixel Texture Classification, Hardware Shading via Pixel Texture

## Overview
3D Texture Mapping is a family of hardware-accelerated volume rendering techniques that leverage the GPU's texture mapping hardware to perform volumetric sampling, classification, and shading. It solves the performance bottleneck of software-based ray casting by offloading trilinear interpolation and transfer function evaluation to dedicated graphics pipelines [S1][S2, unofficial]. Unlike [Direct volume ray casting](19-direct-volume-ray-casting.md) which typically uses fragment shaders for all computations, this family relies on fixed-function or early programmable texture units to resample data into textured planes or lookup tables, often compositing directly into the frame buffer [S1][S2, unofficial].

## Shared principles
The family shares the mechanism of storing volumetric data in 3D texture memory and using the texture engine's hardware interpolation capabilities. The core pipeline for each pixel consists of ray generation, hardware-accelerated sampling, classification, and compositing [S1, unofficial][S2][S4, unofficial].

### Ray Generation
Rays are generated per-pixel. This is typically achieved by rendering the volume's bounding box geometry (front and back faces) to determine entry and exit points, or by computing ray parameters directly from camera matrices and projection models (e.g., MCOP) in the vertex/fragment shader [S1, unofficial][S2][S3, S4, unofficial]. The ray is defined as $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the direction [Parent].

### Hardware-Accelerated Sampling
Scalar volume values $V(\mathbf{p})$ are sampled using the GPU's texture hardware. The volume data is stored in 3D texture memory, and the texture engine performs trilinear interpolation of the eight surrounding voxels in parallel [S1, unofficial][S2][S3, S4, unofficial]. This avoids the computational cost of manual interpolation in shader code [S2].

### Fragment Shader Compositing
The core ray marching loop, including classification (transfer function application) and front-to-back compositing, is executed within the fragment shader [S1, unofficial][S2][S3, S4, unofficial]. The standard compositing equation (over operator) updates the accumulated color $C'$ and accumulated opacity $\alpha'$ for each sample $i$:

$$
C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
$$
$$
\alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
$$

where $C'_0 = 0$ and $\alpha'_0 = 0$ [Parent].

### Early Ray Termination (ERT)
To optimize performance, rays are terminated when the accumulated opacity $\alpha'$ exceeds a threshold (typically 0.95 or 1.0), as subsequent samples are fully occluded [S1, unofficial][S2][S3, S4, unofficial].

## Subtopics

### [Hardware-Accelerated Volume Rendering via 3D Texturing](2799-hardware-accelerated-volume-rendering-via-3d-texturing.md)
Uses 3D texture mapping to resample volumetric data into textured planes parallel to the image plane, then composites these slices into the frame buffer. It differs from [Shear-Warp](9530-shear-warp.md) by performing trilinear interpolation followed by classification without a final warp step, and from [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md) by using explicit geometry for resampling planes.

### [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md)
Maps multi-dimensional voxel attributes (Data Value, Gradient Magnitude, Second Derivative) to color and opacity using a 3D texture lookup. It sets itself apart by structuring the lookup texture around specific derivative axes to emphasize boundaries, rather than relying on generic scalar textures.

### [Hardware Shading via Pixel Texture](6056-hardware-shading-via-pixel-texture.md)
Adds surface shading to texture-memory based volume rendering by encoding surface normals into a 2D pixel texture using 16-bit quantization and nearest-neighbor interpolation. It targets illumination quality with negligible performance cost, distinct from classification-focused methods.

## Background
The method originated in the mid-1990s with the introduction of 3D texture mapping capabilities in graphics hardware, such as the SGI RealityEngine [S1][S2, unofficial]. It was developed to address the need for interactive volume visualization by exploiting the parallel nature of GPUs for resampling and compositing [S4]. Early implementations focused on direct summation and radiograph rendering, later evolving to include opacity-based rendering and hardware shading [S2, unofficial].

## Sources
[S1] 3D Texture Mapping for Volume Rendering, URL, date fetched
[S2] Hardware-Accelerated Volume Rendering Techniques, URL, date fetched
[S3] Hardware Shading for Volume Rendering, URL, date fetched
[S4] Interactive Volume Visualization using 3D Texturing, URL, date fetched
