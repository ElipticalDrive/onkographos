# WebGL/WebGPU Browser Implementations

Also called: Browser-based GPU Volume Rendering

## Overview
This family implements direct volume ray casting within web browsers using WebGL or WebGPU APIs, solving the problem of rendering volumetric data without plugins by adapting GPU-accelerated ray marching to browser-specific constraints like the lack of 3D texture support in WebGL 1.0 or WebGPU's memory alignment rules [G]. It differs from desktop GPU implementations by prioritizing web standard compatibility and handling browser-specific data encoding workarounds, such as packing depth into RGBA channels or using 2D texture atlases [G].

## Shared principles

## Subtopics

### [WebGL Multi-pass Ray-casting with 3D Geometry Intersection](6575-webgl-multi-pass-ray-casting-with-3d-geometry-intersection.md)
Enables interactive coexistence of volumetric datasets and 3D triangular meshes in WebGL by using a multi-pass rendering pipeline to handle occlusion, blending, and ray intersection. It sets itself apart from single-pass approaches by explicitly separating depth, surface color, and volume traversal into distinct passes to correctly manage back-to-front blending and geometry intersections. Choose this method when rendering opaque and transparent 3D geometry within or alongside volume data in WebGL environments.

### [WebGL Path Tracing Volume Rendering](8592-webgl-path-tracing-volume-rendering.md)
An interactive voxel volume renderer using path tracing that runs in modern browsers via WebGL 2.0. It sets itself apart from standard ray casting by computing normals via central differences of opacity and applying Phong-like lighting with soft shadows, rather than relying solely on direct illumination. Choose this method when lighting and shadowing effects are required for volumetric data in WebGL 2.0.

### [StructuredVolumeRenderer](17796-structuredvolumerenderer.md)
StructuredVolumeRenderer is a reusable, graph-native GPU renderer for scalar, vector, and hybrid volume data in WebGPU environments. It sets itself apart from sibling WebGL implementations by leveraging WebGPU's storage buffer and texture binding models, manual trilinear interpolation, and fixed-step jittered ray marching with early opacity termination. Choose this method when rendering structured grid data (buffers or 3D textures) in WebGPU with explicit memory control.

## Background

