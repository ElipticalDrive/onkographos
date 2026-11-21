# Scene Representation and Geometry Constraints


## Overview
This family of methods modifies the standard 3D Gaussian Splatting (3DGS) representation to improve geometric fidelity, enforce surface constraints, or extract explicit geometry, addressing the limitation that base 3DGS produces volumetric, soft primitives that lack precise surface alignment and watertight mesh compatibility [S1, 14095-mesh-extraction-and-representation.md]. It sets itself apart from sibling methods like [DeSplat](12925-desplat.md) and [360 Gaussian Splatting](12929-360-gaussian-splatting.md) by focusing on the structural definition of the primitives (e.g., planar alignment, SDF coupling, or discontinuity boundaries) rather than just rendering efficiency or compression.

## Shared principles
All methods in this family operate on the standard 3D Gaussian Splatting representation, where the scene is defined by a set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$. Each Gaussian $G_i$ is defined by its mean $\boldsymbol{\mu}_i$, covariance matrix $\boldsymbol{\Sigma}_i$ (derived from rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and view-dependent color $\mathbf{c}_i(\omega)$.

The rendering process evaluates the Gaussian density along a ray $\mathbf{r}(t) = \mathbf{o} + t\omega$ and integrates contributions. The Gaussian density function is:
$$ G(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})\right) $$

The core variation across methods lies in the definition, storage, and optimization of $\mathbf{c}_i(\omega)$ and potentially $\alpha_i$, moving beyond the fixed SH coefficients used in base 3DGS. Common strategies include:
1.  **Post-Processing Transformation:** Applying a learned affine transformation to the rendered image based on 3D position and view embedding [14740-color-and-appearance-modeling.md].
2.  **Factorized Illumination:** Separating scene lighting (via a tensor grid) from material properties (via BRDF features) to model specular effects [14740-color-and-appearance-modeling.md].
3.  **Image-Conditioned Modulation:** Using an MLP and per-image embeddings to modulate Gaussian color and opacity, enabling transient object removal and appearance synthesis [14740-color-and-appearance-modeling.md].
4.  **Neural Field Substitution:** Replacing per-Gaussian SH with a shared grid-based neural field to reduce parameters and improve generalization [14740-color-and-appearance-modeling.md].
5.  **Volumetric Attenuation:** Modifying the alpha-blending equation to incorporate medium properties, such as direct attenuation $B_d$ and backscatter parameters $B_\infty, B_b$, to model light transport in non-vacuum media [15148-physics-and-optical-modeling.md].

## Subtopics

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
This family modifies the standard 3DGS representation to improve geometric fidelity, rendering efficiency, or physical accuracy. It sets itself apart by addressing specific limitations of the base method: [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md) changes the radial basis function for better tail behavior; [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md) decouples visibility from opacity to allow aggressive culling; [Content-Aware Texturing](12606-content-aware-texturing-for-gaussian-splatting.md) adds dynamic textures to capture high-frequency details; [Directed Distance Field](12854-directed-distance-field-ddf-for-gaussian-splatting.md) provides a neural oracle for secondary rays; and [Atomized Proliferation](12924-atomized-proliferation.md) constrains Gaussian sizes for precise geometry. Choose this when you need to refine the primitive shape or density function itself.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Discontinuity-aware Gaussian Splatting (DisC-GS) enables 3DGS to represent and render sharp boundaries and discontinuities, addressing the limitation of standard 3DGS which tends to blur such features. It differs from sibling methods like [Planar Gaussian Splatting](16706-planar-gaussian-splatting.md) by explicitly modeling boundaries as cubic Bézier curves within the Gaussian's local subspace and applying a "pre-scissoring" operation during rendering, rather than relying solely on geometric constraints or mesh extraction. Choose this when rendering scenes with sharp occlusion boundaries or thin structures.

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
This family converts the volumetric, implicit representation of 3DGS into explicit geometric structures, primarily triangular meshes or dense point clouds, to enable compatibility with standard 3D graphics pipelines, mesh editing tools, and physics simulations. Unlike standard 3DGS, which renders via alpha-blended splats, these methods extract a discrete surface or point set that approximates the radiance field, allowing for operations such as texture mapping, rigging, and collision detection. The key distinction lies in whether the extraction is driven by surface-alignment regularization during training (e.g., SuGaR) or by post-hoc probabilistic sampling of the trained density field (e.g., Point Cloud Conversion). Choose this when you need a watertight mesh for downstream applications.

