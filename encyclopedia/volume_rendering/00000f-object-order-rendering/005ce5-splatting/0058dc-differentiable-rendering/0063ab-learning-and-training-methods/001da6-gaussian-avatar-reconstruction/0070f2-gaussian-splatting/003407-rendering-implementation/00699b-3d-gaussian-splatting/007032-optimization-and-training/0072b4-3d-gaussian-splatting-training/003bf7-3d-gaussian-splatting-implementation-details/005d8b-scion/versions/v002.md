# SCION

Also called: Scene Composition with Instanced Neural Primitives

## Overview
SCION is a hierarchical compositional scene representation that replaces the millions of independent Gaussians used in standard 3D Gaussian Splatting (3DGS) with a compact vocabulary of reusable local primitives and lightweight world-space instances [S1, unofficial]. It solves the redundancy and lack of structural manipulation handles in standard 3DGS by discovering and encoding repetitive structures end-to-end from multi-view captures [S1, unofficial]. Unlike prior codebook methods that share parameters across distinct splats or require manual instance selection, SCION builds shared primitives and recovers structure automatically, enabling compact storage (e.g., 1.2 MB) and instance-level editing without retraining [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of posed RGB images $\{I_v\}$ with calibrated cameras $\{\pi_v\}$ and a sparse COLMAP point cloud [S1, unofficial].
*   **Output:** A scene representation consisting of a primitive vocabulary $\mathcal{C}$ and a set of instances $\Phi$, optimized to render novel views with high fidelity [S1, unofficial].

**Data Structures**
The representation consists of two main components:

1.  **Primitive Vocabulary $\mathcal{C} = \{c_k\}_{k=1}^K$:** A library of $K$ reusable primitives [S1, unofficial]. Each primitive $c_k$ is a bundle of $S_k$ local-frame Gaussians (splats) [S1, unofficial].
    *   The $j$-th splat in primitive $k$ has parameters $\theta_{kj}^L = (\mu_{kj}^L \in \mathbb{R}^3, q_{kj}^L \in S^3, s_{kj}^L \in \mathbb{R}^3, o_{kj}^L \in \mathbb{R}, c_{kj}^L \in \mathbb{R}^3)$ [S1, unofficial].
    *   $\mu^L$: Position, $q^L$: Unit quaternion rotation, $s^L$: Anisotropic log-scale, $o^L$: Opacity logit, $c^L$: RGB color (zeroth-order spherical harmonic) [S1, unofficial].
    *   Total parameters per primitive: $P_{splat} \times S_k$, where $P_{splat} = 14$ [S1, unofficial].

2.  **Instances $\Phi = \{\phi_i\}_{i=1}^N$:** $N$ lightweight world-space placements [S1, unofficial].
    *   The $i$-th instance has parameters $\phi_i^W = (\mu_i^W \in \mathbb{R}^3, q_i^W \in S^3, s_i^W \in \mathbb{R}, o_i^W \in \mathbb{R}, a_i \in \{1, \dots, K\})$ [S1, unofficial].
    *   $\mu^W$: World position, $q^W$: World rotation, $s^W$: Scalar log-scale, $o^W$: Opacity logit, $a_i$: Discrete index selecting the primitive $c_{a_i}$ [S1, unofficial].
    *   Total parameters per instance: $P_{instance} = 10$ (excluding the integer assignment) [S1, unofficial].

**Rendering Assembly**
Rendering materializes transient Gaussians by combining local splats with instance transforms [S1, unofficial]. For each instance $i$ and each local splat $j$ in its assigned primitive $k=a_i$, the world-frame Gaussian parameters $\tilde{\theta}_{ij}^W$ are computed as [S1, unofficial]:

$$
\tilde{\mu}_{ij}^W = \mu_i^W + q_i^W \text{rotate}(s_{ij}^L, \mu_{a_i j}^L)
$$
$$
\tilde{s}_{ij}^W = s_{a_i j}^L + s_i^W \mathbf{1}
$$
$$
\tilde{q}_{ij}^W = q_i^W \otimes q_{a_i j}^L
$$
$$
\tilde{o}_{ij}^W = o_{a_i j}^L + o_i^W
$$
$$
\tilde{c}_{ij}^W = c_{a_i j}^L
$$

where $q(\cdot)$ denotes rotation of a vector by a quaternion, $\otimes$ is quaternion composition, and $\mathbf{1}$ is a vector of ones [S1, unofficial]. The total number of assembled Gaussians is $T = \sum_{i=1}^N S_{a_i} = \sum_{k=1}^K n_k S_k$, where $n_k$ is the number of instances assigned to primitive $k$ [S1, unofficial].

**Optimization Procedure**
The representation is fitted via joint optimization over discrete (assignment) and continuous parameters [S1, unofficial]. The assignment $a_i$ is fixed after initialization using frozen DINOv3 features [S1, unofficial]. Optimization proceeds in two alternating phases per iteration. To avoid coupled gradients between primitive and instance parameters, each training iteration renders the same view twice [S1, unofficial]:

1.  **Splat Pass:** Render view with detached instance parameters: $\hat{I}_\theta = \mathcal{R}(\theta, \text{sg}(\phi))$ [S1, unofficial].
2.  **Instance Pass:** Render view with detached primitive parameters: $\hat{I}_\phi = \mathcal{R}(\text{sg}(\theta), \phi)$ [S1, unofficial].

Here $\mathcal{R}$ is the Gaussian rasterizer and $\text{sg}$ stops gradients [S1, unofficial].

**Loss Functions**
The total loss combines photometric, regularization, and adversarial terms [S1, unofficial]:

$$
\mathcal{L}_{total} = \mathcal{L}_{recon}^{(t)}(\hat{I}_\theta, I) + \mathcal{L}_{reg} + \mathcal{L}_{recon}^{(t)}(\hat{I}_\phi, I) + \mathcal{L}_{reg} + \mathbb{1}_{t \ge t_D} \lambda_{adv} \mathcal{L}_{adv}(\hat{I}_\theta, I)
$$

*   **Reconstruction Loss:**
    $$
    \mathcal{L}_{recon}^{(t)}(I, \hat{I}) = (1 - \lambda_{ssim}) \|I - \hat{I}\|_1 + \lambda_{ssim}(1 - \text{SSIM}(I, \hat{I})) + \mathbb{1}_{lpips}(t) \lambda_{lpips} \text{LPIPS}_{vgg}(I, \hat{I})
    $$
*   **Regularization Loss:**
    $$
    \mathcal{L}_{reg} = \mathcal{L}_{scale} + \mathcal{L}_{opac} + \lambda_{anch} \sum_i |s_i^W| + \lambda_{cap} \sum_i [\max(0, s_i^W - \tau_{cap})] + \lambda_c \sum_{k \in \mathcal{C}_{\ge 2}} \frac{1}{|A_k|} \sum_{j \in A_k} \|\mu_{kj}^L - \bar{\mu}_k\|_2^2
    $$
    *   $\mathcal{L}_{scale}$: Penalizes anisotropic local-frame splat scales.
    *   $\mathcal{L}_{opac}$: Caps large local opacity logits.
    *   $\lambda_{anch}$: Anchors instance log-scale near initialization.
    *   $\lambda_{cap}$: Caps positive instance log-scales to prevent runaway growth.
    *   $\lambda_c$: Keeps primitive active splats close to their local-frame mean $\bar{\mu}_k$.
    *   $\mathcal{C}_{\ge 2}$: Set of primitives with at least two active splats. $A_k$ is the active subset of primitive $k$.
*   **Adversarial Loss:** Added after warmup step $t_D$ to preserve per-instance sharpness against averaged reconstruction losses [S1, unofficial]. The discriminator compares randomly cropped real and rendered patches [S1, unofficial].

**Densification (Density Optimization)**
SCION uses two-level densification scoped to primitives and instances [S1, unofficial].

*   **Splat Density Optimization:**
    Uses the MH-3DGS birth rule scoped to one primitive [S1, unofficial]. A dead row in $c_k$ is filled with an opacity-weighted alive donor from $c_k$, and new children are written back into $c_k$ [S1, unofficial].
    1.  **Z-scoring:** Parent importance score is z-scored within each primitive to respond to local under-fitting [S1, unofficial].
    2.  **Density Normalization:** Acceptance is divided by same-primitive voxel density to prevent births piling up [S1, unofficial].
    3.  **Mass Correction:** When a row is copied into $\nu$ overlapping instances, opacity and scale are corrected to preserve alpha-compositing mass [S1, unofficial].

*   **Instance Density Optimization:**
    1.  **Pruning:** Remove instances with opacity below threshold or non-finite parameters [S1, unofficial].
    2.  **Ranking:** Rank remaining instances by accumulated gradient magnitude $\bar{h}_i$ [S1, unofficial]:
        $$
        \bar{h}_i = \frac{\sum_{v \in W} \sum_{j=1}^{S_{a_i}} \|\nabla_{\pi_v} (\tilde{\mu}_{ij}^W)\|_2^2}{\sum_{v \in W} S_{a_i}}
        $$
    3.  **Cloning (Small Instances):** If linear scale $e^{s_i^W}$ is below threshold:
        $$
        \mu_{i'}^W = \mu_i^W + \eta_{clone} e^{s_i^W} \xi, \quad \xi \sim \mathcal{N}(0, I)
        $$
        $$
        q_{i'}^W = q_i^W, \quad s_{i'}^W = s_i^W, \quad o_{i'}^W = o_i^W, \quad a_{i'} = a_i
        $$
    4.  **Subdivision (Large Instances):** Split into $M$ children. Children inherit rotation, assignment, and opacity.
        $$
        \mu_{i,m}^{W'} = \mu_i^W + \eta_{sub} e^{s_i^W} \epsilon_m, \quad \epsilon_m \sim \mathcal{N}(0, I)
        $$
        $$
        s_{i,m}^{W'} = s_i^W - \frac{1}{3} \log M
        $$
        $$
        o_{i,m}^{W'} = o_i^W
        $$
        *Pitfall:* Instance children occupy different world positions, so they do **not** use the alpha-compositing opacity correction applied to splat copying. Reducing opacity here would remove rendered mass [S1, unofficial].

**Parameters**
*   $P_{splat} = 14$: Parameters per local splat [S1, unofficial].
*   $P_{instance} = 10$: Parameters per instance [S1, unofficial].
*   $\lambda_{anch}, \lambda_{cap}, \lambda_c$: Regularization weights [S1, unofficial].
*   $t_D$: Warmup step for adversarial loss [S1, unofficial].
*   $M$: Number of children when subdividing a large instance [S1, unofficial].

## Operating characteristics
*   **Complexity:** Representation space complexity is $O(K \bar{S} + N)$, where $\bar{S}$ is the average number of local splats per primitive, $K$ is the number of primitives, and $N$ is the number of instances [S1, unofficial]. This is significantly smaller than the $O(\sum S_i)$ complexity of unshared 3DGS.
*   **Storage:** SCION maintains high quality even at 1.2 MB storage size [S1, unofficial].
*   **Compression Ratio:** Relative to an unshared representation with the same assembled Gaussians:
    $$
    CR = \frac{B_{ours}}{P_{splat} \sum_{k=1}^K n_k S_k}
    $$
    where $B_{ours} = P_{splat} \sum_{k=1}^K S_k + P_{instance} N$ [S1, unofficial].
*   **Strengths:** Compact representation, direct editing/animation handles, end-to-end structure discovery, rate-distortion favorable to existing compression methods [S1, unofficial].
*   **Limitations:** Trade-off of per-pixel fidelity for reusable structures; optimization is more complex due to two-level densification and adversarial loss [S1, unofficial].

## Background
SCION was introduced to address the redundancy in neural scene representations that model elements independently. Traditional methods treat each Gaussian as unique, storing millions of parameters. SCION leverages the fact that natural and built worlds are full of repetition (e.g., grass blades, bricks, leaves) [S1, unofficial]. It builds on 3D Gaussian Splatting [Kerbl et al., 2023] and addresses limitations of prior codebook-style methods like Splat and Replace, which require manual instance selection or start from already trained scenes [S1, unofficial]. SCION recovers structure end-to-end from multi-view captures, enabling instance-level scene editing and animation without retraining [S1, unofficial].

## Sources
[S1] SCION: Scene Composition with Instanced Neural Primitives, https://doi.org/10.48550/arxiv.2610.02322, fetched 2026-10-26 (unofficial)
