# HumanNeRF

Also called: HumanNeRF

## Overview
HumanNeRF is a neural representation for high-fidelity free-view synthesis of dynamic humans that generalizes to unseen subjects and camera settings using sparse RGB inputs, avoiding per-scene training through aggregated pixel-alignment features and a pose-embedded non-rigid deformation field [S1]. It differs from methods like [37] which generate blur texture results due to reliance on implicit texture representation, by combining image-based rendering with NeRF-based volume rendering into a novel neural blending scheme [S1].

## How it works
**Inputs and Outputs:**
The input consists of $K$ synchronized videos of a performer captured at different viewpoints (preferably 360°) with $T$ frames, denoted as $I = \{I_{k,t}\}$, aiming to synthesize free-viewpoint videos and generalize motion to an arbitrary person [S1]. The output is a rendered image from a novel viewpoint. The method consists of a generalizable neural radiance field, an optional fast per-scene fine-tuning scheme, and a novel neural appearance blending field [S1].

**Data Structures and Network Architectures:**
The method employs five main neural components. All linear layers use ReLU activation unless specified. Positional Encoding (PE) is applied to specific inputs as detailed below.

1.  **Feature Extractor Network ($U$):** A U-Net network extracting 2D image feature maps representing local image appearance from input image $I_k \in \mathbb{R}^{H \times W \times 4}$ (with mask as the last channel), producing a 2D feature map $f_k \in \mathbb{R}^{H \times W \times C}$ [S1].
    *   Layer type: CRB2D (ConvBnReLU2D) with kernel size $k=3$, stride $s=1$, dilation $d=1$ for all layers [S1].
    *   Architecture:
        *   Down0: 4/32
        *   Down1: 32/64
        *   Down2: 64/128
        *   Down3: 128/256
        *   Up1: 256/256
        *   Up2: 256/128
        *   Up3: 128/64
        *   Final Output: 64/32 [S1]

2.  **Feature Blending Network ($MLP_B$):** Estimates blending weights for aggregating features from multiple views [S1].
    *   Inputs: View direction $\vec{v}$, angle $\theta$, PE0, features $f$ [S1].
    *   Architecture:
        *   PE0: 6/54, 3/27 [S1]
        *   LR0: 54+27+32*6/256 [S1]
        *   LR1-4: 256/256 [S1]
        *   LR5: 256/128 [S1]
        *   LR6: 128/6 [S1]

3.  **Deformation Network ($MLP_d$):** Learns subtle displacements of the human body using SMPL skeleton information [S1].
    *   Inputs: $R_d$ (distances between sample point $p$ and 24 SMPL joints), PE0, $R_v$ (directions between $p$ and 24 SMPL joints), $F$ (blended feature) [S1].
    *   Architecture:
        *   PE0: 24/216 [S1]
        *   LR0: 216+72+32/256 [S1]
        *   LR1-4: 256/256 [S1]
        *   LR5: 128/3 [S1]

4.  **Generalizable Dynamic Neural Radiance Field ($\Phi$):** Regresses volume density and RGB radiance at location $p'$ in canonical space and its corresponding blending features $F$ [S1].
    *   Inputs: Position $x$, view direction $d$, PE0, LR2, PE0, LR2, PE1, $F$, LR7, LR8 [S1].
    *   Architecture:
        *   PE0: 3/63 [S1]
        *   LR0: 63/256 [S1]
        *   LR1-4: 256/256 [S1]
        *   LR5: 27+256/256 [S1]
        *   LR6: 256/256 [S1]
        *   Density $\sigma$: 256/1 [S1]
        *   PE1: 3/27 [S1]
        *   LR7: 256+27+32/256 [S1]
        *   LR8: 256/128 [S1]
        *   Color $c$: 128/3 [S1]

5.  **Appearance Blending Network ($MLP_A$):** Refines texture details by aggregating colors from neighboring views [S1].
    *   Inputs: $f_{r,0r}, f_{l,0l}$ (two adjacent image features and occlusion maps) [S1].
    *   Architecture:
        *   LR0: (32+3)*2/256 [S1]
        *   LR1-5: 256/256 [S1]
        *   LR6: 256/128 [S1]
        *   Blending weights $W$: 128/3 [S1]

**Numbered Steps:**

1.  **Feature Extraction:** For each input image $I^k$, extract the 2D feature map $f^k$ using the U-Net network $U$:
    $$ f^{k} = U(I^{k}) $$
    [S1]

