# Hardware Implementation Pipeline

Also called: Single Scattering Rendering Pipeline, Volumetric Shadow Volume Method

## Overview
The Hardware Implementation Pipeline is a 5-step method for rendering single scattering and volumetric shadows in real-time using GPU shaders and the stencil buffer, designed for minimal texture memory cost [S1, unofficial]. It differs from ray-marching approaches by leveraging conventional polygonal rendering and shadow volume logic, making it straightforward to integrate into traditional graphics engines [S1, unofficial].

## How it works
The pipeline consists of five steps, repeated for each light source [S1, unofficial]. An initialization stage is required beforehand to obtain ambient lighting and the first depth map [S1, unofficial].

**Inputs:** Scene geometry (moving shadow casters), light source position (moving or static), view ray direction.
**Outputs:** Radiance $L$ with direct single scattering ($L_{dss}$) and volumetric shadows.

**Data Structures:**
*   **Stencil Buffer:** 1-bit or integer buffer tracking occlusion status.
*   **Shadow Planes:** Geometry derived from object silhouettes relative to the light source.

**Steps:**

1.  **CPU: Compute Silhouettes.** Compute the silhouettes of every moving shadow caster. If the light source is moving, every silhouette must be recomputed [S1, unofficial].
2.  **GPU: Base Rendering.** Render the scene using conventional polygonal rendering to obtain direct illumination and indirect single scattering. Use shadow plane algorithms for surface shadows. The stencil buffer now contains lit areas of the scene. Add an ambient fog to account for absorption and multiple scattering [S1, unofficial].
3.  **GPU: Single Scattering Pass.** Render the scene once more. Compute direct single scattering for each vertex. Set depth test to equality. Only render lit parts of the scene using the stencil buffer [S1, unofficial].
4.  **CPU: Sort Shadow Planes.** Sort the shadow planes determined by the object's silhouettes in back-to-front order [S1, unofficial].
5.  **GPU: Shadow Volume Rendering.** Render shadow planes in the sorted order.
    *   **Depth Test:** Accepts only planes closer to the camera.
    *   **Stencil Logic:**
        *   Front-facing planes: Decrement stencil buffer. Allow fragments if stencil is 1.
        *   Back-facing planes: Increment stencil buffer. Allow fragments if stencil is 0.
    *   **Contribution:** Front-facing planes add contribution; back-facing planes subtract contribution [S1, unofficial].

**Radiance Model:**
The observed radiance $L$ along a view ray $\vec{u}$ from point $P$ is given by [S1, unofficial]:

$$L_{\vec{u}\vec{P}} = L_{dss}(\vec{u}\vec{P}) + e^{-k_t OP} L(P) + L_a e^{-k_t OP} + L_{oiss}(\vec{u}\vec{P})$$

Where:
*   $L_{dss}(\vec{u}\vec{P})$: Direct single scattering.
*   $L(P)$: Radiance at point $P$ (from step 2/3).
*   $k_t$: Extinction coefficient.
*   $OP$: Distance from observer to $P$.
*   $L_a$: Ambient radiance.
*   $L_{oiss}(\vec{u}\vec{P})$: Other indirect single scattering terms [S1, unofficial].

**Pitfalls:**
*   Moving lights or objects require silhouette recomputation in Step 1 [S1, unofficial].
*   Initialization stage is mandatory before the main pipeline [S1, unofficial].

## Operating characteristics
*   **Complexity:** Real-time performance for classical graphics scenes [S1, unofficial].
*   **Memory:** Minimal texture memory cost [S1, unofficial].
*   **Strengths:** Easy to integrate into traditional engines; supports moving objects and lights [S1, unofficial].
*   **Limitations:** Limited to single scattering and volumetric shadows; relies on stencil buffer operations which may have performance implications on some hardware [S1, unofficial].

## Background
The method was proposed in "Single Scattering Effects for Computer Games" [S1, unofficial]. It follows the same strategy as the shadow volume algorithm, using only shaders and textures [S1, unofficial].

## Sources
[S1] Single Scattering Effects for Computer Games, https://hal.science/hal-00681726v1, fetched 2026-10-26 (unofficial)
