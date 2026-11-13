# Hatching and Stroke-based Rendering

Also called: Stroke-based Volume Illustration, Pen-and-Ink Volume Rendering

## Overview
Hatching and Stroke-based Rendering (Hatching) is a family of non-photorealistic volume rendering techniques that simulate hand-drawn line art, such as hatching, stippling, or paint strokes, to convey shape, spatial relationships, and aesthetic qualities of 3D volumetric data. It solves the problem of abstracting continuous volumetric features into discrete, interpretable graphical elements that mimic traditional artistic media. Unlike surface-based NPR methods where features are well-defined, this family determines feature areas through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space [S2, S4, unofficial]. It sets itself apart from its siblings by explicitly combining line-strip primitives or textured quads with direct volume rendering (DVR) or particle systems, using data-driven seed point selection and curvature-guided tracing to minimize memory and generation time while maintaining interactivity [S2, unofficial].

## Shared principles
The family shares the goal of enhancing volume visualization through stylized lighting and feature abstraction [G]. All methods operate on fragment-level data, typically utilizing the volume sample gradient as a proxy for surface normal orientation [G]. They decouple illumination from the physical light transport equation, instead using mathematical functions of the view direction, light direction, and scalar field properties to modulate color [G]. Common inputs include the normalized gradient $\nabla f_n$, light vectors $L$, and base colors derived from transfer functions [G]. The output is a final color $c$ that combines ambient/gaseous components with directed illumination effects [G].

The framework utilizes four types of properties for enhancement:
1. Volume sample location and value [G].
2. Local volumetric properties, such as gradient and minimal change direction [G].
3. View direction [G].
4. Light information [G].

Volumetric illustration differs from surface-based NPR in several important ways [G]. In surface NPR, surfaces (features) are well-defined, whereas with volumes, feature areas within the volume must be determined through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space [G].

## Subtopics

### [Interactive Volume Illustration (Hatching)](5272-interactive-volume-illustration-hatching.md)
[Interactive Volume Illustration (Hatching)](5272-interactive-volume-illustration-hatching.md) is an object-space technique for illustrating volumetric data by simulating free-hand line art (hatching) using hardware-assisted volume and line rendering. It solves the problem of conveying shape and spatial relationships in 3D volumes by tracing hatches as polylines over implicit surfaces along either a fixed direction or a smoothed principal curvature direction. Unlike surface-based NPR methods where features are well-defined, this method determines feature areas through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space. It sets itself apart from its siblings by explicitly combining line-strip primitives with direct volume rendering (DVR) in a two-pass pipeline, using data-driven seed point selection to minimize memory and generation time.

### [Fast Paint Texture](14343-fast-paint-texture.md)
[Fast Paint Texture](14343-fast-paint-texture.md) is a stylization technique that simulates 3D paint undulations and overlap shadows on 2D strokes by generating a height field from stroke height maps and overlap counts, rendered via Phong shading, thereby avoiding the computational cost of 3D mesh generation. It distinguishes itself from sibling methods like [Interactive Volume Illustration](5272-interactive-volume-illustration-hatching.md) by explicitly modeling the physical accumulation of paint layers through a 2D height field rather than relying solely on surface normals or simple shading, and from [Visual Complexity (VC) Algorithm](14351-visual-complexity-vc-algorithm.md) by providing the specific texturing pass mechanism used within the VC framework to create embossed strokes.

### [Visual Complexity (VC) Algorithm](14351-visual-complexity-vc-algorithm.md)
[Visual Complexity (VC) Algorithm](14351-visual-complexity-vc-algorithm.md) is a non-photorealistic volume rendering technique that simulates the local visual variations found in hand-painted artworks to enhance aesthetic judgment and detail perception. Unlike Interpretational Complexity (IC), which establishes global trends across the volume, VC focuses on local variations in stroke properties such as color, thickness, and contour to create a dense, textured appearance. It differs from [Interactive Volume Illustration (Hatching)](5272-interactive-volume-illustration-hatching.md) by employing a single-region, full-coverage painting strategy with varied stroke geometries rather than structured hatching lines, and from [Fast Paint Texture](14343-fast-paint-texture.md) by integrating embossing as a post-processing lighting effect on 2D stroke textures rather than a primary geometric modeling step.

