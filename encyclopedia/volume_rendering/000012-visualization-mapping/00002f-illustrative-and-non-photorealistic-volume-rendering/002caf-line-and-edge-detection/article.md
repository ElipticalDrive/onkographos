# Line and Edge Detection

Also called: Volumetric Edge Detection, Silhouette Detection, Outline Generation

## Overview
Line and Edge Detection is a family of non-photorealistic volume rendering techniques that enhance perceptual clarity by explicitly detecting and rendering edges, silhouettes, or structural boundaries within volumetric data [S2][S1, unofficial]. Unlike surface-based NPR, which relies on well-defined geometric surfaces, this family operates on continuous volumetric fields where feature areas must be determined through local property analysis because volumetric features vary continuously throughout three-dimensional space [S2][S4, unofficial]. It distinguishes itself from sibling methods like [Line Illustration Detection](772-line-illustration-detection.md) by focusing on boundary emphasis and structural outlines rather than flow-aligned strokes, and from [LOG-based Silhouette Detection](6322-log-based-silhouette-detection.md) by employing diverse detection mechanisms ranging from gradient-based particle tracing to user-defined interactive selection [S1, unofficial][S2].

## Shared principles
The family shares the fundamental goal of enhancing perceptual clarity through non-photorealistic edge representation. Common mechanisms include:
1.  **Edge Detection:** Identifying boundaries based on geometric properties (normals, curvature, topology) or image-space properties (depth, normal discontinuities) [S2][S3, S4, unofficial].
2.  **Line Rendering:** Converting detected edges into line primitives (strips, segments) or stylized strokes [S2][S3, S4, unofficial].
3.  **Compositing:** Integrating these lines with the underlying volume or surface rendering, often using depth buffering for hidden line removal [S2][S3, S4, unofficial].

Volumetric illustration differs from surface-based NPR in several important ways [S2][S4, unofficial]. In surface NPR, surfaces (features) are well-defined, whereas with volumes, feature areas within the volume must be determined through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space [S2][S4, unofficial].

## Subtopics

### [Line Illustration Detection](772-line-illustration-detection.md)
Line Illustration Detection is a family of Non-Photorealistic Rendering (NPR) techniques designed to detect and render lines that communicate the shape, structure, and boundaries of volumetric data or isosurfaces. It solves the problem of visualizing complex geometry, such as medical models, where standard photorealistic rendering may obscure structural details. What sets this family apart from its siblings is its focus on line-based abstraction rather than silhouette outlines or edge highlights; it distinguishes between analytical approaches, which are vector-oriented and resolution-independent, and pixel-based methods, which are faster and simpler but rely on image-space buffers.

### [Face-based volume rendering with edge highlights](2453-face-based-volume-rendering-with-edge-highlights.md)
This method renders volumetric data by treating individual cells (faces) as the primary sampling units, applying a scalar-based colormap for the volume body while simultaneously detecting and highlighting cell edges. It distinguishes itself from sibling methods like [Line Illustration Detection](772-line-illustration-detection.md) and [Outline and Contour Generation](4137-outline-and-contour-generation.md) by computing edge highlights directly within the face fragment shader based on distance to cell vertices and edges, rather than relying on post-processing contours or separate line-detection passes. It solves the problem of visualizing internal structures and boundaries in 3D volumes by providing explicit edge cues that enhance depth perception and structural clarity without obscuring the underlying scalar data.

### [Outline and Contour Generation](4137-outline-and-contour-generation.md)
Outline and Contour Generation is a family of non-photorealistic volume rendering techniques that enhance perceptual clarity by explicitly detecting and rendering edges, silhouettes, or topological boundaries within volumetric data. Unlike surface-based NPR, which relies on well-defined geometric surfaces, this family operates on continuous volumetric fields where feature areas must be determined through local property analysis. It distinguishes itself from sibling methods like [Line Illustration Detection](772-line-illustration-detection.md) by focusing on boundary emphasis and structural outlines rather than flow-aligned strokes, and from [LOG-based Silhouette Detection](6322-log-based-silhouette-detection.md) by employing diverse detection mechanisms ranging from gradient-based particle tracing to user-defined interactive selection.

### [LOG-based Silhouette Detection](6322-log-based-silhouette-detection.md)
LOG-based Silhouette Detection is a volumetric edge detection technique used to identify silhouette voxels for rendering sketch-like curves, replacing a more exhaustive 3x3x3 subspace search to improve performance. It identifies boundaries based on local volumetric properties (gradient magnitude and Laplacian of Gaussian values) to highlight structures such as bones or internal features. It sets itself apart from sibling methods like Line Illustration Detection by operating directly on volumetric scalar fields using a precomputed Laplacian of Gaussian operator rather than image-space depth discontinuities or surface normals.

