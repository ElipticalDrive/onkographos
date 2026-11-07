# Local Ambient Occlusion
## Overview
Local Ambient Occlusion (LAO) methods estimate the blocking of indirect light in volume rendering by analyzing the local density distribution or sampling stochastic rays around a point, rather than relying on global geometry or hard shadow maps. Unlike siblings that use Signed Distance Fields (SDF) [2527-ambient-occlusion-via-sdf-geometry.md] or precomputed voxel buffers [6646-voxel-ambient-occlusion.md], LAO methods typically compute occlusion on-the-fly or via lightweight precomputation tailored to the volume's statistical properties or stochastic sampling, providing softer, more volumetric shadow appearances [4502-local-ambient-occlusion-lao.md].
## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment.

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa). The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [S1, unofficial][S6, unofficial].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$. The shading contribution $C_{shade}$ is computed using standard local illumination models (e.g., Lambertian, Phong) [S1, unofficial]:
    $$ C_{shade} = k_d (\mathbf{n} \cdot \mathbf{l}) C_{diffuse} + k_s (\mathbf{n} \cdot \mathbf{h})^\alpha C_{specular} $$
    where $\mathbf{n} = \mathbf{g}/\|\mathbf{g}\|$ [S1, unofficial].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source. This can be done via ray casting (checking for density/geometry along the path) or via precomputed buffers (shadow maps, illumination buffers) [S1, unofficial][S2].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light). The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [S1, unofficial][S2].
## Subtopics

### [Local Ambient Occlusion for Volume Rendering](4300-local-ambient-occlusion-for-volume-rendering.md)
Calculates AO by integrating opacity over the local vicinity's normal distribution, accelerated by approximating the Transfer Function (TF) as piecewise linear segments. It assumes CT values follow a normal distribution to estimate occlusion probability, reducing computation from iterating over all CT values to the number of TF segments. Choose when high performance is needed for CT data with known statistical distributions.

### [Local Ambient Occlusion (LAO)](4502-local-ambient-occlusion-lao.md)
Estimates local occlusion by sampling the volume along random directions around the light vector, weighted by distance. It differs from standard shadow mapping by integrating density along stochastic rays to approximate blocking of indirect light, providing softer, volumetric shadows. Choose when a soft, noise-based approximation of ambient occlusion is acceptable and precomputation is undesirable.

### [Feature-Driven Ambient Occlusion](8903-feature-driven-ambient-occlusion.md)
Computes AO separately for identified features (layers) to prevent overdarkening of internal layers caused by inter-occlusions from neighboring features. It neglects inter-occlusions between features while preserving auto-occlusions. Choose for multi-layer volumes where preserving the visibility of internal structures is critical.

### Lambertian Lighting with AO and Shadows
Combines Lambertian diffuse shading with Local Ambient Occlusion and Soft Shadows contributions to produce the final volume color.
- Compute normal from gradient: `n = normalize(grad);`
- Compute half vector: `halfVector = normalize(vLight - position);`
- Compute Lambertian term: `nDotL = max(dot(n, halfVector), 0.0);`
- Get base color from transfer function: `color = getColor(value, gradientMagnitude(grad));`
- Mix color with LAO darkening: `color = mix(color, color * vec4(0.15, 0.18, 0.32, 1.0), LAOContribution.r * uLAOWeight);`
- Mix color with Shadow darkening: `color = mix(color, color * vec4(0.15, 0.18, 0.22, 1.0), uSoftShadowsContribution.r * uShadowsWeight);` [S1, unofficial]

### Deferred Shading for Ambient Occlusion
Renders world position and surface normal pixels of the image from the current point of view to a buffer, then executes AO computation in the fragment shader on each pixel using the underlying normal and position. This avoids evaluating the computation for hidden geometry [S2].

## Background
The concept of local ambient occlusion in volume rendering has evolved from basic density-based shading to more sophisticated statistical and stochastic methods. Early implementations focused on simple density integration [S1, unofficial]. The use of deferred shading for AO in voxel-based rendering was introduced to improve efficiency by avoiding hidden geometry calculations [S2]. The specific statistical approach using normal distributions for CT data was developed to accelerate rendering by reducing the complexity of transfer function integration [S1, unofficial].
## Sources
[S1] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
