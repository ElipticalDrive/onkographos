# Semantic Radiance Field Rendering

Also called: Semantic NeRF Rendering

## Overview
Semantic Radiance Field Rendering generates point-wise semantic labels for simulated LiDAR points by volume-rendering semantic logits, using weak 2D image labels as supervision [S1]. It differs from standard NeRF rendering by outputting a semantic distribution (logits) rather than just RGB color, enabling the training of semantic fields through cross-entropy loss against ground-truth or weak labels [S1].

## How it works
**Inputs:**
*   A trained Semantic Radiance Field (SRF) model, parameterized by a neural network that maps 3D coordinates and viewing directions to density $\sigma$, color $c$, and semantic logits $s$ [S1, S2].
*   Ray origins $o$ and directions $d$ corresponding to simulated LiDAR beams or camera rays [S1].
*   Distances $\hat{D}$ from the sensor to objects (for LiDAR simulation) [S1].
*   Ground-truth or weak semantic labels $\hat{S}$ for the target rays [S1].

**Outputs:**
*   Rendered semantic logits $S(r)$ for each ray $r$ [S1].
*   Semantic label loss $L_l$ used for backpropagation [S1].

**Data Structures:**
*   **Sampled Points:** For each ray, $N$ points are sampled along the ray segment $[t_n, t_f]$. Each point $i$ has a 3D position $x_i$, a distance from the previous sample $\delta_i$, and accumulated transmittance $T_i$ [S1].
*   **Network Output:** At each sampled point $x_i$ and viewing direction $d$, the network outputs:
    *   Density $\sigma_i \in \mathbb{R}$
    *   Semantic logits $s_i \in \mathbb{R}^C$, where $C$ is the number of semantic classes [S1].

**Formulas:**
1.  **Transmittance and Weight:**
    The transmittance $T_i$ at sample $i$ is defined as:
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    where $\delta_i$ is the distance between sample $i$ and $i+1$ [S1].

2.  **Volume Rendering of Semantic Logits:**
    The rendered semantic logits $S(r)$ for a ray $r$ are computed as a weighted sum of the semantic logits at sampled points:
    $$ S(r) = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) s_i $$
    Here, $T_i$ is the accumulated transmittance, $\sigma_i$ is the density, $\delta_i$ is the segment length, and $s_i$ is the semantic logit vector at point $i$ [S1].

3.  **Semantic Label Loss:**
    The loss is the cross-entropy (CE) between the rendered logits and the ground-truth (or weak) labels $\hat{S}(r)$:
    $$ L_l = \text{CE}(\hat{S}(r), S(r)) $$
    where $\hat{S}(r)$ is the target semantic label for ray $r$ [S1].

4.  **Total Reconstruction Loss:**
    The total loss combines depth, RGB, and semantic components:
    $$ L_{\text{rec}} = L_{\text{depth}} + w_{\text{rgb}} L_{\text{rgb}} + \hat{w}_l \hat{L}_l + L_l $$
    where $w_{\text{rgb}}$ balances RGB geometry reconstruction, and $\hat{w}_l$ (or $w_l$) balances LiDAR rendering and semantic label rendering [S1].

**Steps:**
1.  **Ray Generation:** For simulated LiDAR, calculate the 3D point $p$ for each beam direction $d$ and distance $\hat{D}$ using $p = o + \hat{D}d$ [S1]. Define the ray $r_{\text{LiDAR}}$ originating from the LiDAR sensor [S1].
2.  **Sampling:** Sample $N$ points along the ray $r$ within the bounding volume of the scene [S1].
3.  **Network Query:** Query the SRF network at each sampled point $x_i$ with direction $d$ to obtain density $\sigma_i$ and semantic logits $s_i$ [S1].
4.  **Volume Rendering:** Compute the accumulated transmittance $T_i$ and the final semantic logits $S(r)$ using the weighted sum formula [S1].
5.  **Loss Computation:** Compute the cross-entropy loss $L_l$ between $S(r)$ and the label $\hat{S}(r)$ [S1].
6.  **Backpropagation:** Update the network parameters to minimize $L_{\text{rec}}$ [S1].

**Parameters:**
*   $N$: Number of samples per ray. Not given in the sources.
*   $w_{\text{rgb}}$: Weight for RGB reconstruction loss. Not given in the sources.
*   $w_l$ (or $\hat{w}_l$): Weight for semantic label loss. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Raydrop:** Real LiDAR sensors drop points based on reflectance and incidence angle. To align generated points with real LiDAR, a "raydrop" processing step is learned using ground-truth LiDAR frames as supervision [S1].
*   **Weak Labels:** If no annotated labels exist, weak labels are generated using a pre-trained SegFormer model on mixed datasets (Cityscapes, Mapillary, BDD, IDD) [S1].
*   **Point Cloud Correspondence:** Direct point-to-point correspondence between irregular 3D point clouds is difficult. Equirectangular image projection (panorama sparse depth) is used to leverage point-wise correspondence [S1].

## Subtopics

### SRN Point Cloud Rendering
A technique to produce point clouds of arbitrary resolution by querying the Semantic Radiance Network (SRN) at sampled points, obtaining both RGB and semantic labels, rather than forming an image per pose [S2]. It collects points given by the ray marcher across multiple poses instead of forming an image per each pose, then queries the SRN at each point in this set to obtain both RGB and semantic label point clouds [S2]. It is chosen when dense 3D semantic point clouds are required directly from the neural representation [S2]. In experiments, the method samples 65,165 points for an unseen chair in the test set [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $N$ per ray and the number of rays [S1].
*   **Memory Complexity:** Depends on the neural network size and the number of samples stored during forward/backward passes [S1].
*   **Performance:** The method enables full 3D reconstruction and dense semantic segmentation from posed RGB images or semantic masks [S2]. It requires only 30 human-annotated, posed semantic segmentation masks for training in semi-supervised settings [S2].
*   **Strengths:** Allows joint reconstruction of appearance, geometry, and semantic labels [S2]. Can simulate realistic LiDAR point clouds [S1].
*   **Limitations:** Requires weak label generation if no 3D semantic labels are available [S1]. Sensitive to LiDAR raydrop effects [S1].

## Background
Semantic Radiance Field Rendering originates from the integration of Neural Radiance Fields (NeRF) with semantic segmentation. It builds on [Semantic Loss Computation](15616-semantic-loss-computation.md) which computes the cross-entropy loss over coarse and fine semantic predictions [S1]. The concept of rendering semantic logits through volume rendering was proposed by Semantic NeRF (Zhi et al. 2021) [S1]. The specific application to LiDAR simulation and weak supervision was detailed in NeRF-LiDAR (Manivasagam et al. 2020; Fang et al. 2020) [S1]. The point cloud rendering variant was presented in Semantic Implicit Neural Scene Representations (SRN) [S2].

## Sources
[S1] NeRF-LiDAR: Generating Realistic LiDAR Point Clouds with Neural Radiance Fields, https://doi.org/10.1609/aaai.v38i7.28546, fetched 2026-10-29
[S2] Semantic Implicit Neural Scene Representations With Semi-Supervised Training, https://doi.org/10.1109/3dv50981.2020.00052, fetched 2026-10-26
