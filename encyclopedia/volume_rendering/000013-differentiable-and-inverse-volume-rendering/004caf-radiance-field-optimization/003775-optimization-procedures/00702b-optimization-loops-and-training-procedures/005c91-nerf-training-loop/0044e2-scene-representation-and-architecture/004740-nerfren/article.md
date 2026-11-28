# NeRFReN

Also called: Reflective NeRF

## Overview
NeRFReN addresses the failure of standard Neural Radiance Fields (NeRF) to render reflective surfaces, such as glass and mirrors, by separating the radiance field into transmitted and reflected components [S1]. It distinguishes itself from sibling methods by explicitly modeling the reflection fraction $\beta$ derived from the transmitted geometry, allowing for the separation of reflection artifacts from physical entities [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, and ray parameters [S1].
*   **Outputs:** Final pixel intensity $I$ [S1].

**Data Structures:**
The method separates the radiance field into two coupled components: a transmitted field with parameters $(\sigma_t, \mathbf{c}_t)$ representing physical geometry and direct lighting, and a reflected field with parameters $(\sigma_r, \mathbf{c}_r)$ representing virtual geometry and reflected lighting [S1].

**Algorithm Steps:**
1.  **Sample Transmitted Field:** For a given ray, sample points $\mathbf{x}_i$ and compute transmitted density $\sigma_{t,i}$ and color $\mathbf{c}_{t,i}$ [S1].
2.  **Compute Transmittance:** Calculate the transmittance $T_{\sigma_{t,i}}$ along the ray up to sample $i$ using the transmitted density [S1]:
    $$ T_{\sigma_{t,i}} = \prod_{j=1}^{i-1} (1 - \alpha_{t,j}) $$
3.  **Compute Reflection Fraction:** Calculate the scalar reflection fraction $\beta$ based on the geometry of the transmitted radiance field [S1]:
    $$ \beta = \sum_{i} T_{\sigma_{t,i}} (1 - \exp(-\sigma_{t,i} \delta_i)) \alpha_i $$
    where $\alpha_i$ is a coefficient defined in the source context [S1].
4.  **Sample Reflected Field:** Sample the reflected field $(\sigma_r, \mathbf{c}_r)$ along the ray to compute reflected intensity $I_r$ [S1].
5.  **Composite Pixel:** Compute the final pixel intensity $I$ as a weighted sum of the transmitted intensity $I_t$ and reflected intensity $I_r$ [S1]:
    $$ I = I_t + \beta I_r $$

**Loss Functions:**
Training employs three loss components [S1, S2]:
1.  **Photometric Loss:** Standard reconstruction loss between rendered and ground-truth images [S1].
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
