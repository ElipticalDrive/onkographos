# Neural Depth Renderer as DDF

Also called: Neural Depth Rendering as Directed Distance Field

## Overview
This method treats a neural depth renderer $f$ as a conditional Directed Distance Field (DDF) by mapping camera parameters $\Pi$ and scene shapes $\Lambda_S$ to depth and visibility images [S1]. It differs from standard neural rendering by explicitly framing the output pixels as functions of ray origins and directions derived from camera extrinsics, thereby inheriting DDF properties such as the relationship between depth gradients and surface normals [S1].

## How it works
**Inputs and Outputs:**
- Input: Continuous camera parameters $\Pi \in \Lambda_\Pi$ (intrinsics and extrinsics) and shape representation $\Lambda_S$.
- Output: An $m$-pixel image where each pixel $p$ contains a tuple $(d(p, v), \xi(p, v))$, where $d$ is depth and $\xi$ is visibility [S1].

**Data Structures:**
- Camera Parameters $\Pi$: Defines the center of projection $p$ and ray direction $v$ for each pixel. $p$ is determined by the camera position; $v$ is determined by the pixel's direction from $p$ into the scene [S1].
- Shape $\Lambda_S$: The geometric representation queried by the renderer.

**Steps:**
1. For each pixel in the output image, compute the ray origin $p$ and direction $v$ based on the camera parameters $\Pi$ [S1].
2. Evaluate the neural renderer $f(\Pi, \Lambda_S)$ to obtain the depth $d$ and visibility $\xi$ for the ray $(p, v)$ [S1].
3. The value at pixel $p$ is set to $(d(p, v), \xi(p, v))$ [S1].

**Formulas:**
- Pixel Value: $(d(p, v), \xi(p, v))$ [S1].
- Ray Origin: $p = \text{CenterOfProjection}(\Pi)$ [S1].
- Ray Direction: $v = \text{RayDirection}(p, \text{PixelIndex}, \text{Intrinsics}(\Pi))$ [S1].

**Properties:**
- Property II: The gradient of the depth image with respect to camera position, $\partial_{C_p} I_d$, relates to the surface normals image [S1].
- Property III: The gradient of the depth image with respect to viewpoint, $\partial_{C_v} I_d$, is constrained [S1].

**Edge Cases:**
- The method holds regardless of the internal architecture of $f$ [S1].
- Precision issues in ray-triangle intersection are noted as a general concern for mesh data extraction but are not detailed for the neural renderer itself [S1].

## Subtopics

### [CPDDF-based SI3DR](9291-cpddf-based-si3dr.md)
Uses a Conditional Probabilistic DDF for single-image 3D reconstruction, differing by conditioning on a latent code for view-consistent rendering.

## Operating characteristics
Not given in the sources.

## Background
This perspective was proposed to parallelize DDFs and improve the framework for differentiable neural rendering of geometric quantities [S1]. It builds on the properties of Directed Distance Fields to apply them to neural rendering frameworks [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
