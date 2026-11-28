# Illumination and Tone Shading Models

Also called: Illustrative Illumination, Non-Photorealistic Lighting for Volumes

## Overview
Illumination and Tone Shading Models provide non-photorealistic lighting cues to enhance the visual clarity and artistic style of volume renderings. Unlike photometric models that simulate physical light transport, these methods map geometric or scalar properties to pre-defined color variations, prioritizing interpretability and aesthetic control over physical accuracy. They are distinct from [Geometric Buffers](781-geometric-buffers-g-buffers.md) by focusing on color modulation rather than depth/normal encoding, and from [Base Gaseous Illumination Renderer](794-base-gaseous-illumination-renderer.md) by incorporating directed shading components.

## Shared principles
The family shares the goal of enhancing volume visualization through stylized lighting. All methods operate on fragment-level data, typically utilizing the volume sample gradient as a proxy for surface normal orientation. They decouple illumination from the physical light transport equation, instead using mathematical functions of the view direction, light direction, and scalar field properties to modulate color. Common inputs include the normalized gradient $\nabla f_n$, light vectors $L$, and base colors derived from transfer functions. The output is a final color $c$ that combines ambient/gaseous components with directed illumination effects.

## Subtopics

### [Lit Sphere Shading](785-lit-sphere-shading.md)
Uses an image of a sphere as a lookup table for view-dependent lighting. It maps eye-space normals directly to pre-computed artistic styles, ignoring front/back face ambiguity. Choose this for stylized, view-dependent lighting that rotates with the camera.

### [Volume Illustration Tone Shading](805-volume-illustration-tone-shading.md)
Combines gaseous illumination with directed shading based on the signed dot product between the gradient and light vector. It uses continuous warm/cool color interpolation. Choose this for realistic yet stylized volume rendering with clear directional cues.

### [Vignette Shading](1830-vignette-shading.md)
Applies a cubic vignette effect based on the alignment between ray direction and the z-axis. Choose this for adding depth and focus to the volume.

### [Blinn-Phong Toon Shading](4708-blinn-phong-toon-shading.md)
A toon-shaded variant of Blinn-Phong that posterizes diffuse lighting and harshly cuts specular reflections. Choose this for discrete, cartoon-like lighting effects.

### [Volumetric Toon Shading](5276-volumetric-toon-shading.md)
Simulates toon shading using a 3D normal map and hardware texture combiners. Choose this for efficient, hardware-accelerated cel shading without explicit surface vertices.

## Background
The concept of non-photorealistic volume rendering emerged to address the ambiguity of standard volume rendering, where internal structures can be hard to distinguish. Early work focused on enhancing edge detection and silhouette rendering. The specific illumination models in this family, such as Lit Sphere Shading and Volume Illustration Tone Shading, were developed to provide intuitive lighting cues that help viewers perceive shape and depth in volumetric data without the computational cost of full global illumination. These techniques are widely used in medical visualization, scientific data exploration, and artistic volume rendering.

## Sources
[S1] Lit Sphere Shading, https://example.com/lit-sphere, 2023-10-01
[S2] Volume Illustration Tone Shading, https://example.com/volume-illustration, 2023-10-02
