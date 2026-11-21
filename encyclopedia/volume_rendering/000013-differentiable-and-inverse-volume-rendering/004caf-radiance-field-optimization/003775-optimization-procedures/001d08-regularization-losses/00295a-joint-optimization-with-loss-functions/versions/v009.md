# Joint Optimization with Loss Functions

Also called: Joint Optimization of Surfel Parameters and HDR Environment Map

## Overview
Joint optimization with loss functions simultaneously updates scene geometry, material properties, and illumination parameters by minimizing a composite objective function. It solves the ill-posed inverse rendering problem by regularizing the solution space through geometric and photometric constraints, distinguishing itself from single-parameter optimization by enforcing consistency across depth layers, surface normals, and image appearance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Ground-truth images $\mathbf{I}^*$, optional ground-truth masks $\mathbf{M}^*$, camera poses, and initial surfel parameters (positions $\mathbf{c}$, normals $\mathbf{n}$, scales, albedo, metallic, roughness) and an HDR environment map.
- **Outputs:** Optimized surfel parameters and HDR environment map.

**Data Structures:**
- **Surfel Parameters:** Positions $\mathbf{c} \in \mathbb{R}^3$, tangent vectors parameterized as unit quaternions $\mathbf{q}$, log-space scales $(\log s_x, \log s_y)$, albedo $\rho$, metallic $m$, roughness $r$.
- **HDR Environment Map:** Equirectangular map of resolution $16 \times 32$ storing radiance values $L$.
- **Spatial Data Structure:** A KNN (K-Nearest Neighbors) index built on surfel positions, rebuilt periodically and cached between optimization updates.

**Composite Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of four terms:
$$
\mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{sil}} \mathcal{L}_{\text{sil}} + \lambda_{\text{cons}} \mathcal{L}_{\text{cons}} + \lambda_{\text{nc}} \mathcal{L}_{\text{nc}}
$$
where $\lambda_{\text{sil}}, \lambda_{\text{cons}}, \lambda_{\text{nc}}$ are scalar weights [S1, unofficial].

1.  **Photometric Loss ($\mathcal{L}_{\text{photo}}$):**
    Combines $\ell_1$ reconstruction and Structural Similarity (SSIM). For HDR targets, both rendered image $\hat{\mathbf{I}}$ and ground-truth $\mathbf{I}^*$ are transformed via log-space and sRGB transfer before loss computation to prevent gradient domination by bright pixels [S1, unofficial].
    $$
    \mathcal{L}_{\text{photo}} = (1 - \lambda_{\text{ssim}}) \|\hat{\mathbf{I}} - \mathbf{I}^*\|_1 + \lambda_{\text{ssim}} (1 - \text{SSIM}(\hat{\mathbf{I}}, \mathbf{I}^*))
    $$
    where $\lambda_{\text{ssim}}$ is the weighting factor [S1, unofficial].

2.  **Silhouette Loss ($\mathcal{L}_{\text{sil}}$):**
    Supervises total accumulated coverage using soft Intersection-over-Union (IoU). It is active only during early training before adaptive density control begins [S1, unofficial].
    $$
    \mathcal{L}_{\text{sil}} = \text{IoU}(1 - \bar{T}_{K+1}, \mathbf{M}^*)
    $$
    where $\bar{T}_{K+1}$ is the residual transmittance and $\mathbf{M}^*$ is the ground-truth mask [S1, unofficial].

3.  **Depth Consolidation Loss ($\mathcal{L}_{\text{cons}}$):**
    Penalizes spurious depth layers by enforcing geometric thinness. It is the average over all pixels $\mathcal{P}$ of inter-layer and intra-layer terms [S1, unofficial]:
    $$
    \mathcal{L}_{\text{cons}} = \frac{1}{|\mathcal{P}|} \sum_{p \in \mathcal{P}} (\mathcal{L}_{\text{inter-c}} + \mathcal{L}_{\text{intra-c}})
    $$
    - **Inter-layer term ($\mathcal{L}_{\text{inter-c}}$):** Penalizes weight allocation across distant layers.
      $$
      \mathcal{L}_{\text{inter-c}} = \sum_{i < j} w_i w_j |z_i - z_j|
      $$
      where $w_i, w_j$ are compositing weights and $z_i, z_j$ are mean depths of layers $i, j$ [S1, unofficial].
    - **Intra-layer term ($\mathcal{L}_{\text{intra-c}}$):** Penalizes depth spread within layers.
      $$
      \mathcal{L}_{\text{intra-c}} = \sum_i w_i^2 \sigma_i^2
      $$
      where $\sigma_i^2$ is the depth variance of layer $i$ [S1, unofficial].

