# NeuDA

Also called: NeuDA (Neural Deformable Anchor), Neural Deformable Anchor

## Overview
NeuDA is a neural implicit surface reconstruction method that utilizes hierarchical deformable anchors to encode local geometry, building on the NeuS [6060-neus.md] framework. It distinguishes itself by replacing standard positional encoding with Hierarchical Positional Encoding (HPE) and employing a shallower SDF network (4 layers vs. 8 for NeuS) [S1]. This design achieves high-fidelity surface reconstruction with lower computational cost and sharper surfaces compared to siblings like [HAME-NeRF](9325-hame-nerf.md), [D-NeuS](10217-d-neus.md), [NeuS2](10220-neus2.md), and [HF-NeuS](12039-hf-neus.md) [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses, optional foreground masks, and a 3D bounding box defining the scene volume.
*   **Outputs:** A signed distance function (SDF) $f(x)$, a predicted normal vector $\hat{n}(x)$, a geometric feature $z(x)$, and a color $C(r)$ for any ray $r$.

**Data Structures:**
1.  **Hierarchical Deformable Anchors $\psi(G)$:**
    *   Arranged into $L$ levels (default $L=8$) [S1].
    *   Each level $l$ contains $T_l$ coordinate vectors (anchors) $\mathbf{p}_n \in \mathbb{R}^3$.
    *   At the coarsest level ($l=0$), $T_0 = 16$.
    *   For finer levels, $T_{l} = T_{l-1} \times 1.38$ [S1].
    *   Initialized uniformly at the vertices of a voxel grid $G$ covering the scene [S1].
    *   These anchors are learnable parameters optimized during training.
2.  **SDF Network $\mathcal{F}$:**
    *   A 4-layer Multi-Layer Perceptron (MLP) [S1].
    *   Hidden layer size: 256 [S1].
    *   Input: Hierarchical Positional Encoding $\mathcal{H}(p)$.
    *   Output: Tuple $(f(x;\theta), \hat{n}(x;\theta), z(x;\theta))$ [S1].
3.  **Color Network:**
    *   Follows the NeuS architecture [S1].
    *   4-layer MLP with hidden layer size 256 [S1].
    *   Inputs: Geometric feature $z(x;\theta)$, view direction, and potentially other features.

**Algorithm Steps:**

1.  **Anchor Initialization:**
    Initialize the set of deformable anchors $\psi(G)$ uniformly distributed at the vertices of a voxel grid $G$ [S1].

2.  **Hierarchical Positional Encoding (HPE):**
    For a target point $p$, retrieve the $L$ levels of anchors. For each level $l \in \{0, \dots, L-1\}$:
    *   Let $\mathbf{p}_l$ be the set of anchor points at level $l$.
    *   Apply frequency encoding to each coordinate of each anchor point $\mathbf{p}_n$ individually. The frequency encoding function for level $l$ is:
        $$ \gamma(\mathbf{p}_{n,l}) = (\sin(2^l \pi \mathbf{p}_{n,l}), \cos(2^l \pi \mathbf{p}_{n,l})) $$
        This returns a 6-dimensional embedding for each anchor [S1].
    *   Interpolate these embeddings based on the spatial proximity of the target point $p$ to the anchors $\mathbf{p}_{n,l}$ to obtain a local embedding vector $\phi(\hat{\mathbf{p}}_l)$ for level $l$ [S1].
    *   Concatenate the multi-level embedding vectors to form the hierarchical positional encoding:
        $$ \mathcal{H}(p) = (\phi(\hat{\mathbf{p}}_0), \phi(\hat{\mathbf{p}}_1), \dots, \phi(\hat{\mathbf{p}}_{L-1})) $$
        [S1].

3.  **SDF and Feature Prediction:**
    Feed the encoded hierarchical feature $\mathcal{H}(p)$ into the SDF network $\mathcal{F}$:
    $$ \mathcal{F}(x; \theta) = \mathcal{F}(\mathcal{H}(p); \theta) = (f(x;\theta), \hat{n}(x;\theta), z(x;\theta)) $$
    where $f$ is the signed distance, $\hat{n}$ is the predicted normal, and $z$ is the geometric feature [S1].

4.  **Volume Rendering:**
    Sample points along rays. Compute transmittance $T_{r,i}$ and alpha $\alpha_{r,i}$ using the SDF values $f(x_{r,i})$ and a density function (typically derived from $f$ via a sigmoid or similar function as in NeuS, though the specific density function formula is not explicitly detailed in the provided notes, it is implied by the rendering pipeline [S1]). Render the color $C(r)$ using the color network [S1].

5.  **Loss Computation:**
    Compute the total loss $\mathcal{L}$ as a weighted sum of four terms [S1]:
    *   **Color Loss ($\mathcal{L}_c$):** Mean absolute error between rendered color $C(r)$ and ground truth $\hat{C}(r)$:
        $$ \mathcal{L}_c = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \| C(r) - \hat{C}(r) \| $$
        [S1].
    *   **Eikonal Regularization ($\mathcal{L}_{reg}$):** Ensures the gradient of the SDF has unit magnitude:
        $$ \mathcal{L}_{reg} = \frac{1}{|\mathcal{R}|N} \sum_{r,i} (\|\nabla f(\mathcal{H}(p_{r,i}))\|_2 - 1)^2 $$
        where $N$ is the number of sample points per ray [S1].
    *   **Normal Regularization ($\mathcal{L}_{norm}$):** Ties the SDF gradient to the predicted normal $\hat{n}_{r,i}$:
        $$ \mathcal{L}_{norm} = \sum_{r,i} T_{r,i} \alpha_{r,i} \| \nabla f(\mathcal{H}(p_{r,i})) - \hat{n}_{r,i} \| $$
        [S1].
    *   **Mask Loss ($\mathcal{L}_{mask}$):** Binary Cross Entropy (BCE) between the mask label $m_r$ and accumulated alpha:
        $$ \mathcal{L}_{mask} = \text{BCE}(m_r, \sum_i^n T_{r,i} \alpha_{r,i}) $$
        [S1].

    Total Loss:
    $$ \mathcal{L} = \mathcal{L}_c + \lambda_{eik}\mathcal{L}_{reg} + \lambda_{norm}\mathcal{L}_{norm} + \lambda_{mask}\mathcal{L}_{mask} $$
    [S1].

6.  **Optimization:**
    *   Use the Adam optimizer [S1].
    *   Train for 300,000 iterations [S1].
    *   Decay learning rate from $5 \times 10^{-4}$ to $2.5 \times 10^{-5}$ using a cosine decay scheduler [S1].

**Parameters:**
*   $L$: Total number of anchor grid levels. Default: 8 [S1].
*   $\lambda_{eik}$: Eikonal loss weight. Value: 0.1 [S1].
*   $\lambda_{norm}$: Normal regularization loss weight. Value: $3 \times 10^{-5}$ [S1].
*   $\lambda_{mask}$: Mask loss weight. Value: 0.1 [S1].
*   Positional encoding frequencies: 8 frequencies for deformable anchors, 4 frequencies for view directions [S1].

**Pitfalls and Edge Cases:**
*   **Anchor Levels:** Using too many hierarchical levels (e.g., $L=10$) can introduce redundant information that harms SDF approximation, leading to slightly lower performance (higher Chamfer Distance) than $L=8$ [S1].
*   **Standard Deviation Interpretation:** Standard deviation curves of the SDF gradient may not directly reflect global reconstruction quality. For instance, Instant-NeuS yields a slightly better mean Chamfer Distance than NeuS despite having a higher standard deviation [S1].

## Operating characteristics
*   **Time and Memory Complexity:** NeuDA uses a shallower SDF network (4 layers) compared to NeuS (8 layers), reducing computational cost [S1]. The HPE produces a low-dimensional embedding compared to standard Multi-Level Positional Encoding (ML-PE), which reduces memory overhead for feature storage [S1].
*   **Measured Performance:** On the DTU dataset (w/o mask setting), NeuDA achieves a mean Chamfer Distance (CD) of 0.67, outperforming Instant-NeuS (0.80) and NeuS (0.76) [S1]. With mask, it achieves 0.65, outperforming Instant-NeuS (0.79) [S1].
*   **Surface Sharpness:** NeuDA achieves the lowest standard deviation value compared to NeuS and Instant-NeuS, indicating sharper surfaces with less time cost [S1].
*   **Strengths:** High-fidelity reconstruction, sharper surfaces, lower computational cost due to shallower network and efficient encoding [S1].
*   **Limitations:** The method relies on empirical validation without strict mathematical proof for the deformable anchor mechanism [S1].

## Background
NeuDA was introduced in the paper "NeuDA: Neural Deformable Anchor for High-Fidelity Implicit Surface Reconstruction" [S1]. It builds upon the NeuS [6060-neus.md] framework, which was designed to improve geometric reconstruction quality in volume rendering. NeuDA specifically addresses the limitations of standard positional encoding and deep networks by introducing deformable anchors and hierarchical encoding.

## Sources
[S1] NeuDA: Neural Deformable Anchor for High-Fidelity Implicit Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00819, fetched 2026-10-26
