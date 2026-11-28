# Neural SDF Inference via Gaussian Pulling

Also called: Center Loss, Disk Loss, Disk Loss / Center Loss, Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set, Neural-Pull, Pulling to Centers, Pulling to Disks

## Overview
Neural SDF Inference via Gaussian Pulling is a method for reconstructing continuous surfaces from multi-view RGB images by dynamically aligning discrete 3D Gaussians to the zero-level set of a neural Signed Distance Function (SDF) [S1]. It solves the problem of inferring accurate geometry from 3D Gaussian Splatting (3DGS), which typically suffers from discreteness, sparseness, and off-surface drift, by jointly optimizing the Gaussians and the SDF using differentiable pulling operations [S1]. Unlike [3D Gaussian Splatting for Surface Reconstruction](17501-3d-gaussian-splatting-for-surface-reconstruction.md) and [GSurf](20857-gsurf.md), which often rely on volume rendering branches or monocular priors, this method seamlessly merges 3DGS with neural SDF learning by enforcing multi-view consistency and explicit geometric constraints (thin disks, tangency) to refine the SDF near the surface [S1].

## How it works
**Inputs:** Posed multi-view RGB images $\{v_i\}_{i=1}^I$.
**Outputs:** A set of optimized 3D Gaussians $\{g_j\}_{j=1}^J$ and a neural SDF network $f$.

**Data Structures:**
1.  **3D Gaussians $\{g_j\}$:** Each Gaussian $g_j$ is defined by:
    *   Mean (center) $\boldsymbol{\mu}_j \in \mathbb{R}^3$.
    *   Covariance matrix $\boldsymbol{\Sigma}_j \in \mathbb{R}^{3 \times 3}$, derived from rotation $\mathbf{R}_j$ and scale $\mathbf{S}_j$. The variances along the three principal axes are $r_1, r_2, r_3$.
    *   Opacity $\alpha_j \in [0, 1]$.
    *   Color $\mathbf{c}_j \in \mathbb{R}^3$.