### [Appearance and Physical Modeling](15766-appearance-and-physical-modeling.md)
Appearance and Physical Modeling extends the standard 3DGS representation to support physically based rendering (PBR), relighting, and volumetric light transport, solving the limitations of base 3DGS which relies on fixed Spherical Harmonic (SH) coefficients that cannot accurately represent complex specularities, dynamic lighting, or non-vacuum media. It sets itself apart from sibling methods like [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by focusing on the radiance and material representation rather than geometric simplification, and distinguishes itself from [Physics and Optical Modeling](15148-physics-and-optical-modeling.md) by prioritizing surface material decomposition and relighting over volumetric attenuation equations, although both families modify the core Gaussian properties beyond simple color. Choose this for PBR or volumetric effects.

### [Neural SDF Inference via Gaussian Pulling](15776-neural-sdf-inference-via-gaussian-pulling.md)
Neural SDF Inference via Gaussian Pulling is a method for reconstructing continuous 3D surfaces from multi-view RGB images by jointly optimizing a set of 3D Gaussians and a neural Signed Distance Function (SDF). It solves the problem of inferring accurate geometry from 3DGS, which typically suffers from discreteness, sparseness, and off-surface drift, making direct SDF inference difficult. Unlike standard 3DGS which renders appearance without explicit surface constraints, this method dynamically aligns Gaussians to the SDF's zero-level set and constrains them to be thin disks tangent to the surface, enabling robust surface reconstruction. It differs from [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by explicitly coupling the Gaussian positions to a continuous implicit field via differentiable pulling, and from [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md) by learning the SDF directly rather than extracting meshes from static Gaussian clusters. Choose this for high-fidelity surface reconstruction from RGB inputs.

### [Planar Gaussian Splatting](16706-planar-gaussian-splatting.md)
Planar Gaussian Splatting (PGS) is an unsupervised neural rendering framework that reconstructs 3D geometry and explicitly parses 3D planar surfaces from multiple RGB images without requiring 3D labels or depth supervision. It extends the standard 3DGS representation by introducing discriminative 3D plane descriptors and enforcing local planar alignment constraints. Unlike sibling methods that focus on high-fidelity rendering or generic mesh extraction, PGS is designed to identify distinct 3D plane instances through a hierarchical Gaussian Mixture Tree (GMT). It sets itself apart by using 2D segmentation priors from foundation models (like SAM) to guide the learning of these descriptors, enabling robust planar reconstruction across different scenes. Choose this for scenes dominated by planar structures like indoor environments.

### [3D Gaussian Splatting for Surface Reconstruction](17501-3d-gaussian-splatting-for-surface-reconstruction.md)
3D Gaussian Splatting for Surface Reconstruction (3DGSR) is an implicit surface reconstruction method that integrates a neural Signed Distance Field (SDF) with the explicit 3DGS representation. It solves the problem of extracting high-quality, watertight meshes from 3DGS, which traditionally yields unstructured point clouds that are difficult to process with standard post-processing techniques like Poisson reconstruction. Unlike sibling methods that rely on post-processing or separate optimization stages, 3DGSR unifies the optimization of the Gaussian appearance and the implicit surface geometry end-to-end. It achieves this by coupling the SDF with the Gaussians via alignment constraints and regularizing the SDF using volumetric rendering consistency, enabling reconstruction quality superior to Gaussian-based methods like 2DGS while maintaining the rendering speed of 3DGS. Choose this for end-to-end surface reconstruction with high geometric accuracy.

### [Sparse2DGS](18424-sparse2dgs.md)
Sparse2DGS is a Gaussian Splatting method designed for surface reconstruction from sparse input views, addressing the ill-posed nature of geometric optimization when dense Structure-from-Motion points are unavailable. It distinguishes itself from standard 3DGS and its sibling [2DGS](12855-gaussian-representation-and-truncation.md) by initializing primitives with unsupervised Multi-View Stereo (MVS) points and employing geometric-prioritized enhancements—such as fixed feature splatting and selective updates—to prevent appearance overfitting and improve geometric learning. Choose this when training data is sparse.

#### Model-based depth estimation for volume parameter update
- **Step:** During optimization, use Eq. (6) to extract depth values $z$ from the current scene estimate. These depth values are then used to update the medium-parameter estimates ($B_\infty, B_b$) [S1].
- **Formula:** Depth extraction from 3DGS: $z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}$. Here $d_i$ is the depth of Gaussian $i$. This is computed at each pixel during rendering [S1].

