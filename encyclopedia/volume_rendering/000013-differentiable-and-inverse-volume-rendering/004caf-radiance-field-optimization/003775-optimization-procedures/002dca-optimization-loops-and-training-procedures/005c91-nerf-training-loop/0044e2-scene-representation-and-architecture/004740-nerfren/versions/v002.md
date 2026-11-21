# NeRFReN

Also called: Reflective NeRF

## Overview
NeRFReN addresses the failure of standard Neural Radiance Fields (NeRF) to render reflective surfaces, such as glass and mirrors, by separating the radiance field into transmitted and reflected components [S1]. It distinguishes itself from sibling methods by explicitly modeling the reflection fraction $\beta$ derived from the transmitted geometry, allowing for the separation of reflection artifacts from physical entities [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, and ray parameters.
*   **Outputs:** Final pixel intensity $I$, composed of transmitted intensity $I_t$ and reflected intensity $I_r$.

**Data Structures:**
The method utilizes two coupled radiance fields, each represented by a neural network following the standard NeRF topology (density network $F_\sigma$ and color network $F_c$ with positional encoding and skip connections) [S1, S2]:
1.  **Transmitted Field:** Parameters $(\sigma_t, \mathbf{c}_t)$ representing the physical geometry and direct lighting.
2.  **Reflected Field:** Parameters $(\sigma_r, \mathbf{c}_r)$ representing the virtual geometry and reflected lighting.

**Algorithm Steps:**
1.  **Sample Transmitted Field:** For a given ray, sample points $\mathbf{x}_i$ and compute transmitted density $\sigma_{t,i}$ and color $\mathbf{c}_{t,i}$.
2.  **Compute Transmittance:** Calculate the transmittance $T_{\sigma_{t,i}}$ along the ray up to sample $i$ using the transmitted density:
    $$ T_{\sigma_{t,i}} = \prod_{j=1}^{i-1} (1 - \alpha_{t,j}) $$
    where $\alpha_{t,j} = 1 - \exp(-\sigma_{t,j} \delta_j)$ is the opacity at sample $j$ and $\delta_j$ is the step size [S1].
3.  **Compute Reflection Fraction:** Calculate the scalar reflection fraction $\beta$ based on the transmitted geometry. This value determines how much of the reflected field contributes to the final pixel [S1]:
    $$ \beta = \sum_{i} T_{\sigma_{t,i}} (1 - \exp(-\sigma_{t,i} \delta_i)) \alpha_i $$
    Here, $\alpha_i$ is a coefficient defined in the source context (Eq. 5 in [25]), typically related to the opacity or weighting of the sample [S1].
4.  **Sample Reflected Field:** Sample the reflected field $(\sigma_r, \mathbf{c}_r)$ along the ray (often using the same spatial samples or a mirrored ray direction depending on implementation specifics not fully detailed in the summary, but implied by the separation) to compute reflected intensity $I_r$ [S1].
5.  **Composite Pixel:** Compute the final pixel intensity $I$ as a weighted sum:
    $$ I = I_t + \beta I_r $$
    where $I_t$ is the standard volumetric rendering of the transmitted field [S1].

**Loss Functions:**
Training employs three loss components [S1, S2]:
1.  **Photometric Loss:** Standard reconstruction loss between rendered and ground-truth images.
2.  **Depth Smoothness Loss ($L_d$):** Applied to the transmitted field to encourage correct, smooth geometry [S1, S2].
3.  **Bidirectional Depth Consistency Loss ($L_{bdc}$):** Applied to the reflected field to enforce consistency between depth predictions from different viewpoints, helping to separate reflection artifacts from physical structures [S1, S2].

**Pitfalls:**
*   **Complex Interlacing:** The method cannot effectively distinguish between physical entities and reflection artifacts when they are complexly interlaced [S2].
*   **Transparency:** Performance degrades when occluding objects are light-loss transparent glass, as reflection artifacts may contaminate the training data [S2].

## Operating characteristics
*   **Performance:** The method outperforms baseline NeRF and NerfingMVS on the RFFR dataset for reflective surfaces [S1].
*   **Capabilities:** Supports scene editing via reflection removal and substitution [S1].
*   **Limitations:** Struggles with complex interlacing of entities and artifacts [S2].

## Background
NeRFReN was introduced in November 2021 [S1]. It was developed to address the common failure case of standard NeRF approaches when rendering reflective surfaces like glasses [S1]. The method was evaluated on the authors' RFFR dataset [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
