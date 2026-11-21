# IRON

Also called: IRON, Inverse Rendering by Optimizing Neural SDFs and Materials, PIR (Photometric Inverse Rendering), Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization

## Overview
IRON is a neural inverse rendering pipeline that reconstructs high-quality 3D geometry and spatially varying materials from photometric images (multi-view images captured with a collocated flashlight and moving camera) [S1]. It outputs triangle meshes and material textures compatible with standard graphics pipelines [S1]. Unlike standard NeRF variants that optimize density and color directly, IRON uses Signed Distance Fields (SDFs) for geometry and separate neural networks for diffuse, specular, and roughness parameters [S1]. It employs a hybrid optimization scheme: a first stage using volumetric radiance field rendering to recover correct topology, followed by a second stage using edge-aware physics-based surface rendering to refine geometry and disentangle materials from lighting [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of photometric images captured with a flashlight co-located with the camera [S1]. Assumptions include opaque objects, no ambient light, and ignored shadows/inter-reflections [S1].
*   **Output:** A triangle mesh and material textures (diffuse albedo, specular albedo, roughness) [S1].

**Data Structures:**
The method uses four Multi-Layer Perceptrons (MLPs) with positional encoding $\gamma(\cdot)$ [S1].
1.  **Neural SDF ($F_{\Theta_s}$):**
    *   Input: 3D position $\mathbf{x}$.
    *   Output: Tuple $(S, \mathbf{f})$, where $S \in \mathbb{R}$ is the SDF value and $\mathbf{f} \in \mathbb{R}^{256}$ is a local geometric feature descriptor [S1].
2.  **Neural Diffuse Albedo ($F_{\Theta_\beta}$):**
    *   Input: Position $\mathbf{x}$, surface normal $\mathbf{n}$, view direction $\mathbf{v}$ (or normal $\mathbf{n}$ in Stage 2), and feature descriptor $\mathbf{f}$ [S1].
    *   Output: Diffuse albedo $\beta \in \mathbb{R}^3$ [S1].
3.  **Neural Specular Albedo ($F_{\Theta_\kappa}$):**
    *   Input: Position $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$.
    *   Output: Specular albedo $\kappa \in \mathbb{R}^3$ [S1].
4.  **Neural Roughness ($F_{\Theta_\alpha}$):**
    *   Input: Position $\mathbf{x}$, surface normal $\mathbf{n}$, and feature descriptor $\mathbf{f}$.
    *   Output: Roughness $\alpha \in \mathbb{R}$ (small values indicate shiny surfaces, large values indicate matte surfaces) [S1].

**Optimization Stages:**

**Stage 1: Volumetric Radiance Field Rendering**
This stage initializes the topology (number and location of holes) [S1].
1.  Treat the neural diffuse albedo network $F_{\Theta_\beta}(\mathbf{x}, \mathbf{n}, \mathbf{v}, \mathbf{f})$ as a view-dependent neural radiance field by substituting the view direction $-\mathbf{d}$ for the second normal argument [S1].
2.  Optimize $F_{\Theta_s}$ and $F_{\Theta_\beta}$ using volumetric rendering [S1].
3.  The pixel color $C(\mathbf{r})$ is computed via the standard volumetric rendering integral:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    Here, $\sigma_i$ is derived from the SDF $S$, $\mathbf{c}_i = F_{\Theta_\beta}(\mathbf{x}_i, \mathbf{n}_i, -\mathbf{d}_i, \mathbf{f}_i)$, and $\delta_i$ is the step size [S1].
4.  If this stage is skipped, optimization frequently diverges or gets stuck in local minima with incorrect topology [S1].

**Stage 2: Edge-aware Physics-based Surface Rendering**
This stage refines geometric details and factorizes materials from lighting [S1].
1.  Jointly optimize $F_{\Theta_s}$, $F_{\Theta_\beta}$, $F_{\Theta_\kappa}$, $F_{\Theta_\alpha}$, and light intensity $L$ [S1].
2.  Use physics-based surface rendering. The light is modeled as a point source:
    $$
    L_i(\mathbf{x}) = \frac{L}{\|\mathbf{x} - \mathbf{o}\|^2}
    $$
    where $L$ is light intensity, $\mathbf{x}$ is the surface location, and $\mathbf{o}$ is the light/camera position [S1].
3.  **Edge Sampling:** To handle depth discontinuities, a novel edge sampling algorithm is used [S1].
    *   Input: Ray-surface intersection $\hat{\mathbf{x}}$.
    *   Hyperparameters: Max steps $K$, step size $\epsilon$, threshold $\delta$.
    *   Algorithm:
        ```python
        def locate_edge_points(x_hat, o, K, epsilon, delta):
            xt = x_hat
            for i in range(1, K + 1):
                vec = xt - o
                dist = np.linalg.norm(vec)
                if dist < 1e-8:
                    return x_hat
                dir_vec = vec / dist
                n_t = compute_normal(xt)
                if np.dot(dir_vec, n_t) < delta:
                    return xt
                else:
                    projection = n_t - np.dot(dir_vec, n_t) * dir_vec
                    xt = xt + epsilon * projection
            return "NOT FOUND"

        def compute_normal(x):
            return compute_sdf_gradient(x) # Computes gradient of SDF network [G]
        ```
4.  The rendering equation accounts for diffuse and specular components using the optimized material parameters [S1].

**Parameters:**
*   $K$: Max steps for edge search (Not given in the sources) [S1].
*   $\epsilon$: Step size for edge search (Not given in the sources) [S1].
*   $\delta$: Threshold for edge detection (Not given in the sources) [S1].
*   $L$: Light intensity (Optimized scalar) [S1].

**Pitfalls:**
*   Skipping Stage 1 leads to divergence or incorrect topology [S1].
*   The method assumes ideal collocated camera-lighting; deviations cause self-shadowing errors if not handled (addressed in PIR) [S2].
*   Inter-reflections are ignored in the original IRON, leading to inaccuracies in concave regions [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Not given in the sources [S1].
*   **Strengths:** Recovers correct topology via volumetric initialization; produces mesh-compatible outputs; handles spatially varying materials [S1].
*   **Limitations:** Assumes opaque objects; ignores ambient light, self-shadows, and inter-reflections (original version); requires photometric images with collocated flashlight [S1].
*   **When to use:** When high-quality mesh and texture extraction from photometric image sets is required, and computational resources allow for two-stage optimization [S1].
*   **When not:** When self-shadows or inter-reflections are significant and cannot be ignored, or when only environmental lighting is available [S2].

## Background
IRON was proposed by Kai Zhang, Fujun Luan, Zhengqi Li, and Noah Snavely from Cornell University, Adobe Research, and Google Research [S1]. It was published in CVPR 2022 [S1]. The method builds on neural SDF representations (e.g., IDR, NeuS) and neural radiance fields (NeRF) [S1]. It addresses the limitations of mesh-based inverse rendering by using neural SDFs for flexibility and compactness [S1]. A later extension, PIR (Photometric Inverse Rendering), addresses IRON's shortcomings regarding self-shadows and inter-reflections by jointly optimizing light position and using DINO feature regularization [S2].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
[S2] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