#### Scale Constrained by Mesh Geometry
- **Definition:** A constraint to prevent discrepancies between Gaussian Splat scales and the scales of corresponding triangular mesh faces by bounding the relative scale coefficients [S2].
- **Formula:** $s = s_{rel} \odot s_0$, where $s$ is the final scale, $s_{rel} = (s_{rel,1}, s_{rel,2}) \in [s_{min}, s_{max}] \times [s_{min}, s_{max}]$ is the coefficient vector, $s_0$ is the predefined scale derived from the mesh faces, and $\odot$ denotes element-wise multiplication [S2].
- **Parameter:** $s_{min}, s_{max}$: The lower and upper bounds for the relative scale coefficients $s_{rel}$ [S2].

#### Normal Estimation via Back-propagation
- **Definition:** A method to solve for the normal attribute $n$ for each 3D Gaussian by optimizing it from an initial random vector via back-propagation, perceiving the depth of all Gaussians along a ray as a distribution [S3].
- **Step:** Determine the normal for each pixel by estimating the expectation of the depth distribution along the ray [S3].
- **Formula:** Normal consistency loss: $\mathcal {L}_{n}=\|\mathcal {N}-\tilde {\mathcal {N}}\|_2$ [S3].

#### Tangent Plane Constrained Gaussian Splatting
- **Definition:** An extension of 3DGS where Gaussians are constrained to a 2D tangent plane to resolve the conflict between volumetric Gaussians and surface representation, enabling accurate normal map estimation [S4].
- **Step:** Normal Rendering: Perform splatting of normals in the same manner as color splatting to generate a normal map, leveraging the tangent plane constraint [S4].
- **Formula:** A point $P$ on the 2D Gaussian plane with coordinates $(u, v)$ is defined as: $P(u, v) = p_k + s_u t_u u + s_v t_v v = H(u, v, 1, 1)^T$, where $H = [p_k, t_u, t_v, t_w]$ is the transformation matrix [S4].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 as a real-time explicit scene representation [G]. Subsequent work focused on extracting explicit geometry from these soft primitives, leading to families like Mesh Extraction and SDF-based reconstruction. Methods like DisC-GS and Planar GS address specific geometric artifacts (blurring, lack of planarity) inherent in the original formulation. The integration of SDFs (e.g., Neural SDF Inference, 3DGSR) represents a shift towards hybrid explicit-implicit representations for higher geometric fidelity.

## Sources
[S1] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S2] OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction, https://doi.org/10.1109/WACV61042.2026.00427, fetched 2026-10-30
[S3] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S4] Photometric Stereo Using Gaussian Splatting And Inverse Rendering, https://hal.science/hal-05553367v1, fetched 2026-10-26
