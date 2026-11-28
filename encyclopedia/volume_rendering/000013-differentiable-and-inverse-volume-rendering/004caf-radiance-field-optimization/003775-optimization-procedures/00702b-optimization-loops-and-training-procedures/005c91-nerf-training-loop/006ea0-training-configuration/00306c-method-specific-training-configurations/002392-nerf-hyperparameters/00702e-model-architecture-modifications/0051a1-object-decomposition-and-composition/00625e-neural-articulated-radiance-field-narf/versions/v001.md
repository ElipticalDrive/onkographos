# Neural Articulated Radiance Field (NARF)
Also called: NARFP, Part-Wise NARF

## Overview
Neural Articulated Radiance Field (NARF) models articulated objects by decomposing them into rigid parts, each represented by a separate Rigidly Transformed Neural Radiance Field (RT-NeRF) [S1]. Unlike sibling methods that may learn implicit dependencies or holistic representations, NARF explicitly estimates rigid transformations via forward kinematics to ensure each part maintains a consistent radiance field across different 3D pose configurations [S1]. It avoids the "part dependency" issue found in earlier approaches like P-NeRF by combining part-wise estimates into a unified global density and color using a differentiable softmax operation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of pose-annotated images; a kinematic 3D pose configuration defined by a root transformation $T_0$, joint angles $\theta$, and shape parameters $\zeta$ [S1][S2, unofficial].
*   **Outputs:** Rendered color $C(\mathbf{r})$ for a camera ray $\mathbf{r}$; optionally, a foreground mask $M(\mathbf{r})$ and part segmentation masks $\mathbf{S}^i(\mathbf{r})$ [S1].

**Data Structures:**
The method utilizes $P$ separate RT-NeRF models, denoted as $F_{\Theta^i}^{l^i, \boldsymbol{\zeta}}$ for $i \in \{1, \dots, P\}$ [S1]. Each model takes encoded local position, local viewing direction, local shape parameters, and global shape parameters as input and outputs density $\sigma^i$ and feature vector $\mathbf{h}^i$ (which is subsequently decoded to color $\mathbf{c}^i$) [S1].

**Steps:**

1.  **Compute Global Rigid Transformations:**
    For each rigid part $i$, compute the global rigid transformation $l_i = \{R^i, \mathbf{t}^i\}$ using forward kinematics based on the input pose configuration $\{T_0, \zeta, \theta\}$ [S1].

2.  **Transform Coordinates:**
    For a global 3D location $\mathbf{x}$ and viewing direction $\mathbf{d}$, compute the local coordinates for each part $i$:
    $$
    \mathbf{x}^i = R^{i-1}(\mathbf{x} - \mathbf{t}^i)
    $$
    $$
    \mathbf{d}^i = R^{i-1}\mathbf{d}
    $$
    [S1]

3.  **Query Part-Wise RT-NeRFs:**
    For each part $i$, query the corresponding RT-NeRF to obtain density and color estimates. The input to the $i$-th RT-NeRF consists of the encoded local position $\gamma(\mathbf{x}^i)$, encoded local direction $\gamma(\mathbf{d}^i)$, encoded local shape parameters $\gamma(\boldsymbol{\xi}^i)$, and encoded global shape parameters $\gamma(\boldsymbol{\zeta})$ [S1]:
    $$
    F_{\Theta^i}^{l^i, \boldsymbol{\zeta}} : (\gamma(\mathbf{x}^{l^i}), \gamma(\mathbf{d}^{l^i}), \gamma(\boldsymbol{\xi}^i), \gamma(\boldsymbol{\zeta})) \rightarrow (\mathbf{c}^i, \sigma^i)
    $$
    [S1]

4.  **Combine Densities and Colors:**
    Combine the estimates $\{\sigma^i, \mathbf{c}^i\}_{i=1}^P$ from all $P$ parts into a single global density $\sigma$ and color $\mathbf{c}$ using a differentiable weighted sum via softmax over the densities, controlled by temperature parameter $\tau$ [S1]:
    $$
    \sigma = \frac{\sum_{i=1}^P \exp(\sigma^i/\tau)\sigma^i}{\sum_{i=1}^P \exp(\sigma^i/\tau)}, \quad \mathbf{c} = \frac{\sum_{i=1}^P \exp(\sigma^i/\tau)\mathbf{c}^i}{\sum_{i=1}^P \exp(\sigma^i/\tau)}
    $$
    [S1]
    *Note:* The max operation for selecting the highest density is not differentiable, so the softmax approximation is used to allow gradient flow to all RT-NeRF models during training [S1].

