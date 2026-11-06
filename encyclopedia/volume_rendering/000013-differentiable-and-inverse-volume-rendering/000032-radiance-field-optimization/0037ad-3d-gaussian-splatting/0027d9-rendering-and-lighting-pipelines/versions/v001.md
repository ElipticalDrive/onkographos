# Rendering and Lighting Pipelines

Also called: 3DGS Rendering, Gaussian Splatting Compositing, Volume Lighting in 3DGS

## Overview
This family defines the differentiable rendering pipelines that convert a set of 3D Gaussian primitives (or their variants) into 2D images by projecting them onto the camera plane and compositing them. It solves the problem of efficiently rasterizing soft, view-dependent primitives while supporting inverse rendering tasks such as lighting estimation and physical parameter optimization. What sets these methods apart from standard rasterization is the use of alpha blending or weighted-sum compositing over sorted or spatially hashed primitives, often augmented with volume-based occlusion or explicit geometric intersection for sharp features.

## Shared principles
All methods in this family share the core mechanism of projecting 3D primitives into 2D screen space and accumulating color based on depth order or spatial locality.

1.  **Projection:** Primitives are transformed from world space to camera space. For Gaussians, this involves computing the 2D covariance matrix in screen space using the Jacobian of the projection matrix and the 3D covariance [S2].
2.  **Rasterization:** Primitives are assigned to screen tiles. Visibility is determined by bounding boxes or projected extents.
3.  **Compositing:** Color is accumulated using front-to-back alpha blending or order-independent techniques. The standard alpha blending equation is:
    $$ C_{out} = C_{in} (1 - \alpha) + C_{prim} \alpha $$
    where $C_{in}$ is the accumulated color, $\alpha$ is the primitive's opacity, and $C_{prim}$ is the primitive's color [S2].
4.  **Sorting:** Most methods require sorting primitives by depth within tiles to ensure correct front-to-back compositing, although some variants (like Duplex-GS) use spatial hashing or cell-level sorting to approximate this or enable order-independent transparency [S1, unofficial].

## Subtopics

### [Duplex-GS](1988-duplex-gs.md)
A dual-hierarchy framework for real-time order-independent Gaussian Splatting. It combines alpha-blending with reformulated Weighted-Sum Rendering (WSR) to eliminate popping and transparency artifacts. Unlike vanilla OIT, it reintroduces physical constraints via cell-level transmittance decay, allowing early termination. It differs from Sort-free GS by using cell-level sorting for physically grounded transmittance.

### [LinPrim Differentiable Rendering Pipeline](2195-linprim-differentiable-rendering-pipeline.md)
Replaces Gaussian ellipsoids with convex polyhedra (octahedra/tetrahedra) to enable efficient volumetric rendering with bounded geometry. It solves the problem of reconstructing sharp geometric features by allowing gradients to backpropagate through explicit ray-triangle intersections (Möller-Trumbore), distinguishing it from methods relying on implicit density fields.

### [GsplatVolumeLighting](2203-gsplatvolumelighting.md)
Integrates volume occlusion directly into the per-splat lighting evaluation loop. It ray-marches through a 3D density texture to compute transmittance for shadows, combined with per-light attenuation. Unlike methods treating volume as a post-process, it shares the 3D density texture with the splat evaluation.

### [NeILF Rendering and Loss Calculation](2920-neilf-rendering-and-loss-calculation.md)
Decomposes Gaussian Splatting features into diffuse, specular, and ambient components using a Blinn-Phong model. It supports inverse rendering via loss functions on depth, normals, and color sparsity.

### [4DGS-Volume-Cloud Two-Stage Relighting](4101-4dgs-volume-cloud-two-stage-relighting.md)
A two-stage pipeline for physically parameterized volume clouds. Stage 1 trains physical parameters (geometry, extinction, albedo, scattering) under a single sun. Stage 2 freezes these and trains a global environment network for arbitrary sun direction relighting and sky atmospheric scattering.

### Opacity to Alpha Conversion
A minor variant involving the extraction and conversion loop for alpha values from raw opacity parameters to ensure they are in the valid range for blending [S1, unofficial].

### Alpha Blending for Non-Pinhole Cameras
After obtaining 2D Gaussian function values via adapted projection for non-pinhole cameras, alpha blending is performed to generate the final image, identical to the original 3D-GS compositing step [S2].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S2]. The rendering pipeline relies on efficient rasterization of 2D Gaussians projected from 3D. Subsequent works have adapted this pipeline for differentiable rendering, volume lighting, and physical parameter estimation. The "unofficial" sources [S1, unofficial] provide implementation details for JSplat, a Java-based 3DGS renderer.

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S2] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
