# Neural Articulated Radiance Field (NARF)
Also called: NARFP, Part-Wise NARF

## Overview
Neural Articulated Radiance Field (NARF) models articulated objects by decomposing them into $P$ rigid parts, each represented by a separate Rigidly Transformed Neural Radiance Field (RT-NeRF) [S1]. Unlike implicit methods that learn pose-dependent deformations within a single network, NARF explicitly computes global rigid transformations via forward kinematics and combines part-wise outputs using a differentiable softmax weighting scheme [S1]. This approach avoids the "part dependency" issue found in earlier methods like P-NeRF, where a single network might learn erroneous correlations between unrelated parts [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera ray $\mathbf{r}$, articulated object pose configuration $\{\mathbf{T}_0, \boldsymbol{\zeta}, \boldsymbol{\theta}\}$ (root transform, global shape variations, joint angles), and number of parts $P$.
*   **Outputs:** Rendered color $\mathbf{C}(\mathbf{r})$, optional foreground mask $\mathbf{M}(\mathbf{r})$, and optional segmentation mask $\mathbf{S}^i(\mathbf{r})$.

**Data Structures:**
*   **RT-NeRF Models:** $P$ separate neural networks $F_{\Theta^i}^{l^i, \boldsymbol{\zeta}}$ for $i=1 \dots P$. Each network maps local coordinates to density $\sigma$ and feature/color $\mathbf{h}, \mathbf{c}$.
*   **Pose Parameters:**
    *   $\mathbf{T}_0$: Global root transformation.
    *   $\boldsymbol{\zeta$: Global shape variation parameters (e.g., limb length).
    *   $\boldsymbol{\theta}$: Joint angles.
*   **Transformation:** Rigid transformation $l_i = \{\mathbf{R}_i, \mathbf{t}_i\}$ for each part $i$, derived from forward kinematics.

**Steps:**

1.  **Compute Global Rigid Transformations:**
    For each part $i \in \{1, \dots, P\}$, compute the global rigid transformation $l_i = \{\mathbf{R}_i, \mathbf{t}_i\}$ from the pose configuration $\{\mathbf{T}_0, \boldsymbol{\zeta}, \boldsymbol{\theta}\}$ using forward kinematics [S1].

2.  **Transform Ray and Position:**
    For a query 3D point $\mathbf{x}$ and viewing direction $\mathbf{d}$ along ray $\mathbf{r}$, transform them into the local coordinate system of each part $i$:
    $$
    \mathbf{x}^i = \mathbf{R}_i^{-1}(\mathbf{x} - \mathbf{t}_i)
    $$
    $$
    \mathbf{d}^i = \mathbf{R}_i^{-1}\mathbf{d}
    $$
    [S1]

3.  **Query Part-Wise RT-NeRFs:**
    For each part $i$, query the corresponding RT-NeRF $F_{\Theta^i}^{l^i, \boldsymbol{\zeta}}$ to obtain density $\sigma^i$ and color $\mathbf{c}^i$. The input to the network includes the encoded local position $\gamma(\mathbf{x}^i)$, encoded local direction $\gamma(\mathbf{d}^i)$, encoded shape parameters $\gamma(\boldsymbol{\xi}^i)$ (if applicable), and encoded global shape variations $\gamma(\boldsymbol{\zeta})$ [S1]:
    $$
    (\mathbf{c}^i, \sigma^i) = F_{\Theta^i}^{l^i, \boldsymbol{\zeta}}(\gamma(\mathbf{x}^i), \gamma(\mathbf{d}^i), \gamma(\boldsymbol{\xi}^i), \gamma(\boldsymbol{\zeta}))
    $$
    [S1]

4.  **Combine Densities and Colors:**
    Combine the outputs from all $P$ parts into a global density $\sigma$ and color $\mathbf{c}$ using a differentiable weighted sum via softmax over the densities, controlled by a temperature parameter $\tau$ [S1]:
    $$
    \sigma = \frac{\sum_{i=1}^P \exp(\sigma^i/\tau)\sigma^i}{\sum_{i=1}^P \exp(\sigma^i/\tau)}
    $$
    $$
    \mathbf{c} = \frac{\sum_{i=1}^P \exp(\sigma^i/\tau)\mathbf{c}^i}{\sum_{i=1}^P \exp(\sigma^i/\tau)}
    $$
    [S1]
    *Note:* The softmax ensures differentiability, allowing gradients to flow to all RT-NeRF models during training, unlike a non-differentiable max operation [S1].

5.  **Volume Rendering:**
    Perform standard volume rendering using the combined density $\sigma$ and color $\mathbf{c}$ to generate the rendered color $\mathbf{C}(\mathbf{r})$ [S1]:
    $$
    \mathbf{C}(\mathbf{r}) = \sum_{j=1}^N T_j (1 - \exp(-\sigma_j \delta_j)) \mathbf{c}_j
    $$
    where $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_k)$ is the accumulated transmittance, $\delta_j$ is the distance between samples, and $\sigma_j, \mathbf{c}_j$ are the density and color at sample $j$ [S1].

6.  **Optional: Foreground Mask and Segmentation:**
    *   **Foreground Mask:** Estimate the foreground mask $\mathbf{M}(\mathbf{r})$ as the integral of opacity along the ray [S1]:
        $$
        \mathbf{M}(\mathbf{r}) = \sum_{j=1}^N T_j (1 - \exp(-\sigma_j \delta_j))
        $$
        [S1]
    *   **Segmentation Mask:** Render a segmentation image indicating which part $i$ contributed most to each pixel. First, determine the part index with the highest density at each sample $j$:
        $$
        s_j(\mathbf{r}) = \arg \max_{i \in \{1, \dots, P\}} \{\sigma_j^i\}
        $$
        [S1]
        Then, accumulate the opacity weighted by the indicator function for part $i$:
        $$
        \mathbf{S}^i(\mathbf{r}) = \sum_{j=1}^N T_j (1 - \exp(-\sigma_j \delta_j)) \cdot \mathbb{I}(s_j(\mathbf{r}) == i)
        $$
        [S1]

**Parameters:**
*   $\tau$: Temperature parameter for the softmax function in Eq. 15. Typical values are not specified in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Zero Density Problem:** Training is dominated by zero-density samples because a surface point belongs to only one part, resulting in zero density for all other parts. This can make training inefficient as parts with small densities do not significantly affect the final value but still require computation [S1].
*   **Non-Differentiable Max:** Using a max operation to select the highest density is not differentiable, preventing gradient flow to all RT-NeRF models. The softmax approximation is required for effective training [S1].
*   **Computational Cost:** The computational cost is proportional to the number of object parts $P$, which limits the representation capacity for complex articulated objects with many parts [S1].

## Subtopics

### Holistic NARF
Holistic NARF (NARFH) combines the inputs of the RT-NeRF models from Part-Wise NARF and feeds them as a whole into a single NeRF model for direct regression of final density and color [S1]. Unlike Part-Wise NARF, NARFH does not satisfy Part Dependency because all parameters are considered for each 3D location, and object part segmentation masks cannot be generated using Eq. 18 [S1]. However, NARFH has computational cost almost constant to the number of object parts and naturally avoids the zero density problem [S1].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of object parts $P$ in Part-Wise NARF (NARFP), as each part requires a separate RT-NeRF query [S1]. Holistic NARF (NARFH) has computational cost almost constant to the number of object parts [S1].
*   **Memory Complexity:** Proportional to $P$ for NARFP, as it stores $P$ separate RT-NeRF models [S1]. NARFH uses a single model, reducing memory overhead relative to $P$ [S1].
*   **Strengths:** Avoids part dependency issues by explicitly modeling rigid transformations via forward kinematics [S1]. Allows for novel view synthesis, depth estimation, and body part segmentation [S2, unofficial]. Differentiable combination allows gradient flow to all parts [S1].
*   **Limitations:** NARFP suffers from high computational cost for complex objects with many parts and inefficient training due to zero-density samples [S1]. NARFH loses the ability to generate explicit part segmentation masks based on density dominance [S1].
*   **When to Use:** Use NARFP when explicit part segmentation and strict part dependency avoidance are required, and the number of parts is small. Use NARFH when computational efficiency is critical and part-specific segmentation is not needed.

## Background
NARF was introduced in 2021 by [NSLH21] [S1]. It builds on the concept of RT-NeRF, which models rigidly transformed object parts [S1]. The method addresses limitations in previous approaches like P-NeRF, which suffered from part dependency issues where the network learned erroneous dependencies between parts [S1]. NARF is trained using pose-annotated images, decomposing the articulated object into rigid parts with local coordinate systems and global shape variations [S2, unofficial]. The converged NARF can be used for novel view synthesis, depth estimation, and body part segmentation [S2, unofficial].

## Sources
[S1] Neural Articulated Radiance Field, https://doi.org/10.1109/iccv48922.2021.00571, fetched 2026-10-26
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