5.  **Volume Rendering:**
    Perform standard volume rendering using the combined density $\sigma$ and color $\mathbf{c}$ to generate the rendered color $C(\mathbf{r})$ along the camera ray $\mathbf{r}$ [S1].

6.  **Optional Outputs:**
    *   **Foreground Mask:** Estimate the foreground mask $M(\mathbf{r})$ as the integral of opacity along the ray [S1]:
        $$
        M(\mathbf{r}) = \sum_{j=1}^N T_j(1 - \exp(-\sigma_j \delta_j))
        $$
        [S1]
    *   **Segmentation Masks:** Render a segmentation image $\mathbf{S}^i(\mathbf{r})$ for part $i$ by identifying the part with the greatest density at each sample point $j$ [S1]:
        $$
        s_j(\mathbf{r}) = \arg \max_i \{\sigma_j^i | i \in [1, P]\}
        $$
        $$
        \mathbf{S}^i(\mathbf{r}) = \sum_{j=1}^N T_j(1 - \exp(-\sigma_j \delta_j))(s_j(\mathbf{r}) == i)
        $$
        [S1]

**Parameters:**
*   $\tau$: Temperature parameter for the softmax function. Typical values are not specified in the sources [S1].
*   $P$: Number of rigid object parts.

**Edge Cases and Pitfalls:**
*   **Zero Density Samples:** Training is dominated by zero density point samples because a surface point belongs to only one part, resulting in zero density for all other parts. This leads to inefficient training as parts with small densities do not significantly affect the final value but still require calculation [S1].
*   **Non-Differentiable Max:** Using a hard max operation to select the highest density prevents gradient flow to all RT-NeRF models. The softmax approximation is required for effective training [S1].
*   **Computational Cost:** The cost is proportional to $P$, limiting representation capacity for complex articulated objects with many parts [S1].

## Subtopics

### Holistic NARF
Holistic NARF (NARFH) combines the inputs of the RT-NeRF models from Part-Wise NARF and feeds them as a whole into a single NeRF model for direct regression of final density and color [S1]. Unlike Part-Wise NARF, NARFH does not satisfy Part Dependency because all parameters are considered for each 3D location, and object part segmentation masks cannot be generated using Eq. 18 [S1]. However, NARFH has computational cost almost constant to the number of object parts and naturally avoids the zero density problem [S1].

## Operating characteristics
*   **Time Complexity:** The computational cost of Part-Wise NARF (NARFP) is proportional to the number of object parts $P$ [S1]. Holistic NARF (NARFH) has a computational cost that is almost constant with respect to the number of object parts [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Explicitly estimated rigid transformations via forward kinematics avoid implicit transformation issues and part dependency [S1]. It allows for novel view synthesis, depth estimation, and body part segmentation by manipulating poses [S2, unofficial].
*   **Limitations:** NARFP suffers from inefficient training due to zero density samples and high computational cost for complex objects [S1]. NARFH loses part dependency, preventing explicit part segmentation masks [S1].
*   **When to use:** Use NARFP when explicit part segmentation and strict part dependency are required, and the number of parts is small. Use NARFH when computational efficiency is prioritized over explicit part segmentation capabilities [S1].

## Background
Neural Articulated Radiance Field (NARF) was introduced in 2021 by Nian et al. [S1]. It builds on the concept of Rigidly Transformed Neural Radiance Fields (RT-NeRF) to model articulated objects. The method was developed to address issues in previous approaches like P-NeRF, specifically the part dependency problem where the network learns erroneous dependencies between parts [S1]. NARF is trained via pose-annotated images, decomposing the articulated object into rigid parts with local coordinate systems and global shape variations [S2, unofficial].

## Sources
[S1] Neural Articulated Radiance Field, https://doi.org/10.1109/iccv48922.2021.00571, fetched 2026-10-26
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
