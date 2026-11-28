# Clipping Geometry Illumination
Also called: Clipping Surface Shading

## Overview
Clipping Geometry Illumination is a shading technique used during volume rendering to provide realistic lighting at the boundaries of user-defined clipping geometry [S1, unofficial]. While standard volume rendering relies on the volume's internal gradient for surface normals, this method substitutes the gradient with the geometric normal of the clipping boundary at the intersection point [S1, unofficial]. This distinguishes it from sibling methods like [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md) which may suppress data entirely, or [Pre-integrated Clipping Boundary Handling](7867-pre-integrated-clipping-boundary-handling.md) which focuses on integration artifacts; Clipping Geometry Illumination specifically addresses the visual discontinuity of lighting at the cut plane [S1, unofficial].

## How it works
**Inputs:**
- Volume dataset $V$ with scalar values.
- Transfer function $T$ mapping scalar values to color and opacity.
- Clipping geometry defined by a set of planes or surfaces.
- Light source position and direction.
- Camera position and view direction.

**Outputs:**
- Final pixel color with correct illumination at clipping boundaries.

**Data Structures:**
- Ray structure: origin $\mathbf{o}$, direction $\mathbf{d}$, length $L$.
- Sample point structure: position $\mathbf{p}$, scalar value $s$, opacity $\alpha$, color $\mathbf{c}$.
- Normal vector $\mathbf{n}$: computed either from volume gradient $\nabla V(\mathbf{p})$ or from clipping geometry.

**Steps:**
1. **Ray Casting:** For each pixel, cast a ray through the volume.
2. **Sampling:** Advance along the ray, sampling the volume at discrete intervals.
3. **Boundary Detection:** At each sample point $\mathbf{p}$, determine if the point lies on the boundary of the clipping geometry.
   - If the point is inside the volume and not on a clipping boundary, compute the normal $\mathbf{n}$ using the volume gradient $\nabla V(\mathbf{p})$.
   - If the point is on the boundary of the clipping geometry, compute the normal $\mathbf{n}$ using the geometric surface normal of the clipping geometry at $\mathbf{p}$ [S1, unofficial].
4. **Illumination Calculation:** Compute the lighting model (e.g., Phong) using the determined normal $\mathbf{n}$, light direction $\mathbf{l}$, and view direction $\mathbf{v}$.
   - Diffuse term: $k_d (\mathbf{n} \cdot \mathbf{l})$
   - Specular term: $k_s (\mathbf{r} \cdot \mathbf{v})^{\alpha_{spec}}$ where $\mathbf{r}$ is the reflection vector.
5. **Compositing:** Accumulate color and opacity using front-to-back or back-to-front compositing, applying the lighting result to the sampled color [S1, unofficial].

**Formulas:**
- Gradient normal: $\mathbf{n}_{vol} = \frac{\nabla V(\mathbf{p})}{\|\nabla V(\mathbf{p})\|}$
- Geometric normal: $\mathbf{n}_{clip} = \text{Normal of clipping geometry at } \mathbf{p}$
- Final Normal: $\mathbf{n} = \begin{cases} \mathbf{n}_{vol} & \text{if not on clipping boundary} \\ \mathbf{n}_{clip} & \text{if on clipping boundary} \end{cases}$

**Parameters:**
- Clipping geometry definition: Typically defined by plane equations $ax + by + cz + d = 0$ or mesh surfaces.
- Lighting coefficients: $k_d, k_s, \alpha_{spec}$ as per standard Phong model.

**Edge Cases and Pitfalls:**
- **Precision:** Detecting exact boundary intersection requires careful tolerance handling to avoid gaps or overlaps between volume gradient normals and geometric normals [G].
- **Degenerate Input:** If the clipping geometry is not watertight or has self-intersections, normal computation may be ambiguous [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** Same as standard volume rendering, $O(N)$ where $N$ is the number of samples per ray. The additional normal computation is negligible.
- **Memory Complexity:** $O(1)$ additional memory per ray for storing the clipping normal.
- **Performance:** No significant performance impact compared to standard gradient-based shading.
- **Strengths:** Provides visually consistent lighting at clipping boundaries, avoiding the "unlit" or "noisy" appearance that can occur with gradient estimation at sharp cuts.
- **Limitations:** Requires explicit definition of clipping geometry normals. Does not handle internal volume gradients at the boundary, which may appear discontinuous if the geometry normal does not align with the volume structure [S1, unofficial].

## Background
Clipping Geometry Illumination was described by Weiskopf et al. in TVCG 2003 [S1, unofficial]. It is part of the broader field of direct volume rendering techniques aimed at improving visual quality during interactive exploration. The method addresses the specific problem of lighting discontinuities when parts of the volume are made transparent via clipping geometry [S1, unofficial].

## Sources
[S1] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
