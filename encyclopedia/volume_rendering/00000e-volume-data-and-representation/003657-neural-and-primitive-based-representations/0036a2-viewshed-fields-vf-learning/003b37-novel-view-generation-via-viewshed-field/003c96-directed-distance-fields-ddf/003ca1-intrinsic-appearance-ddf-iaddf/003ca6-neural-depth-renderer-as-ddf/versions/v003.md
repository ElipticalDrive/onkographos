# Neural Depth Renderer as DDF

Also called: Neural Depth Rendering as Directed Distance Field

## Overview
This method treats a neural depth renderer $f$ as a conditional Directed Distance Field (DDF) by mapping camera parameters $\Pi$ and scene shapes $\Lambda_S$ to depth and visibility images [S1]. It differs from standard neural rendering by explicitly framing output pixels as functions of ray origins and directions derived from camera parameters, thereby inheriting DDF properties such as the relationship between depth gradients and surface normals [S1].

## How it works
**Inputs and Outputs:**
- Input: Continuous camera parameters $\Pi \in \Lambda_\Pi$ (intrinsics and extrinsics) and shape representation $\Lambda_S$ [S1].
- Output: An $m$-pixel image where each pixel contains a tuple $(d(p, v), \xi(p, v))$, with $d \in \mathbb{R}^+$ (depth) and $\xi \in [0, 1]$ (visibility) [S1].

**Data Structures:**
- Camera Parameters $\Pi$: Defines the ray origin $p$ (center of projection) and ray direction $v$ (pixel direction from $p$ into the scene) [S1].
- Shape $\Lambda_S$: The geometric representation queried by the renderer [S1].

**Steps:**
1. For each pixel, compute the ray origin $p$ and direction $v$ as fixed functions of $\Pi$ [S1].
2. Evaluate the neural renderer $f(\Pi, \Lambda_S)$ to obtain the depth $d$ and visibility $\xi$ for the ray $(p, v)$ [S1].
3. Assign the pixel value $(d(p, v), \xi(p, v))$ [S1].

**Formulas:**
- Pixel Value: $(d(p, v), \xi(p, v))$ [S1].
- Ray Origin: $p = \text{CenterOfProjection}(\Pi)$ [S1].
- Ray Direction: $v = \text{RayDirection}(p, \text{PixelIndex}, \text{Intrinsics}(\Pi))$ [S1].

**Parameters:**
- Not given in the sources.

**Edge Cases:**
- The formulation holds regardless of the internal architecture of $f$ [S1].
- For mesh data extraction, $(\xi, d)$ is obtained via ray-triangle intersection after computing $(p, v)$ [S1].

## Subtopics

### [CPDDF-based SI3DR](9291-cpddf-based-si3dr.md)
Uses a Conditional Probabilistic DDF for single-image 3D reconstruction, conditioning on a latent code for view-consistent rendering and explicit sampling. Choose when an implicit, differentiable shape representation is preferred over direct mesh or point cloud prediction.

## Operating characteristics
Not given in the sources.

## Background
Proposed to parallelize DDFs and improve the framework for differentiable neural rendering of geometric quantities [S1]. Builds on DDF properties to apply them to neural rendering frameworks [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