2.  **Pixel Alignment and Aggregation:** For each spatial point $p \in \mathbb{R}^3$ fed to NeRF, project it into view $k$ at $q_k \in \mathbb{R}^2$ and fetch the corresponding feature vector $f_{q}^k$. The aggregated pixel alignment feature $F_p$ is a weighted summation of image features:
    $$ F_p = \sum_{k=1}^{K} w_q^k f_q^k $$
    where $w_q^k$ are blending weights estimated by $MLP_B$:
    $$ w_q^k = MLP_B(\vec{v}, \theta_q^k, f_q^k) $$
    Here, $\vec{v}$ is the viewing direction and $\theta_q^k$ is the angle of the viewing direction relative to the query ray of the source view [S1].

3.  **Deformation to Canonical Space:** The method leverages the parametric human body model SMPL for estimating a basis model. Use $R_d \in \mathbb{R}^{24}$ and $R_v \in \mathbb{R}^{72}$ to model the distances and directions between sample point $p$ and the 24 joints of the SMPL skeleton. Deform the point $p$ to canonical space $p'$ using the rigid deformation field $\mathcal{S}$ (based on SMPL motion $\mathcal{M}$ and skinning weights $w^s$) and the non-rigid displacement predicted by $MLP_d$:
    $$ p' = \mathcal{S}(p, \mathcal{M}, w^s) + MLP_d(R_d, R_v, F_p) $$
    [S1]

4.  **Radiance Field Evaluation:** Use the generalizable dynamic neural radiance field $\Phi$ to regress volume density $\sigma$ and RGB radiance $c$ at location $p'$ in canonical space and its corresponding blending features $F$, conditioned on the aggregated features $F_p$ and viewing direction [S1].

5.  **Appearance Blending (Optional/Refinement):** To address artifacts and lack of high-frequency details, refine texture details by aggregating colors from neighboring views using the neural appearance blending scheme. The network $MLP_A$ takes two adjacent image features and occlusion maps ($f_{r,0r}, f_{l,0l}$) as input to produce blending weights $W$ for color aggregation [S1].

**Parameters:**
*   SMPL skeleton joints: 24 joints used for computing distances ($R_d$) and directions ($R_v$) for the deformation network [S1].
*   Fine-tuning duration: 300 frames for a specific performer using only an hour to generate improved results [S1].

**Edge Cases and Pitfalls:**
*   **Multi-person or human-object interactions:** The method uses a regressed parametric human model, limiting it to single-person setups and failing on multi-person or human-object interactions [S1].
*   **Lighting variation artifacts:** No explicit lighting model; significant brightness or color changes between views cause severe artifacts. Switching nearest views during appearance blending causes jumping artifacts, especially with high brightness variance [S1].
*   **Generalization limits:** Capability is limited because human dataset distributions only cover a small portion of human dynamics and appearances [S1].

## Operating characteristics
*   **Time Complexity:** The method is significantly more efficient than per-scene methods. On the 'batman' video (300 frames, multi-view dataset), full training takes 1.2 hours, compared to Neural Body (6.7h), Neural Volumes (8.4h), and ST-NeRF (9.5h) [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On six input views, the method achieves PSNR = 32.59, SSIM = 0.9817, LPIPS = 0.0304 [S1].
*   **Scaling:** The average error increases rapidly as the camera number decreases (e.g., from six to two views) [S1].
*   **Strengths:** Requires ~1 hour fine-tuning for unseen identities, significantly less than per-scene methods. Without fine-tuning, provides comparable results to previous general/per-scene methods [S1].
*   **Limitations:** Limited to single-person setups; sensitive to lighting variations; generalization is constrained by the distribution of training human datasets [S1].
*   **When to use:** Used for dynamic human rendering from sparse multi-view inputs, specifically targeting unseen identities and poses with efficient generalization [S1].

## Background
HumanNeRF was proposed by [S1]. It addresses the limitations of existing neural human representations which require dense multi-view inputs and costly per-frame training, making them limited to static models. It builds on the idea of generalizable neural radiance fields, analogous to how IBRNet assists NeRF by avoiding per-scene training, by employing aggregated pixel-alignment features across multi-view inputs along with a pose-embedded non-rigid deformation field for tackling dynamic motions [S1].

## Sources
[S1] HumanNeRF: Efficiently Generated Human Radiance Field from Sparse Inputs, https://doi.org/10.1109/cvpr52688.2022.00759, fetched 2026-10-26