#### Voxel Highlight Edge Detection
- **Step**: [S1, unofficial] Calculate entry point $hp = org + dir \times t_{cur}$. Calculate local cell coordinates $local = (hp - bmin) / cell$. Extract 2D face coordinates $f$ based on entry axis: if $axis=0$, $f=fract(local.yz)$; if $axis=1$, $f=fract(local.xz)$; if $axis=2$, $f=fract(local.xy)$. Calculate edge distance $edge = \min(\min(f.x, 1.0-f.x), \min(f.y, 1.0-f.y))$. If $edge < 0.06$, highlight color is yellow $(1.0, 0.85, 0.1)$; otherwise mix lit color with yellow at 0.35 weight.

#### Hough Space Silhouette Detection
- **Definition**: [S2] An approach by Li et al. to detect silhouettes in Hough space for edge sampling.
- **Usage**: [S2] Used in Li's edge sampling approach [22, 23].

#### Object Space Enhancement
- **Definition**: [S3, unofficial] A pre-computed shape enhancement cue based on object-space analysis using a sharpened filter (Laplacian-like) on a 3x3x3 unit volume to enhance edges and contours, solving the concave/convex inconsistency of local illumination.
- **Step**: [S3, unofficial] Select a 3x3x3 point unit volume surrounding the sampling center in the data volume. [S3, unofficial] Clamp the shape factor: if $g(x, y, z) + c > 1$, set it to 1.
- **Formula**: [S3, unofficial] Calculate the sharpened filter value $52f$ as: $52f = [f(x+1, y, z) + f(x-1, y, z) + \dots + f(x+1, y+1, z+1)] - 26f(x, y, z)$, where $f(x,y,z)$ is the intensity of the sampling point. [S3, unofficial] Calculate the enhanced intensity $g(x,y,z)$ as: $g(x, y, z) = f(x, y, z) - 52f$. [S3, unofficial] Calculate the shape factor $S_{object}$ as: $S_{object} = g(x, y, z) + c$, where $c$ is a factor to adjust the effect of object space enhancement.
- **Parameter**: [S3, unofficial] Factor $c$: A parameter to adjust the effect of object space enhancement.
- **Pitfall**: [S3, unofficial] Local illumination alone cannot solve the display of concave parts; this cue makes up for that shortcoming.
- **Complexity**: [S3, unofficial] The cue is pre-computed before ray casting, so it does not occupy rendering time, allowing real-time interactivity.

#### Feature Detection and Adaptive Lens Positioning
- **Definition**: [S4, unofficial] Detects prominent edges in the volume using a Sobel filter and dynamically positions a magnification lens to center on the largest connected feature.
- **Step**: [S4, unofficial] Apply Sobel filter to the volume: `edges = ndimage.sobel(self.volume_data)`. [S4, unofficial] Compute threshold as twice the mean edge magnitude: $T = \text{mean}(E) \times 2$. [S4, unofficial] Create binary feature mask: $M = E > T$. [S4, unofficial] Label connected components in the mask: `labeled, num_features = ndimage.label(feature_mask)`. [S4, unofficial] Find bounding box of the largest feature: `largest_feature = ndimage.find_objects(labeled)[0]`. [S4, unofficial] Compute center coordinates: $C_y = (s_{start} + s_{stop}) / 2$, $C_x = (s_{start} + s_{stop}) / 2$. [S4, unofficial] Normalize center to [0,1] range and update lens position: `self.lens['position'] = glm.vec2(center[1] / H, center[0] / W)`.

## Background
Line and edge detection in volume rendering emerged as a response to the limitations of photorealistic rendering in conveying structural information, particularly in medical and scientific visualization where internal boundaries are critical [S2][S1, unofficial]. Early approaches relied on surface extraction (isosurfacing) followed by traditional NPR techniques, but this introduced artifacts and lost volumetric context [S2]. Direct volumetric edge detection methods were developed to operate on the scalar field itself, utilizing gradient information and local neighborhood analysis to identify features without explicit surface reconstruction [S1, unofficial][S3, unofficial]. The field has evolved to include both analytical methods for resolution-independent line drawing and pixel-based methods for efficient real-time rendering [S1, unofficial]. Techniques such as LOG-based detection and object-space enhancement cues have been introduced to improve the robustness of edge detection in noisy or low-contrast data [S1, unofficial][S3, unofficial].

## Sources
[S1] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
[S2] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S3] Local Detail Enhancement for Volume Rendering under Global Illumination, https://doi.org/10.2312/pg.20161334, fetched 2026-10-26 (unofficial)
[S4] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
