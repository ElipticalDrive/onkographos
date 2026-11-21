# CoNeRF

Also called: Controllable NeRF, Controllable Neural Radiance Field

## Overview
CoNeRF extends the HyperNeRF framework to enable intuitive, interpretable, and sparse supervision of localized scene attributes (e.g., facial expressions) by treating them as latent variables [S1]. Unlike HyperNeRF, which learns an arbitrary attribute space, or domain-specific methods like A-NeRF that rely on SMPL models, CoNeRF provides direct control over localized appearance using only sparse 2D mask annotations and attribute values, without requiring dense 3D supervision or heavy engineered control models [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A collection of $C$ color images $\{C_c\} \in [0, 1]^{W \times H \times 3}$ [S1].
*   **Sparse Supervision:** For a subset of images and attributes, the user provides:
    *   Attribute values $\alpha_{c,a} \in [-1, 1]$ for the $a$-th attribute in the $c$-th image [S1].
    *   Mask annotations $M_{c,a}^{gt} \in [0, 1]^{W \times H}$ specifying the image region controlled by the $a$-th attribute [S1].
*   **Output:** A radiance field model capable of rendering novel views and synthesizing attribute combinations not seen during training (e.g., closed eyes with a smiling mouth) [S1].

**Data Structures**
The model consists of three neural networks and a set of latent codes:
1.  **Canonicalizer $K$:** Maps query point $x$ to canonical space. Parameters $\theta_K$.
    *   $K(x) \equiv K(x | \beta, \theta)$ [S1].
2.  **Hyper Map $H$:** Maps the latent code $\beta$ to a hyperspace field $\beta(x)$. Parameters $\theta_H$.
    *   $\beta(x) \equiv H(x | \beta, \theta)$ [S1].
3.  **Radiance Field $R$:** Takes canonical coordinates $K(x)$ and hyperspace features $\beta(x)$ to output density $\sigma(x)$ and color $c(x)$. Parameters $\theta_R$.
    *   $c(x), \sigma(x) = R(K(x), \beta(x) | \theta)$ [S1].
4.  **Regressors $A$ and $M$:** Map the latent code $\beta$ to attribute values $\alpha$ and mask fields $m(x)$ respectively. Parameters $\theta_A, \theta_M$.
    *   $\alpha = A(\beta | \theta_A)$ [S1].
    *   $m(x) = M(x | \beta, \theta_M)$ [S1].
5.  **Latent Codes:** A set of latent codes $\{\beta_c\}_{c=1}^C$, one per input image, optimized jointly with network parameters [S1].

**Numbered Steps**
1.  **Initialization:** Initialize parameters $\theta = \{\theta_K, \theta_H, \theta_R, \theta_A, \theta_M\}$ and latent codes $\{\beta_c\}$.
2.  **Canonicalization:** For a query 3D point $x$ and image $c$, compute canonical coordinates:
    $$ K(x) = K(x | \beta_c, \theta) $$
    [S1]
3.  **Hyperspace Lifting:** Compute the hyperspace feature field for the attribute:
    $$ \beta(x) = H(x | \beta_c, \theta) $$
    [S1]
4.  **Attribute and Mask Regression:**
    *   Regress the attribute value $\alpha_c$ from the latent code:
        $$ \alpha_c = A(\beta_c | \theta_A) $$
        [S1]
    *   Regress the 3D volumetric mask field $m_a(x)$ for attribute $a$:
        $$ m_a(x) = M(x | \beta_c, \theta_M) $$
        [S1]
5.  **Radiance Field Evaluation:** Compute density and color:
    $$ \sigma(x), c(x) = R(K(x), \beta(x) | \theta) $$
    [S1]
6.  **Volume Rendering:** Render the color $C(r)$ and mask $M(r)$ for a ray $r$ parameterized by $t \in [t_n, t_f]$:
    $$ C(r) = \int_{t_n}^{t_f} T(t) \sigma(r(t)) c(r(t), v) dt $$
    $$ M(r) = \int_{t_n}^{t_f} T(t) \sigma(r(t)) m(r(t)) dt $$
    where transmittance $T(t) = \exp\left(-\int_{t_n}^t \sigma(r(s)) ds\right)$ [S1].
7.  **Loss Computation:** Calculate the total loss (see Formulas).
8.  **Optimization:** Update $\theta$ and $\{\beta_c\}$ to minimize the total loss.

**Formulas**
The total optimization objective is:
$$ \arg \min_{\theta=\{\theta, \beta_c\}} L_{rep}(\theta | \{C_c\}) + L_{ctrl}(\theta | \{M_{c,a}^{gt}\}, \{\alpha_{c,a}\}) $$
[S1]

**Reconstruction Loss ($L_{rep}$):**
$$ L_{rep}(\cdot) = L_{recon}(\theta, \{\beta_c\} | \{C_c\}) + L_{enc}(\{\beta_c\}) $$
[S1]

*   **Photometric Reconstruction Loss ($L_{recon}$):**
    $$ L_{recon}(\cdot) = \sum_{c} \mathbb{E}_{r \sim C_c} \| C(r | \beta_c, \theta) - C_{gt}(r) \|^2 $$
    [S1]

*   **Latent Code Regularization ($L_{enc}$):**
    $$ L_{enc}(\cdot) = \sum_{c} \| \beta_c \|^2 $$
    [S1]

**Control Loss ($L_{ctrl}$):**
$$ L_{ctrl}(\cdot) = L_{mask}(\theta, \{\beta_c\} | \{M_{c,a}^{gt}\}) + L_{attr}(\theta, \{\beta_c\} | \{\alpha_{c,a}\}) $$
[S1]

*   **Attribute Loss ($L_{attr}$):**
    $$ L_{attr}(\cdot) = \sum_{c} \sum_{a} \delta_{c,a} \| \alpha_{c,a} - \alpha_{c,a}^{gt} \|^2 $$
    where $\delta_{c,a} = 1$ if annotation is provided, else $0$. [S1]

*   **Mask Loss ($L_{mask}$):**
    $$ L_{mask}(\cdot) = \sum_{c} \sum_{a} \delta_{c,a} \mathbb{E}_{r} \text{CE}(M(r | \beta_c, \theta), M_{c,a}^{gt}(r)) $$
    where $\text{CE}$ is the cross-entropy loss between the rendered mask and the ground truth mask [S1].

**Parameters**
*   **Attribute Values $\alpha_{c,a}$:** Constrained to $[-1, 1]$ [S1].
*   **Mask Annotations $M_{c,a}^{gt}$:** Values in $[0, 1]^{W \times H}$ [S1].
*   **Frequency Bands:** Not given in the sources.
*   **MLP Layers:** Not given in the sources.

**Edge Cases and Pitfalls**
*   **Quasi-Conditional Independence:** The method assumes spatial quasi-conditional independence of attributes, allowing the synthesis of attribute combinations never seen in training data (e.g., closed eyes with a smiling mouth) [S1].
*   **Sparse Supervision:** Attributes are supervised only when annotations are provided ($\delta_{c,a}=1$). For frames without annotations ($\delta_{c,a}=0$), attributes are discovered automatically by the framework during training [S1].
*   **Domain Agnosticism:** Unlike methods relying on SMPL, CoNeRF does not require domain-specific control models, making it applicable to general scenes [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Allows fine-grained, direct, and interpretable control over attributes with as little as two annotations per attribute [S1]. Does not require dense 3D supervision or heavily engineered control models [S1].
*   **Limitations:** Relies on the assumption of spatial quasi-conditional independence of attributes [S1]. Requires sparse manual annotations for training [S1].

## Background
CoNeRF was introduced in the paper "CoNeRF: Controllable Neural Radiance Fields" [S1]. It builds upon the HyperNeRF framework [S1]. The method addresses the limitations of existing works that either enable only compositional control of object location or rely on domain-specific models like SMPL for human bodies [S1]. CoNeRF was designed to allow intuitive and interpretable user control over scene attributes via few-shot supervision [S1].

## Sources
[S1] CoNeRF: Controllable Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.01807, fetched 2026-10-26
