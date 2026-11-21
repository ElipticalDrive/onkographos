# Surface Splatting and Lighting

Also called: Surface splatting, point-based surface rendering

## Overview
Surface Splatting and Lighting is a family of rendering techniques that reconstruct continuous, shaded surfaces from unstructured point cloud samples by projecting each point as a spatially extended primitive (a splat) carrying a reconstruction kernel. It solves the problem of sparse dot cloud aliasing and transparency artifacts inherent in raw point rendering by ensuring hole-free reconstructions even for moderate sampling densities. Unlike forward splatting methods that apply lighting during rasterization, variants like [Deferred Surface Splatting](1072-deferred-surface-splatting.md) decouple geometric projection from lighting calculation, allowing multiple light sources to be applied efficiently in screen space without re-projecting geometry.

## Shared principles
The family operates on a set of point cloud samples $P = \{p_i\}$, where each sample $p_i$ has a position $\mathbf{p}$, a color $\mathbf{c}$, and optionally a normal $\mathbf{n}$. Each sample is converted into a splat $S_j$ defined by a center $c_j$, tangent vectors $\mathbf{u}_j, \mathbf{v}_j$, and radii $r_u, r_v$.

The rendering process generally involves:
1. **Projection:** Transforming the object-space splat (often an elliptical disk or quadrilateral) into screen space.
2. **Rasterization:** Rasterizing the splat footprint, which acts as a low-pass filter to suppress aliasing.
3. **Accumulation:** Blending the contributions of overlapping splats. This typically involves computing a weight $w$ based on the distance from the splat center and accumulating attributes (color, normal, depth) additively.
4. **Lighting:** Computing illumination. This can be done per-splat or in a final screen-space pass.

## Subtopics

### [Deferred Surface Splatting](1072-deferred-surface-splatting.md)
Converts point clouds into continuous surfaces using a deferred pipeline with a G-buffer. It decouples projection from lighting, allowing efficient multi-light support in screen space. Choose this when multiple lights or complex shading models are required without re-projecting geometry.

### [Phong Splatting](2802-phong-splatting.md)
Assigns a linearly varying normal field to each elliptical splat, enabling per-pixel Phong shading. It is projectively correct and solves piecewise constant shading artifacts. Choose this for high-quality, physically-based shading of point clouds with fewer primitives than triangle meshes.

### [EWA Surface Splatting](8199-ewa-surface-splatting.md)
Uses band-limited reconstruction kernels (surface splats) to efficiently suppress aliasing and guarantee hole-free reconstructions under arbitrary transformations. It builds on [Surface Splatting and Lighting](9275-surface-splatting-and-lighting.md) and is chosen for high-quality, band-limited rendering of dense point clouds.

## Background
Surface splatting was introduced as a method to render point samples by superimposing band-limited reconstruction kernels in the image domain. It efficiently suppresses aliasing artifacts and guarantees hole-free reconstructions even for moderate sampling densities. The technique has been extended to support various lighting models, including deferred shading and Phong shading, to address the visual quality limitations of simple point sprites.

## Sources
