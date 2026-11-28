# NeRF Composition Framework

Also called: General Implicit Pipeline for Fast NeRF Composition

## Overview
The NeRF Composition Framework is a general implicit pipeline for composing multiple Neural Radiance Fields (NeRF) objects that enables dynamic shadow casting and arbitrary rigid transformations without querying complex spatial structures [S1]. It solves the problem of prohibitively expensive pre-computation and high memory storage costs associated with hybrid or explicit acceleration methods when rendering composed scenes [S1]. Unlike explicit methods that require complex spatial structures (e.g., octrees), this method avoids excessive memory-storage costs by not querying any complex structure [S1].

## How it works
The framework operates as a three-step rendering pipeline that leverages Neural Depth Fields (NeDF) for spatial relationship determination and NeRFs for color estimation [S1].

**Inputs and Outputs:**
*   **Inputs:** A collection of pre-trained NeRF models, a corresponding collection of NeDFs (one per object), camera parameters, light source definitions (point or parallel), and object transformation matrices (arbitrary rigid/affine).
*   **Outputs:** A final composited color buffer.

**Data Structures:**
1.  **Depth Buffer:** Stores the distance from the camera ray origin to the nearest surface intersection for each pixel.
2.  **ID Buffer:** Stores the object identifier (index) corresponding to the surface intersection at each pixel.
3.  **Color Buffer:** Stores the final RGB color for each pixel.
4.  **Shadow Buffer:** Stores the shadow factor (0 to 1) for each pixel.

**Steps:**

1.  **NeDFs Generation Step:**
    For each pixel in the image, cast a ray from the camera through the pixel. For each object in the scene, use its NeDF to compute the distance to the surface intersection along this ray. The NeDF function $F_\Phi$ maps the ray to the distance from the ray's starting point to the intersection point [S1].
    *   Determine the minimum distance among all objects.
    *   Update the **Depth Buffer** with this minimum distance.
    *   Update the **ID Buffer** with the index of the object providing this minimum distance.
    *   This step establishes the spatial occlusion relationships without explicit spatial queries [S1].

2.  **Deferred Shading Step:**
    Using the **Depth Buffer** and **ID Buffer**, determine which NeRF model to query for each pixel.
    *   For each pixel, retrieve the object ID from the ID Buffer.
    *   Query the corresponding NeRF model at the 3D position defined by the camera ray and the depth value from the Depth Buffer.
    *   Compute the color (RGB) and opacity (alpha) using the NeRF's standard volume rendering integral or direct sampling at the surface point.
    *   Fill the **Color Buffer** with these values. Shadows are *not* computed in this step [S1].

3.  **Shadow Step:**
    Generate shadow rays based on the **Depth Buffer** and light source positions.
    *   For each pixel, construct a shadow ray from the surface point (derived from Depth Buffer) to the light source.
    *   Use the NeDFs again to determine if this shadow ray intersects any object's surface before reaching the light.
    *   If an intersection is found, the pixel is in shadow.
    *   Compute a shadow factor (e.g., 0 for fully shadowed, 1 for fully lit).
    *   Fill the **Shadow Buffer** with these factors [S1].

4.  **Compositing:**
    Multiply the **Color Buffer** and the **Shadow Buffer** element-wise to produce the final result [S1].

**Parameters:**
*   **Light Sources:** Supports dynamic shadows cast by either point lights or parallel light sources [S1].
*   **Transformations:** Supports arbitrary rigid transformations (rotation and translation) of NeRF objects [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the depth estimation depends on the NeDF's ability to precisely locate the surface. Errors in NeDF can lead to incorrect occlusion and shadow artifacts [S1].
*   **Degenerate Input:** If an object is not visible or the ray does not intersect any object, the depth buffer may contain undefined values. These must be handled (e.g., by setting a default background color) [G].

## Subtopics

### [Fast NeRF Composition Pipeline](21253-fast-nerf-composition-pipeline.md)
The Fast NeRF Composition Pipeline is a three-step rendering framework that replaces traditional ray marching with a single-sample evaluation per pixel to achieve interactive-level speeds for previewing composed NeRF scenes. It builds on [NeRF Composition Framework](21239-nerf-composition-framework.md) by utilizing Neural Depth Fields (NeDF) to generate depth and object ID buffers, enabling deferred shading and shadow casting without querying complex spatial structures.

## Operating characteristics
*   **Time Complexity:** The method enables interactive-level speeds for previewing composed NeRF scenes [S1]. When working with a single object in a composed scene, manipulation speeds can reach real-time in CUDA implementation [S1].
*   **Memory Complexity:** Avoids excessive memory-storage costs by not querying any complex structure [S1].
*   **Strengths:** Enables dynamic shadows within or between objects; supports arbitrary rigid transformations; low storage cost; fast previewing [S1].
*   **Limitations:** May sacrifice perfect quality for interactive speed; relies on the accuracy of NeDF for depth estimation [S1].
*   **When to use:** When interactive previewing of NeRF composition is preferred over perfect quality, especially during manipulation of objects [S1].
*   **When not to use:** When high-fidelity, artifact-free rendering is required without the trade-off of speed [S1].

## Background
The method was proposed in "A General Implicit Framework for Fast NeRF Composition and Rendering" [S1]. It addresses the limitations of modeling a scene as a single NeRF, which restricts functionality in scenarios requiring isolation and manipulation of individual objects (e.g., driving scenes) [S1]. Previous methods for NeRF composition were often incompatible with various implicit methods, preventing real-time composition [S1]. This framework introduces Neural Depth Fields (NeDF) as a novel surface representation for implicit surfaces, aiming for fast and precise depth estimation under flexible camera activity and arbitrary affine transformations [S1]. It leverages an intersection neural network to query NeRF for acceleration instead of depending on explicit spatial structures [S1].

## Sources
[S1] A General Implicit Framework for Fast NeRF Composition and Rendering, https://doi.org/10.1609/aaai.v38i3.27952, fetched 2026-10-29
