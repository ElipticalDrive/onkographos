# GAINS Stage II: Material Properties

Also called: GAINS Material Optimization, Joint Material and Lighting Optimization

## Overview
This method jointly optimizes material properties and environment lighting to resolve the ill-posed nature of inverse rendering from sparse views, preventing overfitting and enabling robust relighting [S1]. It distinguishes itself from its parent [GSGEN Appearance Refinement](15753-gsgen-appearance-refinement.md) by introducing segmentation-based intra-class consistency, intrinsic image decomposition priors, and multi-illuminated score distillation to regularize material estimates [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Gaussian primitives $G$ (position, covariance, opacity, SH coefficients) from Stage I; material parameters $M = \{a_j, r_j, m_j\}$ (albedo, roughness, metallicity) for each Gaussian $j$; environment lighting $L$; input images $I_i$ with known intrinsics.
*   **Outputs:** Optimized material parameters $M$, environment lighting $L$, and updated Gaussian class membership vectors $E_j$.

**Data Structures:**
*   **Gaussian Class Membership:** Each Gaussian $j$ is extended with a one-hot vector $E_j \in \{0, 1\}^K$, where $K$ is the number of semantic classes. $E_j$ encodes the Gaussian's class membership for segmentation guidance [S1].
*   **Material Parameters:** Stored per Gaussian. $a_j$ (albedo), $r_j$ (roughness), $m_j$ (metallicity) [S1].

**Numbered Steps:**

1.  **Segmentation Mask Lifting:**
    *   Sample $V=100$ novel viewpoints orbiting the scenery [S1].
    *   Render SH-colored images $\{\hat{I}_v\}_{v=1}^V$ from these viewpoints [S1].
    *   Generate segmentation masks for each $\hat{I}_v$ using SAM [S1].
    *   Extract features $f_{v,s} = [g_{v,s}, h_{v,s}]$ for each mask region $s$ in view $v$, where $g_{v,s}$ are CLIP features and $h_{v,s}$ are DINOv2 features [S1].
    *   For each view, lift the mask and features to the Gaussian that contributes most to the pixel color (based on $\alpha$-blending weights) [S1].
    *   Merge collected Gaussian objects from previous lifting operations using geometric and feature similarity scores to assign class labels $E_j$ [S1].

2.  **Intra-Class Consistency (ICC) Loss Calculation:**
    *   For each captured viewpoint $i$, identify the set of masks $S_i$ rendered from that view [S1].
    *   Compute the ICC loss $\mathcal{L}_{ICC}$ using the formula below, where $s_i \in S_i$ is a mask region, $A_{s_i}$ is the rendered masked albedo, $R_{s_i}$ is the rendered masked roughness, $M_{s_i}$ is the rendered masked metallicity, and $E_{s_i}$ is the masked re-render loss [S1].
    *   Apply the scaling function $\gamma(|s_i|) = \exp(25 \cdot |I| / ||s_i||^2)$, where $|I|$ is the total image area and $||s_i||^2$ is the number of pixels in the mask [S1].

3.  **Intrinsic Image Decomposition (IID) Loss Calculation:**
    *   Obtain the IID diffuse albedo $\hat{A}_i$ for each captured image $I_i$ using Teamwork [24] (or RGB2X [43] for outdoor data) [S1].
    *   Compute the IID loss $\mathcal{L}_{IID} = \beta(\tau) \cdot ||A_i - \hat{A}_i||_2^2$, where $A_i$ is the rendered diffuse albedo [S1].
    *   Apply the linearly decreasing weight $\beta(\tau)$ based on the current iteration step $\tau$ to mitigate view-inconsistency in IID estimates [S1].
    *   Note: IID supervision is *not* applied to roughness and metallicity due to their lower robustness and higher view-inconsistency [S1].

4.  **Multi-Illuminated Score Distillation Sampling (MI-SDS) Loss Calculation:**
    *   Start diffusion guidance after 3000 optimization steps [S1].
    *   Render relit images $R_{mat}(C_j, G, M, E_l)$ from a novel view $C_j$ under randomly selected lighting $E_l$ from a predetermined set $\{E_l\}_{l=1}^{|E|}$ [S1].
    *   Compute the noisy latent $Z_{jl} = \alpha_t R_{mat}(C_j, G, M, E_l) + \sigma_t \epsilon$, where $\epsilon \sim \mathcal{N}(0,1)$ and $\alpha_t, \sigma_t$ are diffusion schedule parameters [S1].
    *   Calculate the MI-SDS loss $\mathcal{L}_{MI-SDS} = \mathbb{E}_{t,\epsilon} w(t) ||\epsilon_\phi(Z_{jl}; t) - \epsilon||_2^2$, where $\epsilon_\phi$ is the pretrained diffusion model [S1].
    *   Use a guidance scale of 100 [S1].

5.  **Final Loss Combination and Optimization:**
    *   Combine all losses: $\mathcal{L}_{mat} = \mathcal{L}_{color} + \lambda_{ICC} \mathcal{L}_{ICC} + \lambda_{IID} \mathcal{L}_{IID} + \lambda_{MI-SDS} \mathcal{L}_{MI-SDS} + \lambda_{TV} \mathcal{L}_{TV}$ [S1].
    *   $\mathcal{L}_{TV}$ is a total variation loss on the reconstructed lighting for smoothing [S1].
    *   Optimize $M$ and $L$ for 7000 additional iterations [S1].

**Formulas:**

The Intra-Class Consistency loss $\mathcal{L}_{ICC}$ is defined as [S1]:

$$
\mathcal{L}_{ICC} = \frac{1}{|S_i|} \sum_{s_i \in S_i} \gamma(|s_i|) \cdot \left( \mathcal{L}_{r,m} + \mathcal{L}_a + \mathcal{L}_e \right)
$$

Where the components are [S1]:

$$
\mathcal{L}_{r,m} = \text{Var}(R_{s_i}) + \text{Var}(M_{s_i})
$$

$$
\mathcal{L}_a = \left( (1-R_{s_i}) + M_{s_i} \right) \cdot \frac{1}{3} \sum_{c=1}^3 \text{Var}(A_{s_i}^{(c)})
$$

$$
\mathcal{L}_e = \left( (1-M_{s_i}) + R_{s_i} \right) \cdot E_{s_i}
$$

The scaling function is [S1]:

$$
\gamma(|s_i|) = \exp\left(25 \cdot \frac{|I|}{||s_i||^2}\right)
$$

The IID loss is [S1]:

$$
\mathcal{L}_{IID} = \beta(\tau) \cdot ||A_i - \hat{A}_i||_2^2
$$

The MI-SDS loss is [S1]:

$$
\mathcal{L}_{MI-SDS} = \mathbb{E}_{t,\epsilon} w(t) ||\epsilon_\phi(Z_{jl}; t) - \epsilon||_2^2
$$

**Parameters:**
*   $\lambda_{ICC} = 0.1$ [S1]
*   $\lambda_{IID} = 2$ [S1]
*   $\lambda_{MI-SDS} = 0.0001$ [S1]
*   $\lambda_{TV} = 0.1$ [S1]
*   $V = 100$ (number of sampled novel viewpoints) [S1]
*   MI-SDS guidance scale: 100 [S1]
*   MI-SDS start step: 3000 [S1]
*   Optimization iterations: 7000 [S1]

**Edge Cases and Pitfalls:**
*   **Ill-posedness:** Multiple combinations of material $M$ and lighting $L$ produce similar appearances, leading to ambiguous estimates, especially with sparse views [S1]. Mitigated by segmentation, IID, and diffusion priors [S1].
*   **IID View-Inconsistency:** IID diffuse albedo estimates may be view-inconsistent. Mitigated by weighting the loss with $\beta(\tau)$, which linearly decreases over iterations [S1].
*   **Specular/Metallicity IID:** IID supervision is avoided for specular roughness and metallicity as these parameters are less robust and exhibit more inconsistencies across views [S1].

## Operating characteristics
*   **Time Complexity:** The method adds significant overhead due to the lifting process (rendering 100 views, running SAM/CLIP/DINOv2) and the MI-SDS loss (diffusion model inference). The optimization runs for 7000 additional iterations [S1].
*   **Memory Complexity:** Requires storing the one-hot class vectors $E_j$ for all Gaussians and caching features/masks during the lifting phase [S1].
*   **Strengths:** Produces robust, relightable material properties even from sparse inputs by regularizing against overfitting [S1].
*   **Limitations:** Relies on external models (SAM, CLIP, DINOv2, Teamwork/RGB2X, Diffusion) which may introduce errors or inconsistencies [S1]. The lifting process assumes accurate segmentation and feature matching [S1].

## Background
GAINS (Gaussian-Based Inverse Rendering from Sparse Multi-view Captures) was introduced by P. Noras et al. in 2024 [S1]. Stage II specifically addresses the material estimation phase, building on the geometry reconstruction from Stage I [S1]. The method is designed for inverse rendering from sparse multi-view captures, aiming to produce intrinsic images (albedo, roughness, metallicity) and environment lighting suitable for relighting [S1].

## Sources
[S1] GAINS: Gaussian-Based Inverse Rendering from Sparse Multi-view Captures, https://doi.org/10.1007/978-3-032-37602-2_18, fetched 2026-10-29
