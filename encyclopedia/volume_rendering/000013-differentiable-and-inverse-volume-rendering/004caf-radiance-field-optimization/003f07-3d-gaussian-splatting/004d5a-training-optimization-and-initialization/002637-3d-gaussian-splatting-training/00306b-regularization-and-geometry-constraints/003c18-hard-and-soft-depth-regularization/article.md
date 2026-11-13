# Hard and Soft Depth Regularization


## Overview
Hard and Soft Depth Regularization is a technique for 3D Gaussian Splatting that constrains geometry by encouraging the spatial reshaping (movement of centers) of primitives rather than deforming their shapes, thereby preventing the detail loss and blurred appearance caused by forcing anisotropic Gaussians to fit smooth monocular depth [S1]. It sets itself apart from standard depth regularization methods by decoupling the optimization of center positions and opacities through two distinct rendered depth maps, allowing coarse depth fitting and fine color appearance to be balanced independently [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\mathcal{G}$ with parameters $\theta = \{\mu, s, q, \alpha, c\}$ (centers, scales, rotations, opacities, colors); a set of training views with ground-truth images and monocular depth estimates $D_{mono}$; a regularization weight $\lambda_{reg}$.
*   **Outputs:** Updated Gaussian parameters $\theta'$; a scalar regularization loss $\mathcal{L}_{reg}$.

**Data Structures:**
*   **Gaussian State:** Each Gaussian $i$ has a center $\mu_i \in \mathbb{R}^3$, scale $s_i \in \mathbb{R}^3$, rotation quaternion $q_i \in \mathbb{R}^4$, opacity $\alpha_i \in [0,1]$, and spherical harmonic coefficients for color $c_i$.
*   **Depth Maps:** Two rendered depth maps are generated per training step:
    1.  **Hard Depth Map ($D_{hard}$):** Computed using the standard alpha-blending process but weighted to emphasize the center positions. Specifically, it represents the expected depth of the Gaussian centers.
    2.  **Soft Depth Map ($D_{soft}$):** Computed using the standard alpha-blending process weighted by opacity, representing the effective depth of the radiance field.

**Numbered Steps:**
1.  **Freeze Shape Parameters:** During the depth regularization step, freeze the scale parameters $s$ and rotation quaternions $q$ of all Gaussians. Only gradients for centers $\mu$ and opacities $\alpha$ are computed and applied [S1].
2.  **Render Depth Maps:**
    *   Render $D_{hard}$ by treating each Gaussian as a point source at $\mu_i$ with weight proportional to its opacity $\alpha_i$. This map captures the geometric distribution of centers.
    *   Render $D_{soft}$ using the standard differentiable volume rendering equation, which accounts for the full Gaussian shape and opacity distribution [S1].
3.  **Compute Losses:**
    *   Compute the **Hard Depth Loss** $\mathcal{L}_{hard}$ between $D_{hard}$ and the ground-truth or monocular depth estimate $D_{mono}$. This loss drives the movement of centers $\mu$ to align with the coarse geometry.
    *   Compute the **Soft Depth Loss** $\mathcal{L}_{soft}$ between $D_{soft}$ and $D_{mono}$. This loss drives the adjustment of opacities $\alpha$ to refine the appearance and fill gaps without moving centers excessively [S1].
4.  **Combine and Optimize:**
    *   The total regularization loss is $\mathcal{L}_{reg} = \lambda_{hard} \mathcal{L}_{hard} + \lambda_{soft} \mathcal{L}_{soft}$.
    *   Compute gradients $\nabla_{\mu} \mathcal{L}_{reg}$ and $\nabla_{\alpha} \mathcal{L}_{reg}$.
    *   Update $\mu$ and $\alpha$ using an optimizer (e.g., Adam). $s$ and $q$ remain unchanged [S1].

**Formulas:**
*   **Alpha Blending:** For a pixel $p$, the rendered color $C_p$ and depth $D_p$ are computed by sorting Gaussians $i=1 \dots N$ along the ray and blending:
    $$ C_p = \sum_{i=1}^{N} T_i \alpha_i c_i, \quad D_p = \sum_{i=1}^{N} T_i \alpha_i d_i $$
    where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance, and $d_i$ is the depth of the $i$-th Gaussian center (for $D_{hard}$) or the effective depth contribution (for $D_{soft}$).
*   **Depth Loss:** Typically, a combination of L1 and gradient losses is used:
    $$ \mathcal{L}_{depth} = \lambda_1 \| D_{rendered} - D_{mono} \|_1 + \lambda_{grad} \| \nabla D_{rendered} - \nabla D_{mono} \|_1 $$
    [S1] notes that standard scale-invariant losses are insufficient, leading to the use of Global-Local Depth Normalization (see Subtopics).

**Parameters:**
*   $\lambda_{reg}$: Weight for the total regularization term. Typical values are small (e.g., $10^{-3}$ to $10^{-2}$) relative to the RGB loss [S1].
*   $\lambda_{hard}, \lambda_{soft}$: Balancing weights for hard and soft depth losses. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Shape Deformation:** Applying depth regularization directly to NeRF-style MLPs or un-frozen Gaussian shapes forces the primitive geometry to conform to smooth monocular depth, which conflicts with complex color appearance and causes detail loss [S1]. Freezing $s$ and $q$ is critical [S1].
*   **Scale-Invariant Losses:** Standard scale-invariant depth losses align depth maps to a fixed global scale, ignoring minor local depth changes. This leads to noisy primitive distributions and reconstruction failures in regions with intricate textures [S1]. Global-Local Depth Normalization is required to address this [S1].

## Subtopics

### Global-Local Depth Normalization
Global-Local Depth Normalization is a depth loss normalization technique that processes depth patches at both global and local scales to prevent scale-invariant losses from overlooking minor depth variations, thereby improving detailed geometry reshaping in texture-rich regions [S1]. It sets itself apart from standard scale-invariant losses by combining global scale preservation with local scale amplification. Choose this when using monocular depth estimates in scenes with complex textures where standard losses fail to capture fine geometric details [S1].
*   **Steps:**
    1.  Compute depth loss using normalized depth patches [S1].
    2.  Apply global scale normalization to preserve absolute scene scale knowledge [S1].
    3.  Apply local scale normalization to depth patches to amplify and refocus the loss on small local depth errors [S1].
    4.  Combine both normalized losses to guide gradient updates during optimization [S1].
*   **Pitfall:** Standard scale-invariant depth losses align all depth maps to a fixed global scale, which ignores minor local depth changes. This leads to noisy primitive distributions and reconstruction failures in regions with intricate textures [S1].

### Geometric Regularization via Bilateral Filter
Geometric Regularization via Bilateral Filter is a regularization loss that enforces coherent motion by grouping Gaussians in image space using a bilateral filter approach, avoiding the dampening of independent motion caused by spatial KNN [S2]. It sets itself apart from spatial KNN-based methods by weighting neighbors based on color, depth, and spatial distance in image space. Choose this for scenes with oscillatory or complex motion where independent parts (e.g., leaves) should not be forced to move together [S2].
*   **Steps:**
    1.  For a training image $I$ at time $t_I$, sample another time $t'$ [S2].
    2.  Compute the 6D screw-axis representation of the transformation between $t_I$ and $t'$ for each Gaussian [S2].
    3.  Compute the geometric loss $\mathcal{L}_{geo}$ as the weighted sum of squared distances between screw-axis representations of neighboring pixels [S2].
*   **Formula:**
    $$ \mathcal{L}_{\text{geo}} = \sum_{p' \in \mathcal{N}(p)} k_{c,c'} k_{d,d'} k_{p,p'} \|s_p - s_{p'}\|^2 $$
    where $k_{c,c'}$ is color similarity, $k_{d,d'}$ is depth similarity, $k_{p,p'}$ is spatial distance weight, and $s_p, s_{p'}$ are screw-axis representations [S2].
*   **Pitfall:** Unlike standard bilateral filters, the loss is not divided by the sum of weights because the goal is comparison, not energy conservation [S2].

### Isotropic Regularization
Isotropic Regularization maintains coherent 3D structure and optimizes newly inserted Gaussians using current and random past keyframes to prevent forgetting [S3, unofficial]. It sets itself apart by penalizing elongation along the viewing direction, encouraging sphericality. Choose this for continuous SLAM or sparse-view settings where Gaussians tend to become highly elongated along the viewing ray [S3, unofficial].
*   **Steps:**
    1.  Use keyframes in current window $W_k$ and two random past keyframes $W_r$ per iteration [S3, unofficial].
*   **Formula:**
    $$ E_{iso} = \sum_{i=1}^{|\mathcal{G}|} \|\mathbf{s}_i - \tilde{\mathbf{s}}_i \cdot \mathbf{1}\|_1 $$
    where $\mathbf{s}_i$ are scaling parameters, $\tilde{\mathbf{s}}_i$ is their mean, and $\mathbf{1}$ is a vector of ones. This penalizes elongation along viewing direction [S3, unofficial].

### Position Regularization
Position Regularization is a loss function applied during PBR training to prevent dramatic position shifts of vertices caused by large gradients in HDR space, enforcing Gaussians to stay near their original mesh triangle positions [S4, unofficial]. It sets itself apart by anchoring Gaussians to initial mesh positions. Choose this for inverse rendering or PBR training where mesh consistency is critical [S4, unofficial].
*   **Formula:**
    $$ L_{sr} = ||\mu - \mu_{init}||^2_2 $$
    where $\mu$ is the current position of the Gaussian bound to the extracted mesh, and $\mu_{init}$ is the initial position of that Gaussian stored separately [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The method adds the cost of rendering two additional depth maps per training step. The complexity is dominated by the rasterization step, which is $O(N)$ per pixel where $N$ is the number of Gaussians. The normalization steps (Global-Local) add a small constant overhead per patch [S1].
*   **Memory Complexity:** Requires storing two additional depth maps per view per iteration, and the initial mesh positions for Position Regularization [S1][S4, unofficial]. The freeze of $s$ and $q$ reduces the number of gradients to store and update, slightly reducing memory overhead for optimizer states [S1].
*   **Measured Performance:** DNGaussian (using HSDR) achieves competitive quality and superior details compared to state-of-the-art methods in sparse-view settings on LLFF, Blender, and DTU datasets. It offers a 25× reduction in training time and over 3000× faster rendering speed compared to NeRF-based methods [S1].
*   **Scaling:** The method scales well with the number of Gaussians as the rasterization is efficient. However, the bilateral filter approach (Geometric Regularization) scales with the number of neighbors in the image space, which can be optimized with fixed-radius neighborhoods [S2].
*   **Strengths:** Prevents detail loss and blurred appearance; balances coarse depth fitting with fine color appearance; effective in sparse-view settings; reduces training time [S1].
*   **Limitations:** Requires monocular depth estimates; freezing shape parameters may limit the ability to correct shape errors; sensitive to the quality of monocular depth [S1].
*   **When to Use:** Sparse-view reconstruction; scenes with complex textures; when monocular depth is available and reliable [S1].
*   **When Not to Use:** Dense-view settings where depth regularization is less critical; scenes where shape deformation is necessary to fit geometry [S1].

## Background
Hard and Soft Depth Regularization was introduced in the DNGaussian paper [S1]. It addresses the limitations of existing depth regularization strategies for NeRFs, which commonly employ depth to regularize the entire model, creating a potential geometry conflict in Gaussian fields [S1]. The method was developed to enable spatial reshaping by encouraging movement among primitives, striking a balance between fitting complex color appearance and smooth coarse depth [S1].

## Sources
[S1] DNGaussian: Optimizing Sparse-View 3D Gaussian Radiance Fields with Global-Local Depth Normalization, https://doi.org/10.1109/cvpr52733.2024.01963, fetched 2026-10-26
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S3] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S4] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
