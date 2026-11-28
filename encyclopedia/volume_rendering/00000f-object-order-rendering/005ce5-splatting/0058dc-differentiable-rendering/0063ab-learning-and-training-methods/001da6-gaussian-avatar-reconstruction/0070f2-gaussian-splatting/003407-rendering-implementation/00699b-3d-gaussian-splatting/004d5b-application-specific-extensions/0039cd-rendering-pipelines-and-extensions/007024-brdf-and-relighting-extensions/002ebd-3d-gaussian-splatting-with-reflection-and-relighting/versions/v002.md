# 3D Gaussian Splatting with Reflection and Relighting

Also called: Gaussian-Shader, 3DGS-DR, IRGS

## Overview
This method extends standard 3D Gaussian Splatting (3DGS) by incorporating classical reflection and shading concepts to address limitations with highly reflective surfaces and enable dynamic relighting [S1]. Unlike standard 3DGS, which bakes lighting into spherical harmonics via an emission-absorption model, this approach allows for physically based rendering (PBR) by augmenting Gaussian kernels with environmental information [S1, S2]. It leverages the natural normal estimation provided by the shortest axis of the Gaussian kernel to achieve high-quality rendering of reflective objects with training efficiency [S2].

## How it works
**Inputs:**
*   3D Gaussian primitives (position, covariance, opacity, base color/SH coefficients).
*   Environment map for direct lighting.
*   Camera parameters.

**Outputs:**
*   Rendered image with dynamic specular shading.

**Data Structures:**
*   **Gaussian Kernel:** Augmented with normal vectors derived from the covariance matrix. The normal is defined as the unit vector corresponding to the shortest axis of the Gaussian kernel [S2].
*   **BRDF Parameters:** Some variants embed albedo, roughness, and surface normals directly into each Gaussian [S1].

**Steps:**
1.  **Normal Estimation:** For each Gaussian, compute the normal vector $\mathbf{n}$ from the covariance matrix $\Sigma$. The normal corresponds to the eigenvector associated with the smallest eigenvalue (shortest axis) [S2].
2.  **Projection and Rasterization:** Project the 3D Gaussians to 2D screen space. Assign primitives to screen tiles and determine visibility based on projected extents [G].
3.  **Lighting Calculation:**
    *   **Direct Lighting:** Reconstruct direct lighting using the environment map and the estimated normal $\mathbf{n}$ [S1].
    *   **Indirect Lighting:** Model indirect lighting using spherical harmonics coefficients, distinct from the baked lighting in standard 3DGS [S1].
4.  **Secondary Ray Tracing:** Integrate secondary rays (reflections and refractions) spawned upon primary ray interactions with Gaussians using ray tracing techniques to model interreflections [S1]. This step distinguishes the method from standard 3DGS, which relies only on primary rays [S1].
5.  **Compositing:** Accumulate color using front-to-back alpha blending, incorporating the calculated specular and diffuse components [G].

**Formulas:**
*   **Normal from Covariance:** Let $\Sigma$ be the covariance matrix of a Gaussian. The normal $\mathbf{n}$ is the eigenvector corresponding to the smallest eigenvalue $\lambda_{min}$ [S2].
*   **Constraint Projection:** In optimization contexts, constraints are projected using $\Delta x = M^{-1} \nabla C^\top(x) \Delta \lambda$ [S2].

**Parameters:**
*   Environment map resolution: Not given in the sources.
*   Ray tracing depth: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Rotational Deformation:** For solid objects, an anisotropy loss is used to cap the stretching ratio during training to mitigate rendering artifacts induced by rotational deformation [S2].
*   **Fluid Surfaces:** For scattered fluid droplets, normal information is estimated via depth volume rendering from the current camera pose [S2].

## Operating characteristics
*   **Time Complexity:** Training is more efficient than NeRF-based PBR methods, which suffer from time-consuming training and slow rendering speed for normal estimation [S2].
*   **Memory Complexity:** Increased memory usage due to the storage of augmented Gaussian attributes (normals, BRDF parameters) and environment maps [S1].
*   **Strengths:** Naturally offers good normal estimation as the shortest axis of the Gaussian kernel, enabling high-quality rendering of reflective objects [S2]. Supports dynamic specular shading synthesis [S2].
*   **Limitations:** Requires ray tracing for secondary rays, which may impact performance compared to pure rasterization [S1].

## Background
Standard 3DGS uses an emission-absorption model, effectively baking lighting conditions into spherical harmonics coefficients, which limits performance on highly reflective surfaces and prevents relighting [S1]. Methods such as Gaussian-Shader [17], 3DGS-DR [18], and IRGS [19] were developed to incorporate classical reflection and shading concepts into the 3DGS framework [S1]. This approach synergizes with constraint-projection-based simulation, allowing for versatile handling of physical problems such as fluids, rigid bodies, or hyperelastic objects [S2].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