2.  **Neural SDF $f$:** An MLP with 8 layers, 256 hidden units, and ReLU activation. It maps a query point $\mathbf{q} \in \mathbb{R}^3$ to a signed distance $s = f(\mathbf{q})$. It is initialized to represent a sphere [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize a set of 3D Gaussians $\{g_j\}$ to represent the scene geometry and color. Initialize the SDF network $f$ as a sphere [S1].
2.  **Gaussian Rendering (Splatting):** Render the learnable Gaussians $\{g_j\}$ into RGB images using the volume rendering equation [S1]:
    $$ C'(u, v) = \sum_{j=1}^{J} \mathbf{c}_j \cdot \alpha_j \cdot p_j(u, v) \prod_{k=1}^{j-1} (1 - \alpha_k \cdot p_k(u, v)) $$
    where $C'(u, v)$ is the rendered color at pixel $(u, v)$, and $p_j(u, v)$ is the 2D projection probability of the $j$-th Gaussian. At a 3D query point $\mathbf{q} = [x, y, z]$, the probability density is:
    $$ p_j(\mathbf{q}) = \exp\left(-\frac{1}{2} (\mathbf{q} - \boldsymbol{\mu}_j)^\top \boldsymbol{\Sigma}_j^{-1} (\mathbf{q} - \boldsymbol{\mu}_j)\right) $$
    [S1]
3.  **Differentiable Pulling (Aligning Gaussians):** Dynamically align each Gaussian $g_j$ to the zero-level set of the neural SDF $f$ using a differentiable pulling operation [S1]:
    *   Calculate the signed distance at the Gaussian center: $s_j = f(\boldsymbol{\mu}_j)$.
    *   Calculate the gradient (surface normal) at the center: $\nabla f(\boldsymbol{\mu}_j)$.
    *   Update the center to $\boldsymbol{\mu}'_j$ on the zero-level set:
        $$ \boldsymbol{\mu}'_j = \boldsymbol{\mu}_j - s_j \frac{\nabla f(\boldsymbol{\mu}_j)}{\|\nabla f(\boldsymbol{\mu}_j)\|} $$
    *   The pulled Gaussian $g'_j$ shares the same attributes ($\mathbf{c}_j, \alpha_j, \boldsymbol{\Sigma}_j$) as $g_j$ but has the new center $\boldsymbol{\mu}'_j$ [S1].
4.  **Gaussian Normal Estimation:** Determine the normal $\mathbf{n}_j$ of the Gaussian $g_j$ by identifying the axis with the minimum variance $\bar{r} = \min\{r_1, r_2, r_3\}$. The direction along this axis represents $\mathbf{n}_j$ [S1].
5.  **SDF Inference via Pulling Queries:** To refine the SDF, pull neighboring query points $\mathbf{q}$ onto the zero-level set using the SDF gradient:
    $$ \mathbf{q}' = \mathbf{q} - s \frac{\nabla f(\mathbf{q})}{\|\nabla f(\mathbf{q})\|} $$
    where $s = f(\mathbf{q})$ [S1].
6.  **Loss Computation:** Compute the total loss comprising rendering and geometric constraints [S1]:
    *   **Rendering Loss ($L_{\text{Splatting}}$):** Render the pulled Gaussians $\{g'_j\}$ into images $\{v'_i\}$. Minimize the difference between rendered and ground truth images:
        $$ L_{\text{Splatting}} = 0.8 \cdot L_1(v'_i, v_i) + 0.2 \cdot L_{D\text{-SSIM}}(v'_i, v_i) $$
        [S1]
    *   **Thin Disk Loss ($L_{\text{Thin}}$):** Encourage Gaussians to become thin disks tangent to the surface by penalizing large minimum variances:
        $$ L_{\text{Thin}} = \|\min\{r_1, r_2, r_3\}\|_1 $$
        [S1]
    *   **Tangent Constraint Loss ($L_{\text{Tangent}}$):** Align the Gaussian normal $\mathbf{n}_j$ with the SDF gradient at the pulled center $\boldsymbol{\mu}'_j$:
        $$ L_{\text{Tangent}} = 1 - \frac{\nabla f(\boldsymbol{\mu}'_j)}{\|\nabla f(\boldsymbol{\mu}'_j)\|} \cdot \mathbf{n}_j $$
        [S1]
    *   **Pulling Probability Loss ($L_{\text{Pull}}$):** For a query $\mathbf{q}$, identify the nearest pulled Gaussian $\bar{g}$ with center $\bar{\boldsymbol{\mu}}$. Maximize the probability that $\mathbf{q}'$ belongs to $\bar{g}$ by minimizing the negative log-probability:
        $$ L_{\text{Pull}}(\mathbf{q}'; \bar{\boldsymbol{\mu}}) = -\log \left( \exp\left(-\frac{1}{2} (\mathbf{q}' - \bar{\boldsymbol{\mu}})^\top \bar{\boldsymbol{\Sigma}}^{-1} (\mathbf{q}' - \bar{\boldsymbol{\mu}})\right) \right) $$
        where $\bar{g} = \arg\min_{g'_j} \|\boldsymbol{\mu}'_j - \mathbf{q}\|^2$ [S1].
    *   **Orthogonal Pulling Constraint Loss ($L_{\text{Orthogonal}}$):** Align the SDF gradient at query $\mathbf{q}$ with the normal $\bar{\mathbf{n}}$ of the nearest pulled Gaussian disk $\bar{g}$:
        $$ L_{\text{Orthogonal}} = 1 - \frac{\nabla f(\mathbf{q})}{\|\nabla f(\mathbf{q})\|} \cdot \bar{\mathbf{n}} $$
        [S1]
7.  **Joint Optimization:** Update the Gaussian attributes and SDF parameters by minimizing the total objective:
    $$ \min_{\{g_j\}, f} L_{\text{Splatting}} + \alpha L_{\text{Thin}} + \beta L_{\text{Tangent}} + \gamma L_{\text{Pull}} + \delta L_{\text{Orthogonal}} $$
    [S1]

**Parameters:**
*   Regularization weights: $\alpha=100$ (Thin), $\beta=0.1$ (Tangent), $\gamma=1$ (Pull), $\delta=0.1$ (Orthogonal) [S1].
*   Optimization iterations: 15,000 total. Densification stops and pulling/constraints are incorporated at 7,000 iterations [S1].
*   SDF Network: MLP, 8 layers, 256 hidden units, ReLU [S1].

**Edge Cases and Pitfalls:**
*   **Discreteness and Sparseness:** 3D Gaussians are discrete and may be sparsely distributed, making SDF inference difficult [S1]. The "pulling to disks" approach is preferred over "pulling to centers" because the disk loss gradient diminishes quadratically as the query approaches the center, providing higher tolerance for off-center queries and preventing the SDF from overfitting to individual Gaussian centers [S1].
*   **Background Artifacts:** Unlike TSDF or screened Poisson fusion, which may reconstruct large sky spheres in the background, this method avoids such artifacts by learning the SDF directly from the pulled Gaussians [S1].
*   **Scene Scale:** For large scenes, the scene must be split into parts for mesh extraction to bypass computational resource limitations [S1].

## Subtopics

### Volume Rendering with Surface Patch Sensing
A variant that infers SDFs from multi-view images by defining a local patch around the estimated ray-surface intersection using the SDF, allowing direct application of surface constraints [S2]. It samples points around the patch using a Gaussian distribution, applies mask and weight filtering, and pulls points to the zero-level set using gradients and SDF values [S2]. It computes depth, photometric, and surface fitting losses on the pulled points [S2]. This method outperforms NeuralRGBD, MonoSDF, DROID-SLAM, NICER-SLAM, and NICE-SLAM on ScanNet, Replica, and Tanks and Temples benchmarks, particularly on planar structures [S2]. It is evaluated using ground truth depth maps (ScanNet), monocular cues from Omnidata (Replica), and MonoSDF setup (Tanks and Temples) [S2].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 21.8 minutes on the DTU dataset and 37.6 minutes on the Tanks and Temples dataset, comparable to 2DGS (20.5m) and significantly faster than NeuS (~12h) [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the DTU dataset, the method achieves a mean Chamfer Distance of 0.52, outperforming 2DGS (0.54) and GSurfels (0.56) [S1]. On the Tanks and Temples dataset, it achieves a mean F-score of 0.84, surpassing NeuS (0.54) [S1].
*   **Strengths:** Recovers accurate, smooth, and complete surfaces with fine geometry details by leveraging multi-view consistency and explicit geometric constraints [S1]. Avoids background artifacts common in TSDF/Poisson fusion [S1].
*   **Limitations:** Performance depends on the quality of the initial 3D Gaussian distribution; sparseness and non-uniformity can still pose challenges, mitigated by the disk-pulling strategy [S1].
*   **When to use:** Suitable for multi-view surface reconstruction where high-quality mesh extraction is required from Gaussian splats, especially when avoiding background artifacts is critical [S1].
*   **When not to use:** Not suitable for single-view reconstruction without additional monocular priors (unless using the variant in Subtopics) or dynamic scenes not covered by the posed multi-view assumption [S1].

## Background
The method was proposed by Zhang et al. in 2024, titled "Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set" [S1]. It builds upon the principles of 3D Gaussian Splatting [30] and Neural-Pull [45], addressing the limitations of previous Gaussian-based reconstruction methods that lacked precise perception of continuous implicit representations [S1]. The approach was developed to seamlessly merge the rendering efficiency of 3DGS with the geometric accuracy of neural SDFs [S1].

## Sources
[S1] Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set, https://doi.org/10.48550/arXiv.2410.14189, fetched 2026-10-29
[S2] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
