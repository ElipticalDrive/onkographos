# CoNeRF

Also called: Controllable NeRF, Controllable Neural Radiance Field

## Overview
CoNeRF is a learning framework that extends Neural Radiance Fields to allow intuitive and interpretable user control over specific scene attributes (e.g., facial expressions) via few-shot supervision, treating attributes as latent variables regressed by the network [S1]. Unlike standard NeRFs that reconstruct a single static scene, or HyperNeRF which learns an arbitrary attribute space that cannot be supervised or related to specific local attributes, CoNeRF provides direct, interpretable control over localized appearance using sparse 2D mask annotations [S1]. It is domain-agnostic, avoiding the need for heavily engineered control models (like SMPL) or dense 3D supervision [S1].

## How it works
CoNeRF builds on the HyperNeRF architecture [S1], which uses a canonicalizer and a hyper map to lift latent codes into a hyperspace field. CoNeRF extends this by introducing regressors that map these latent codes to user-defined attributes and masks, enabling sparse supervision [S1].

**Inputs and Outputs**
*   **Inputs:** A collection of $C$ color images $\{C_c\} \in [0, 1]^{W \times H \times 3}$ [S1].
*   **Sparse Supervision:** For each image $c$ and attribute $a$, the user provides:
    *   Attribute value $\alpha_{c,a} \in [-1, 1]$ [S1].
    *   Mask annotation $M_{c,a}^{gt} \in [0, 1]^{W \times H}$ specifying the region controlled by attribute $a$ [S1].
*   **Outputs:** Rendered color $C(r)$ and density $\sigma$ for any ray $r$ and viewpoint, conditioned on user-specified attribute values [S1].

**Data Structures**
The model parameters $\theta$ include:
1.  **Canonicalizer $K$:** A neural network mapping query point $x$ to canonical space: $K(x) \equiv K(x | \beta, \theta)$ [S1].
2.  **Hyper Map $H$:** A neural network mapping the latent code $\beta$ to a hyperspace field: $\beta(x) \equiv H(x | \beta, \theta)$ [S1].
3.  **Radiance Field $R$:** A neural network taking canonical coordinates $K(x)$ and hyperspace features $\beta(x)$ to output density $\sigma(x)$ and color $c(x)$ [S1].
4.  **Attribute Regressor $A$:** Regresses the attribute value $\alpha$ from the latent code $\beta$ [S1].
5.  **Mask Regressor $M$:** Regresses the mask $m$ from the latent code $\beta$ [S1].

Each image $c$ has an associated learnable latent code $\beta_c$ [S1].

**Numbered Steps**
1.  **Initialization:** Initialize parameters $\theta$ and latent codes $\{\beta_c\}_{c=1}^C$ [S1].
2.  **Canonicalization:** For a 3D query point $x$ and latent code $\beta_c$, compute canonical coordinates:
    $$ K(x) = K(x | \beta_c, \theta) $$
    [S1]
3.  **Hyperspace Lifting:** Compute the hyperspace feature field for attribute $a$:
    $$ \beta(x) = H(x | \beta_c, \theta) $$
    [S1]
4.  **Attribute and Mask Regression:**
    *   Regress the attribute value $\alpha_c$ from $\beta_c$ using regressor $A$:
        $$ \alpha_c = A(\beta_c) $$
        [S1]
    *   Regress the volumetric mask field $m_a(x)$ for attribute $a$ from $\beta_c$ using regressor $M$:
        $$ m_a(x) = M(x | \beta_c, \theta) $$
        [S1]
5.  **Radiance Field Evaluation:** Compute density and color at $x$:
    $$ \sigma(x) = R(K(x), \beta(x)) $$
    $$ c(x) = R(K(x), \beta(x)) $$
    [S1]
6.  **Volume Rendering:** Render the color $C(r)$ for a ray $r$ passing through near plane $t_n$ and far plane $t_f$:
    $$ C(r) = \int_{t_n}^{t_f} T(t) \sigma(r(t)) c(r(t), v) dt $$
    where $v$ is the viewing angle, and transmittance $T(t)$ is:
    $$ T(t) = \exp \left( - \int_{t_n}^{t} \sigma(r(s)) ds \right) $$
    [S1]
    Numerical integration is used for rendering [S1].
7.  **Mask Rendering:** Project the 3D mask field $m_a(x)$ into image space using volume rendering to get $M(r | \beta_c, \theta)$ [S1].
8.  **Optimization:** Minimize the total loss (see Formulas) by updating $\theta$ and $\{\beta_c\}$ [S1].

**Formulas**
The total optimization objective is [S1]:
$$ \arg \min_{\theta=\{\theta, \beta_c\}} L_{rep}(\theta | \{C_c\}) + L_{ctrl}(\theta | \{M_{c,a}^{gt}\}, \{\alpha_{c,a}\}) $$
[S1]

**Reconstruction Loss $L_{rep}$**
Composed of photometric reconstruction loss $L_{recon}$ and latent code regularization $L_{enc}$ [S1]:
$$ L_{rep}(\cdot) = L_{recon}(\theta, \{\beta_c\} | \{C_c\}) + L_{enc}(\{\beta_c\}) $$
[S1]