### VolFliesGPU Framework
#### definition
- [S1] An interactive illustrative visualization framework for real-time pen-and-ink style rendering of volume data, based on a particle system.
#### layout
- [S1] Input: Volume data stored in a 3D texture; Proxy geometry comprising a 3D grid of equally spaced vertices.
#### usage
- [S1] Uses the VolumeFlies hidden surface removal filter: iso-surface is splatted with uniquely colored cones generated by the geometry shader; particle visibility is determined by an off-screen framebuffer.

### Non-photorealistic Volume Rendering via Point Sampling
#### definition
- [S2, unofficial] A methodology for sampling and rendering 3D volume data to simulate classical ink-and-paint non-photorealistic styles, addressing the lack of artist-friendly NPR volume rendering tools.
#### step
- [S2, unofficial] Scatter point data within the 3D volume data based on density values, or create a custom point cloud.
- [S2, unofficial] Ray cast from the camera to the hit position of the volume.
- [S2, unofficial] From the ray's hit position, cast a secondary ray towards a direction, either towards a light source for shading or continuing in the same direction for a headlamp effect.
- [S2, unofficial] Tally and normalize the number of scattered points found within a radius of the direction of the secondary ray.
- [S2, unofficial] Interpolate between the lit color of the volume and the background color using the normalized value from the previous step.

### Curved Strokes
#### definition
- [S3] A method to render strokes as Bézier curves with control points positioned based on data attribute orientations.
#### step
- [S3] Query data attribute values at three sample points s0, s1, s2. Compute associated orientations θ0, θ1, θ2 to position control points A, B, C, D.
- [S3] Place control points so that segment BC has orientation θ0, AB has orientation θ1, and CD has orientation θ2.
- [S3] If max(d1, d2) is large, draw the stroke as a curve; otherwise, likely draw as linear segments (implied by contrast with 'draw stroke as a curve').

### Variegation
#### definition
- [S3] A technique to simulate paint variation by drawing strokes as three quadrilaterals with varying colors and widths.
#### step
- [S3] Sample center color C=(R,G,B) and length l, width w. Vary C to C1, C2 in small interval. Vary w to w1, w2 such that 0 <= w1, w2 < w/8.
- [S3] Draw q centered on sample with length l, width w-(w1+w2), color C. Draw q1 next to q with length l, width w1, color C1. Draw q2 opposite q1 with length l, width w2, color C2.

## Background
The origins of stroke-based volume rendering lie in the broader field of Non-Photorealistic Rendering (NPR) applied to scientific visualization. Early work focused on surface-based hatching and contouring. The application of these techniques to volumetric data required new approaches to handle the continuous nature of volumes. The VolFliesGPU framework, introduced by Busking et al. in 2008, pioneered the use of GPU-based particle systems for interactive illustrative volume rendering, leveraging GPGPU paradigms for real-time performance [S1]. The Visual Complexity algorithm, developed by Liao et al., introduced sophisticated stroke placement, variegation, and flank texturing to simulate hand-painted aesthetics [S3]. Fast Paint Texture provided a method for simulating physical paint properties like overlap and shadow without 3D mesh generation [S1]. More recent methodologies, such as those by [S2, unofficial], have explored point-sampling techniques to generalize ink-and-paint styles for volume data.

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Developing Generalized Volume Sampling and Volume Shading for Non-Photorealistic Rendering, https://hdl.handle.net/1969.1/173845, fetched 2026-10-26 (unofficial)
[S3] Engaging viewers through nonphotorealistic visualizations, https://doi.org/10.1145/1274871.1274886, fetched 2026-10-28
