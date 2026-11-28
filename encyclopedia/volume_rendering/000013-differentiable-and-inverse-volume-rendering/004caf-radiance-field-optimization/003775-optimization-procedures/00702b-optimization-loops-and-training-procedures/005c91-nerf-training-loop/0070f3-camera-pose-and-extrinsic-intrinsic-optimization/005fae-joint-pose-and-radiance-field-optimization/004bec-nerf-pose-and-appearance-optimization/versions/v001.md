# NeRF Pose and Appearance Optimization

Also called: UP-NeRF, Unconstrained Pose-prior-Free Neural Radiance Fields

## Overview
This method jointly optimizes camera poses and neural scene representations for unconstrained image collections containing varying illumination and transient occluders, without requiring camera pose priors [S1, unofficial]. It distinguishes itself from standard NeRF optimization by introducing a candidate head for robust pose estimation during early training and a loss scheduling mechanism that transitions from image-dependent to shared scene representations [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** An unposed collection of $N$ images $\{I_i\}$, their ground-truth colors $\{C(\mathbf{r}_i)\}$, and known camera intrinsics.
*   **Outputs:** Optimized camera poses $\{\hat{p}_i\}$, shared NeRF parameters $\theta$, and transient-aware scene representation.

**Data Structures**
1.  **Shared NeRF MLPs:**
    *   $\Pi_{\theta_1}$: Maps positional encoded 3D point $\gamma_x(\mathbf{r}(t))$ to intermediate feature $\mathbf{z}(t)$ and density $\sigma(t)$.
    *   $\Pi_{\theta_2}$: Maps $\mathbf{z}(t)$ and view direction encoding $\gamma_d(\mathbf{d})$ to color $\mathbf{c}(t)$.
2.  **Candidate Head MLP:**
    *   $\Pi_{\theta_3}$: Maps $\mathbf{z}(t)$ and candidate embedding $\boldsymbol{\ell}_i$ to candidate density $\sigma_i^{(c)}(t)$ and candidate color $\mathbf{c}_i^{(c)}(t)$.
3.  **Candidate Embeddings:**
    *   $\{\boldsymbol{\ell}_i\}_{i=1}^N$: Learnable vectors, one per image. Size is adjustable based on pose difficulty [S1, unofficial].
4.  **Camera Poses:**
    *   $\{\hat{p}_i\}$: Parameterized as 6 degrees of freedom using the $se(3)$ Lie algebra [S1, unofficial].

**Formulas**
*   **Positional Encoding:** $\gamma_x(\cdot)$ for 3D coordinates, $\gamma_d(\cdot)$ for view directions [S1, unofficial].
*   **Opacity:** $\alpha(t_k) = 1 - \exp(-\sigma(t_k)\delta_k)$ [S1, unofficial].
*   **Transmittance:** $T(t_k) = \exp\left(-\sum_{k'=1}^{k-1} \sigma(t_{k'})\delta_{k'}\right)$ [S1, unofficial].
*   **Shared Rendering:**
    $$ \hat{C}(\mathbf{r}) = \sum_{k=1}^K T(t_k) \alpha(\sigma(t_k)\delta_k) \mathbf{c}(t_k) $$
*   **Candidate Rendering:**
    $$ \hat{C}_i^{(c)}(\mathbf{r}) = \sum_{k=1}^K T_i^{(c)}(t_k) \left[ \alpha(\sigma(t_k)\delta_k)\mathbf{c}(t_k) + \alpha(\sigma_i^{(c)}(t_k)\delta_k)\mathbf{c}_i^{(c)}(t_k) \right] $$
    Where $T_i^{(c)}(t_k) = \exp\left(-\sum_{k'=1}^{k-1} (\sigma(t_{k'}) + \sigma_i^{(c)}(t_k'))\delta_{k'}\right)$ [S1, unofficial].
*   **Loss Scheduling Weight $w_{u,v}(l)$:**
    $$ w_{u,v}(l) = \begin{cases} 0 & \text{if } l < u \\ \frac{1 - \cos(\pi(l - u)/(v - u))}{2} & \text{if } u \le l < v \\ 1 & \text{if } l \ge v \end{cases} $$
    Where $l \in [0, 1]$ is training progress, and $u, v$ are hyperparameters [S1, unofficial].
*   **Total Loss:**
    $$ \mathcal{L} = w_{u,v} \mathcal{L}_{rgb}(\hat{C} | C) + (1 - w_{u,v}) \mathcal{L}_{rgb}(\hat{C}_i^{(c)} | C) $$
    Where $\mathcal{L}_{rgb}$ is the mean squared error between ground-truth and synthesized color [S1, unofficial].

**Parameters**
*   $u, v$: Hyperparameters defining the transition window for loss scheduling. Typical values not given in sources.
*   $\delta_k$: Distance between adjacent sample points $t_{k+1} - t_k$ [S1, unofficial].

**Steps**
1.  Initialize shared NeRF parameters $\theta$, candidate head parameters $\theta_3$, candidate embeddings $\{\boldsymbol{\ell}_i\}$, and camera poses $\{\hat{p}_i\}$.
2.  For each training step with progress $l$:
    a. Calculate weight $w_{u,v}(l)$ based on current progress $l$ and hyperparameters $u, v$.
    b. For each image $i$:
        i. Construct ray $\mathbf{r}$ using current pose estimate $\hat{p}_i$ and camera intrinsics.
        ii. Sample points $t_k$ along $\mathbf{r}$ using hierarchical sampling (coarse then fine) [S1, unofficial].
        iii. Compute shared features $\mathbf{z}(t_k)$, density $\sigma(t_k)$, and color $\mathbf{c}(t_k)$ using $\Pi_{\theta_1}$ and $\Pi_{\theta_2}$.
        iv. Compute candidate density $\sigma_i^{(c)}(t_k)$ and color $\mathbf{c}_i^{(c)}(t_k)$ using $\Pi_{\theta_3}$ with embedding $\boldsymbol{\ell}_i$.
        v. Render shared color $\hat{C}(\mathbf{r})$ and candidate color $\hat{C}_i^{(c)}(\mathbf{r})$ using the respective volume rendering equations.
        vi. Compute photometric loss $\mathcal{L}_{rgb}$ for both renders against ground truth $C(\mathbf{r})$.
    c. Compute total loss $\mathcal{L}$ using the weighted sum.
    d. Backpropagate and update $\theta, \theta_3, \{\boldsymbol{\ell}_i\}, \{\hat{p}_i\}$.
3.  Repeat until convergence.

**Edge Cases and Pitfalls**
*   **Hard-Pose Images:** Images with little overlap or unique details may cause incorrect pose optimization if only the shared representation is used early on. The candidate head mitigates this [S1, unofficial].
*   **Transient Occluders:** The candidate head helps block the influence of transient occluders on pose estimation [S1, unofficial].
*   **Regularization:** Unlike NeRF-W, this method does not use regularization terms or uncertainty fields for the candidate head, allowing it to handle hard-pose images more effectively [S1, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Superior performance compared to baselines including BARF and its variants on challenging internet photo collections like the Phototourism dataset [S1, unofficial].
*   **Strengths:** Handles unconstrained images with varying illumination and transient occluders without pose priors.
*   **Limitations:** Requires careful tuning of $u$ and $v$ for loss scheduling. Not given in the sources.

## Background
*   **Origin:** Proposed in UP-NeRF (Unconstrained Pose-prior-Free Neural Radiance Fields) [S1, unofficial].
*   **History:** Addresses limitations of NeRF in handling unconstrained image collections, which often suffer from varying illumination and transient occluders [S1, unofficial]. Builds on prior works like BARF for joint pose optimization [S1, unofficial].
*   **Usage:** Novel view synthesis for unconstrained image collections, virtual/augmented reality, autonomous systems, and robotics [S1, unofficial].

## Sources
[S1] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