1.  **Photometric Loss:** Minimizes L2 error between rendered color $C(r | \beta_c, \theta)$ and ground truth $C_{gt}(r)$ [S1]:
    $$ L_{recon}(\cdot) = \sum_{c} \mathbb{E}_{r \sim C_c} \| C(r | \beta_c, \theta) - C_{gt}(r) \|^2 $$
    [S1]

2.  **Regularization:** Imposes a zero-mean Gaussian prior on latent codes [S1]:
    $$ L_{enc}(\cdot) = \sum_{c} \| \beta_c \|^2 $$
    [S1]

**Control Loss $L_{ctrl}$**
Composed of mask loss $L_{mask}$ and attribute loss $L_{attr}$ [S1]:
$$ L_{ctrl}(\cdot) = L_{mask}(\theta, \{\beta_c\} | \{M_{c,a}^{gt}\}) + L_{attr}(\theta, \{\beta_c\} | \{\alpha_{c,a}\}) $$
[S1]

1.  **Attribute Loss:** Supervises the regressed attribute $\alpha_c$ against the user-specified $\alpha_{c,a}^{gt}$ only when an annotation is provided. Let $\delta_{c,a}$ be an indicator function where $\delta_{c,a}=1$ if annotation exists, else $0$ [S1]:
    $$ L_{attr}(\cdot) = \sum_{c} \sum_{a} \delta_{c,a} \| \alpha_{c,a} - \alpha_{c,a}^{gt} \|^2 $$
    [S1]

2.  **Mask Loss:** Supervises the rendered mask $M(r | \beta_c, \theta)$ against the ground truth mask $M_{c,a}^{gt}(r)$ using Cross-Entropy (CE) loss [S1]:
    $$ L_{mask}(\cdot) = \sum_{c} \sum_{a} \delta_{c,a} \mathbb{E}_{r} \left[ \text{CE} \left( M(r | \beta_c, \theta), M_{c,a}^{gt}(r) \right) \right] $$
    [S1]

**Parameters**
*   **Attribute Values $\alpha_{c,a}$:** User-specified, constrained to $[-1, 1]$ [S1].
*   **Mask Annotations $M_{c,a}^{gt}$:** User-specified, values in $[0, 1]^{W \times H}$ [S1].
*   **Number of Attributes $A$:** Discrete set defined by the user [S1].
*   **Number of Images $C$:** Collection of input images [S1].

**Edge Cases and Pitfalls**
*   **Quasi-Conditional Independence:** The method assumes spatial quasi-conditional independence of attributes [S1]. This allows the synthesis of attribute combinations never seen in training data (e.g., closed eyes with a smiling mouth) [S1].
*   **Sparse Supervision:** Attributes are supervised only when annotations are provided ($\delta_{c,a}=1$). For frames without annotations ($\delta_{c,a}=0$), attributes are discovered automatically by the framework during training [S1].
*   **Domain Agnosticism:** Unlike methods relying on SMPL (A-NeRF, NARF), CoNeRF does not require domain-specific control models, making it applicable to general scenes [S1].

## Subtopics

### [Controllable Radiance Fields (CoRF)](21624-controllable-radiance-fields-corf.md)
CoRF is a method for generalizable, 3D-aware, and motion-controllable synthesis of face dynamics, generating never-before-seen faces with controlled dynamic motions and viewpoints. It extends the NeRF paradigm by integrating a style-based generator that decouples identity and motion, allowing for intuitive control over facial expressions while maintaining 3D consistency. Unlike standard NeRFs that reconstruct a single static scene, CoRF learns a latent space for motion representations that can be interpolated or sampled to synthesize novel dynamic behaviors.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:**
    *   Provides direct, interpretable control over localized appearance via sparse mask annotations [S1].
    *   Requires only few-shot supervision (e.g., as little as two annotations per attribute) [S1].
    *   Domain-agnostic; does not require heavily engineered control models or dense 3D supervision [S1].
    *   Allows synthesis of novel attribute combinations due to quasi-conditional independence assumption [S1].
*   **Limitations:**
    *   Relies on user-provided sparse 2D mask annotations for control regions [S1].
    *   Assumes spatial quasi-conditional independence, which may not hold for all complex interactions [S1].
*   **When to use:** When interpretable, fine-grained control over specific scene attributes (like facial expressions) is required with minimal supervision [S1].
*   **When not to use:** When dense 3D supervision or domain-specific models (like SMPL) are available and preferred, or when attributes are strongly spatially dependent in ways that violate the quasi-conditional independence assumption [S1].

## Background
CoNeRF (Controllable Neural Radiance Fields) was introduced in the paper "CoNeRF: Controllable Neural Radiance Fields" [S1]. It extends the HyperNeRF framework [S1] by incorporating explicit control mechanisms via sparse 2D mask annotations and attribute values. Prior works like HyperNeRF learned arbitrary attribute spaces that could not be supervised or related to specific local attributes [S1]. Domain-specific methods like A-NeRF and NARF relied on SMPL models for human bodies [S1]. CoNeRF addresses these limitations by providing a domain-agnostic framework for controllable neural implicits [S1].

## Sources
[S1] CoNeRF: Controllable Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.01807, fetched 2026-10-26
