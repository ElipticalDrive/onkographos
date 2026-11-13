# Illumination and Tone Shading Models

Also called: Illustrative Lighting, NPR Illumination

## Overview
Illumination and Tone Shading Models are non-photorealistic rendering techniques that enhance volume visualization by decoupling lighting from physical light transport, using mathematical functions of view direction, light direction, and scalar field properties to modulate color [G]. They simulate artistic lighting styles, such as painterly effects, cel-shading, or warm-to-cool gradients, to solve the problem of missing depth cues and feature abstraction in translucent volume rendering [G]. Unlike photometric models, these methods map normal vectors or scalar values directly to pre-computed artistic colors or discrete tone bands, providing view-dependent or orientation-dependent shading that rotates with the camera or adapts to local data structure [G].

## Shared principles
The family shares the goal of enhancing volume visualization through stylized lighting and feature abstraction [G]. All methods operate on fragment-level data, typically utilizing the volume sample gradient as a proxy for surface normal orientation [G]. They decouple illumination from the physical light transport equation, instead using mathematical functions of the view direction, light direction, and scalar field properties to modulate color [G]. Common inputs include the normalized gradient $\nabla f_n$, light vectors $L$, and base colors derived from transfer functions [G]. The output is a final color $c$ that combines ambient/gaseous components with directed illumination effects [G].

The framework utilizes four types of properties for enhancement:
1. Volume sample location and value [G].
2. Local volumetric properties, such as gradient and minimal change direction [G].
3. View direction [G].
4. Light information [G].

Volumetric illustration differs from surface-based NPR in several important ways [G]. In surface NPR, surfaces (features) are well-defined, whereas with volumes, feature areas within the volume must be determined through analysis of local volumetric properties because volumetric features vary continuously throughout three-dimensional space [G].

## Subtopics

### [Lit Sphere Shading](785-lit-sphere-shading.md)
Maps eye-space normal vectors to a pre-computed sphere texture to capture artistic lighting styles. It provides view-dependent lighting that rotates with the camera, ignoring front/back face ambiguity.

### [Orientation-Based Color Modification](798-orientation-based-color-modification.md)
Adjusts HSV color components based on the angle between the gradient and view direction. It simulates painterly effects where facing surfaces are desaturated/bright and receding surfaces are darkened/saturated.

### [Volume Illustration Tone Shading](805-volume-illustration-tone-shading.md)
Maps gradient orientation relative to a light source to a warm-to-cool color gradient. It uses a positive dot product threshold to ensure zero contribution at orthogonal orientations, matching diffuse illumination models.

### [Vignette Shading](1830-vignette-shading.md)
A post-processing effect that darkens image corners based on the cubic alignment between ray direction and pixel frame normal, simulating projection falloff.

### [Blinn-Phong Toon Shading](4708-blinn-phong-toon-shading.md)
Applies Blinn-Phong lighting followed by aggressive quantization (posterization) of diffuse bands and binary thresholding of specular highlights.

### [Volumetric Toon Shading](5276-volumetric-toon-shading.md)
Applies cel-shading to volumetric data using a pre-computed 3D normal map and hardware texture combiners, operating directly on the voxel gradient field without mesh extraction.

### [Lighting Transfer Function with Gradient Aligned Sampling](7966-lighting-transfer-function-with-gradient-aligned-sampling.md)
Enhances material boundaries by using two scalar samples along the gradient direction as inputs to a 2D lighting transfer function, dynamically adjusting lighting coefficients based on local data structure.

### Banks Illumination Model for Curves
Uses the illumination model for curves of Banks to provide a better sense of curvature in line bundle rendering [S1, unofficial]. Line bundles can be more opaque in this variant [S1, unofficial].

### Tone Rendering
A non-photorealistic rendering method used to visualize important information in an abstract but coherent way [S2, unofficial].

### Cone Aperture Angle Shading
The parameter $\theta$ determines the effect of surrounding features, with higher values causing more distant features to increase shading variation [S3]. Tested values include $50^\circ$ and $85^\circ$ [S3].

## Background
The concept of Tone Rendering as a non-photorealistic method for abstract visualization was introduced in the context of volume visualization methods, contrasting with Phong Illumination which guarantees higher realistic images [S2, unofficial]. The Banks illumination model for curves was applied in flow visualization techniques to enhance the perception of curvature in vector fields [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S3] PDF A Directional Occlusion Shading Model for Interactive Direct Volume ..., https://cgl.ethz.ch/teaching/scivis_common/Literature/Schott2009.pdf, fetched 2026-10-29