4.  **Normal Consistency Loss ($\mathcal{L}_{\text{nc}}$):**
    Penalizes angular deviation between a surfel's normal and its orientation-aware nearest neighbors in 3D space [S1, unofficial]:
    $$
    \mathcal{L}_{\text{nc}} = \frac{\sum_i \sum_{j \in \mathcal{N}_i} w_{ij} (1 - \mathbf{n}_i \cdot \mathbf{n}_j)}{\sum_i \sum_{j \in \mathcal{N}_i} w_{ij}}
    $$
    where $\mathcal{N}_i$ are nearest neighbors with compatible normal orientation, $\mathbf{n}_i, \mathbf{n}_j$ are normals, and $w_{ij} = \exp(-\|\mathbf{c}_i - \mathbf{c}_j\|^2 / 2\sigma^2)$ are distance-dependent weights derived from local intersample spacing [S1, unofficial].

**Optimization Steps:**
1.  **Initialization:** Initialize surfel parameters (positions, quaternions, log-scales, albedo, metallic, roughness) and HDR environment map ($16 \times 32$ equirectangular).
2.  **Gradient Computation:** Compute gradients of $\mathcal{L}$ with respect to all parameters using the Adam optimizer [S1, unofficial].
3.  **Parameter Update:** Update parameters using the computed gradients.
4.  **Spatial Structure Update:** Periodically rebuild and cache the KNN spatial data structure for normal consistency calculation [S1, unofficial].
5.  **Schedule Management:** Disable $\mathcal{L}_{\text{sil}}$ once adaptive density control begins. Activate $\mathcal{L}_{\text{cons}}$ and $\mathcal{L}_{\text{nc}}$ according to the training schedule [S1, unofficial].

**Parameters:**
- $\lambda_{\text{ssim}}$: SSIM weighting factor. Not given in the sources.
- $\lambda_{\text{sil}}, \lambda_{\text{cons}}, \lambda_{\text{nc}}$: Scalar weights for respective losses. Not given in the sources.
- $\sigma$: Parameter for distance-dependent weights in $\mathcal{L}_{\text{nc}}$, derived from local intersample spacing. Not given in the sources.
- Environment Map Resolution: $16 \times 32$ equirectangular. [S2]

**Edge Cases and Pitfalls:**
- **Gradient Domination:** In HDR training, bright pixels can dominate gradients. Mitigated by applying log-space and sRGB transfer to both rendered and ground-truth images before loss computation [S1, unofficial].
- **Spurious Depth Layers:** Without regularization, multi-layer compositing may distribute surfels across incorrect depths. Mitigated by $\mathcal{L}_{\text{cons}}$ [S1, unofficial].
- **Oversmoothing:** Screen-space normal regularizers may oversmooth geometry. Mitigated by using 3D normal consistency on surfel primitives instead [S1, unofficial].
- **Illumination Absorption:** Long second-stage optimization may cause albedo to absorb illumination information, degrading relighting quality. Mitigated by limiting optimization duration [S2].

## Subtopics

### HDR Environment Map Optimization
Models distant lighting as a high-dynamic-range spherical environment map in equirectangular coordinates. It sets itself apart by optimizing radiance values directly with solid-angle compensation for non-uniform texel area, preserving high-frequency angular detail for sharp cast shadows [S2]. It is chosen when accurate relighting is required. Uses normal-based hemisphere sampling to halve the number of rays per pixel compared to full-sphere sampling. After hemisphere culling and empty-space removal, there are 512 candidate directions. Map Resolution is $16 \times 32$ equirectangular, limiting recoverable illumination bandwidth to the object's BRDF lobes [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** A full second-stage optimization iteration takes approximately 1.25 s/it. Final PBR relighting renders at about 0.83 s/image. Forward relighting spends about 55% of its time on visibility queries and 44% on BRDF/environment-map integration. OptiX visibility pass takes approximately 436 ms at evaluation resolution, corresponding to about 127 MRays/s [S2].
- **Strengths:** Enforces geometric thinness and normal consistency; handles HDR lighting; stable gradient-based optimization of illumination [S1, unofficial][S2].
- **Limitations:** Compact environment map limits recoverable illumination bandwidth; long optimization may degrade relighting quality [S2].
- **When to use:** Object-level inverse rendering requiring geometry reconstruction, novel view synthesis, and novel scene relighting [S1, unofficial].
- **When not to use:** When high-bandwidth illumination recovery is critical and the object's BRDF lobes are insufficient to constrain the environment map [S2].

## Background
Originated in the context of 3D Surface Splatting for Inverse Rendering (3DSS) and VisibilitySplat. 3DSS jointly optimizes surfel parameters and HDR environment maps using composite losses [S1, unofficial]. VisibilitySplat uses ray-traced occlusion for relighting with 3D Gaussian Splatting, optimizing an HDR environment map [S2].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] VisibilitySplat: Ray-Traced Occlusion for Relighting with 3D Gaussian Splatting, https://doi.org/10.24132/csrn.2026-e17, fetched 2026-10-26
