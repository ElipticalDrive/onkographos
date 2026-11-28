# Illustrative and Non-Photorealistic Rendering Styles

Also called: Illustrative Volume Rendering, NPR Volume Visualization

## Overview
This family encompasses methods that apply non-photorealistic rendering (NPR) techniques to volumetric data to enhance the perception of structure, shape, orientation, and depth . Unlike traditional physics-based volume rendering, which often results in ambiguous structural interpretation due to the continuous nature of volumetric data, these methods decouple illumination from physical light transport to modulate color and opacity based on local volumetric properties [G]. They are distinguished from surface-based NPR by deriving features from continuous 3D scalar or vector field properties rather than well-defined geometric surfaces .

## Shared principles
All methods in this family modify the standard volume rendering pipeline to incorporate stylized enhancements. The core mechanism involves transforming voxel properties (color and opacity) before compositing, based on an analysis of local volumetric features such as gradients, view direction, and light direction .

1.  **Feature Analysis**: For each sample along a ray, local properties are computed. This typically includes the scalar value $s$, the gradient $\nabla f$ (normalized to $\nabla f_n$), and the view $\vec{v}$ and light $\vec{l}$ directions [G].
2.  **NPR Transformation**: Mathematical functions of these properties modulate the base color $c_{base}$ and opacity $\alpha$ derived from the transfer function. Common transformations include boundary enhancement (triggered by high gradient magnitude) and tone shading (based on dot products with light/view vectors) .
3.  **Compositing**: The transformed values are composited using standard front-to-back volume compositing:
    $$
    C_{out} = \sum_{i=1}^{N} c_{final,i} \cdot \alpha_{final,i} \cdot \prod_{j=1}^{i-1} (1 - \alpha_{final,j})
    $$
    All voxels contribute to the final image, but their contribution is stylized .

## Subtopics

### [Volume Illustration](810-volume-illustration.md)
A unified framework that augments physics-based rendering with NPR techniques like boundary detection and tone shading. It is chosen when a general enhancement of structural perception in scalar fields is needed without specific artistic constraints.

### [Illustrative Rendering](14479-illustrative-rendering.md)
Combines Phong shading, half-tone hatching, and silhouette outlines specifically for 3D vector fields (streamtapes). It is chosen for visualizing flow structures where avoiding complex line patterns on narrow objects is critical.

### [Volume Painting](19398-volume-painting.md)
Simulates artistic brush-strokes using user-defined directional texture maps and Perlin noise. It is chosen when a painterly, artistic effect is desired, requiring the simulation of stroke direction and width in 3D space.

### [Illustrative Style Selection and Attribute Tuning](19705-illustrative-style-selection-and-attribute-tuning.md)
A strategy for classifying anatomical objects into focus, context, and container categories to apply specific styles (silhouettes, shading, transparency). It is chosen to reduce visual clutter and guide viewer attention in complex medical scenes.

## Background
The concept of decoupling illumination from physical light transport in volume rendering was introduced to address the limitations of photorealistic rendering in conveying structural information . Volume Illustration, a foundational method in this family, was developed to enhance the perception of shape and depth in volumetric data by applying stylized enhancements such as silhouettes and tone shading . Subsequent methods like Illustrative Rendering and Volume Painting extended these principles to specific data types (vector fields) and artistic styles (brush strokes) . The classification-based approach (Focus/Context/Container) was developed to manage visual hierarchy in complex anatomical visualizations .

