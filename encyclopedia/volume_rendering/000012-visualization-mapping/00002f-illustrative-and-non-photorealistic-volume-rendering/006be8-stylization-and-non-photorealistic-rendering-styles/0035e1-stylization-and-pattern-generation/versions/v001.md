# Stylization and Pattern Generation


## Overview
Stylization and Pattern Generation applies non-photorealistic rendering (NPR) techniques to volume data to enhance feature visibility and aesthetic appeal [G]. It solves the ambiguity of internal structures in complex volumes by replacing photorealistic shading with abstract patterns, such as stippling or example-based textures [S2, unofficial]. Unlike [Geometric Buffers](781-geometric-buffers-g-buffers.md) which focus on geometric abstraction, or [Illumination and Tone Shading Models](3857-illumination-and-tone-shading-models.md) which refine lighting, this family explicitly decouples illumination from physical light transport to modulate color and density based on user-defined artistic styles [S4, unofficial].

## Shared principles
The family shares the goal of enhancing volume visualization through stylized lighting and feature abstraction [G]. All methods operate on fragment-level data, typically utilizing the volume sample gradient as a proxy for surface normal orientation [G]. They decouple illumination from the physical light transport equation, instead using mathematical functions of the view direction, light direction, and scalar field properties to modulate color [S4, unofficial]. Common inputs include the normalized gradient $\nabla f_n$, light vectors $L$, and base colors derived from transfer functions [G]. The output is a final color $c$ that combines ambient/gaseous components with directed illumination effects [G].

The framework utilizes four types of properties for enhancement:
1. Volume sample location and value [S4, unofficial].
2. Local volumetric properties, such as gradient and minimal change direction [S4, unofficial].
3. View direction [S4, unofficial].
4. Light information [S4, unofficial].

Volumetric illustration differs from surface-based NPR in several important ways [S2, S4, unofficial]. In surface NPR, surfaces (features) are well-defined, whereas with volumes, feature areas within the volume must be determined through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space [S2, S4, unofficial].

## Subtopics

### [Stipple Volume Rendering](6321-stipple-volume-rendering.md)
Replaces rasterization with stippled points to create a sketch-like appearance. It uses object-space stippling to determine points per voxel, allowing direct feature enhancements like silhouette detection without texture memory limits.

### [Example-Based Volume Illustration](6614-example-based-volume-illustration.md)
Generates styles by combining 3D pattern textures (Wang Cubes) with color transfer from example images. It uses a shared cube tiling with limited samples to simplify user interaction.

### Procedural Pattern Coloring
**Definition**: Procedural coloring for butterfly wings using polar coordinates in wing space to generate veins, margins, bands, and apex patterns [S1, unofficial].
**Step**: Convert wing coordinates to polar: root = w - (0.04, 0.02), ang = atan(root.y, root.x), rad = length(root) [S1, unofficial]. Calculate base field color: mix AMBER_DEEP and AMBER_BRIGHT based on radius, then apply tint [S1, unofficial]. Calculate vein pattern: combine three sine waves with different frequencies and powers, scaled by radius [S1, unofficial]. Calculate margin, band, inner, and outer patterns using smoothsteps on outline and angle [S1, unofficial]. Mix colors: start with field, mix in INK for veins, margin, and apex, mix in CREAM for inner and outer bands [S1, unofficial].

## Background
Stylization in volume rendering emerged from the need to convey complex internal structures that are often obscured in traditional volume rendering. Early work focused on adapting surface-based NPR techniques, such as hatching and stippling, to volumetric data. Stipple Volume Rendering was introduced to address the specific challenges of volumetric feature detection, such as the lack of well-defined surfaces [S2]. Example-Based Volume Illustration later simplified the creation of diverse artistic styles by leveraging example images and Wang Cubes [S1, unofficial]. These methods are primarily used in scientific visualization and medical imaging to highlight specific anatomical or physical features through artistic abstraction [S2, unofficial].

## Sources
[S1] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
