# Background Parameterization and Inversion
Also called: Inverted Sphere Parameterization, Background Inversion for Unit Sphere

## Overview
This family of methods enables Neural Radiance Fields to render unbounded scenes with complex backgrounds (e.g., 360° captures) by mapping points outside a bounded volume into its interior, thereby avoiding the parameterization failure of standard NeRF which assumes the entire scene fits within a fixed bounding box [S1, unofficial]. It sets itself apart from its parent [NeRF++](6198-nerf.md) by providing specific mathematical mappings for the outer volume, and from siblings like [DoubleNeRF](7391-doublenerf.md) by explicitly separating foreground and background representations through geometric inversion rather than just network architecture changes.

## Shared principles
All methods in this family share the core mechanism of partitioning the scene space into a bounded inner volume (containing cameras and foreground objects) and an unbounded outer volume (containing the background environment). To handle the unbounded nature of the background, they apply a spatial parameterization that maps 3D points $\mathbf{x} = (x, y, z)$ with radius $r > 1$ to a bounded representation. The rendering integral is split into two parts: integration inside the inner volume and integration in the outer volume, combined via the standard volumetric rendering equation [S1, unofficial].

## Subtopics

### [Inverted Sphere Parameterization](5573-inverted-sphere-parameterization.md)
Inverted Sphere Parameterization is a spatial parameterization scheme that allows Neural Radiance Fields to model 360° captures of unbounded scenes by separately representing foreground and background content. It solves the parameterization failure of standard NeRF, which assumes the entire scene fits within a bounded volume, leading to either missing background elements or loss of detail due to limited sampling resolution in large-scale scenes. It sets itself apart from its parent [NeRF++](6198-nerf.md) by providing the specific mathematical mapping for the outer volume. See [Inverted Sphere Parameterization](5573-inverted-sphere-parameterization.md) for implementation details.

### [Background Inversion for Unit Sphere](11463-background-inversion-for-unit-sphere.md)
This method enables volume rendering of scenes with unbounded or complex backgrounds by mapping points outside a unit sphere into its interior, thereby eliminating the need for foreground masks during optimization. It sets itself apart from sibling methods like [Inverted Sphere Parameterization](5573-inverted-sphere-parameterization.md) by explicitly using a separate density voxel grid and shallow MLP for the inverted background, rather than relying on a single continuous field or point-based representations. See [Background Inversion for Unit Sphere](11463-background-inversion-for-unit-sphere.md) for implementation details.

### Implicit Fields Modeling with NeRF++
A method to model unbounded backgrounds in 3D-aware generation by partitioning the scene into foreground and background, where the background is modeled with an additional network using inverted sphere parameterization to handle complex, unbounded scenes (e.g., 360-degree images) [S1, unofficial]. It uniformly samples $G$ background points in an inverse depth range of $[1/R, 0)$ to represent where the background starts, with $R = 2.0$ in experiments [S1, unofficial]. The background network takes inputs of background latent codes and the transformed 3D point $\mathbf{x}_0 = (x/r, y/r, z/r, 1/r)$, where $r = \sqrt{x^2 + y^2 + z^2}$ [S1, unofficial].

## Background
Originating from [NeRF++](6198-nerf.md) (Zhang et al., 2020), this approach addresses the limitation of standard NeRF in handling unstructured images with complex, unbounded backgrounds, such as those from 360-degree directions (e.g., CompCars dataset) [S1, unofficial]. It was further adopted and refined in works like StyleNeRF for 3D-aware image synthesis [S1, unofficial].

## Sources
[S1] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
