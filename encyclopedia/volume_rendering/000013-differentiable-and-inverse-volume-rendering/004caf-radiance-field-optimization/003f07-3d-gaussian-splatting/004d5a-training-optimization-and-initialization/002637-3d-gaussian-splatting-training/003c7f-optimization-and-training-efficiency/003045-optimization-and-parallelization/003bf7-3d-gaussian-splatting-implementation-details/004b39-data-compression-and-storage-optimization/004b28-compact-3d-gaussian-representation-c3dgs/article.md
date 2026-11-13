# Compact 3D Gaussian Representation (C3DGS)

Also called: Optimized Minimal Gaussian (OMG)

## Overview
Compact 3D Gaussian Representation (C3DGS), also known as Optimized Minimal Gaussian (OMG), is a method that reduces the storage size and rendering latency of 3D Gaussian Splatting by minimizing the number of Gaussian primitives and compressing their attributes [S1, S2]. It addresses the high memory overhead of standard 3DGS by replacing per-Gaussian storage of view-dependent color and geometry with a grid-based neural field and codebooks, respectively [S1, S2]. Unlike its siblings, which may rely on ASCII encoding or sensitivity-aware clustering, C3DGS explicitly exploits spatial redundancy through Sub-Vector Quantization (SVQ) and a local distinctiveness-based pruning strategy to achieve extreme compression ratios (up to 300×) while maintaining high-fidelity rendering [S1, S2].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ 3D Gaussians initialized via K-means clustering on scene points [S2].
- Training images and camera poses [S2].
- Hyperparameters: threshold $\tau$, scaling factor $\lambda$, number of nearest neighbors $K$, number of sub-vectors $M$, codebook size $B$, and feature dimension $D$ [S2].

**Outputs:**
- A compact scene representation consisting of:
  - Quantized geometric attributes (scale $\hat{s}_n$, rotation $\hat{r}_n$) stored as codebook indices [S2].
  - Quantized appearance features (static $\hat{T}_n$, view-dependent $\hat{V}_n$) stored as codebook indices [S2].
  - A hash-based grid neural field for spatial feature extraction [S1, S2].
  - MLPs for mapping features to SH coefficients [S2].
  - Post-processed binary file containing quantized positions (G-PCC compressed), Huffman-encoded indices, and LZMA-compressed metadata [S2].

### Data Structures
1.  **Gaussian Attributes:**
    -   Position $p \in \mathbb{R}^{N \times 3}$ [S2].
    -   Opacity $o \in [0, 1]^N$ [S2].
    -   Scale $s \in \mathbb{R}^{N \times 3}$ [S2].
    -   Rotation $r \in \mathbb{R}^{N \times 4}$ (quaternion) [S2].
    -   Static Feature $T \in \mathbb{R}^{N \times D}$ [S2].
    -   View-dependent Feature $V \in \mathbb{R}^{N \times D}$ [S2].
2.  **Codebooks:**
    -   Geometric Codebooks: $C_s^{(m)} \in \mathbb{R}^{B \times 3}$ and $C_r^{(m)} \in \mathbb{R}^{B \times 4}$ for $m \in \{1, \dots, M\}$ [S2].
    -   Appearance Codebooks: $C_T^{(m)} \in \mathbb{R}^{B \times (D/2)}$ and $C_V^{(m)} \in \mathbb{R}^{B \times (D/2)}$ (assuming even split) [S2].
3.  **Indices:**
    -   Geometric Indices: $i_{s,n}^{(m)}, i_{r,n}^{(m)} \in \{1, \dots, B\}$ [S2].
    -   Appearance Indices: $i_{T,n}^{(m)}, i_{V,n}^{(m)} \in \{1, \dots, B\}$ [S2].
4.  **Neural Field:**
    -   Hash-based grid (Instant NGP style) mapping 3D positions to space features $F_n$ [S1, S2].
5.  **MLPs:**
    -   $MLP_s$: Maps positional encoding to space feature $F_n$ [S2].
    -   $MLP_t$: Maps $[T_n, F_n]$ to 0-degree SH coefficients $h_n^{(0)}$ [S2].
    -   $MLP_o$: Maps $[T_n, F_n]$ to opacity $o_n$ [S2].
    -   $MLP_v$: Maps $[V_n, F_n]$ to 1-3 degree SH coefficients $h_n^{(1,2,3)}$ [S2].

### Numbered Steps

1.  **Initialization:** Initialize $N$ Gaussians using K-means clustering on the input point cloud [S2].
2.  **Feature Learning:** Learn per-Gaussian static features $T \in \mathbb{R}^{N \times D}$ and view-dependent features $V \in \mathbb{R}^{N \times D}$ [S2].
3.  **Space Feature Extraction:** For each Gaussian $n$, compute the space feature $F_n$ using positional encoding $\gamma(p_n)$ and an MLP:
    $$ F_n = MLP_s(\gamma(p_n)) $$
    [S2]
4.  **Attribute Reconstruction:**
    -   Compute static color and opacity:
        $$ h_n^{(0)} = MLP_t(\text{cat}(T_n, F_n)) $$
        $$ o_n = MLP_o(\text{cat}(T_n, F_n)) $$
    -   Compute view-dependent color:
        $$ h_n^{(1,2,3)} = MLP_v(\text{cat}(V_n, F_n)) $$
    [S2]
5.  **Sub-Vector Quantization (SVQ) for Appearance:**
    -   Concatenate $T_n$ and $V_n$ into a single vector $z_n \in \mathbb{R}^{2D}$.
    -   Partition $z_n$ into $M$ sub-vectors $z_n^{(m)}$ of length $L$.
    -   For each sub-vector $m$, find the nearest codeword index $i_m$:
        $$ i_m = \arg \min_j \| z_n^{(m)} - C^{(m)}[j] \|_2^2 $$
    -   Reconstruct the quantized vector $\hat{z}_n$ by concatenating the selected codewords:
        $$ \hat{z}_n = \text{cat}(C^{(1)}[i_1], \dots, C^{(M)}[i_M]) $$
    -   Split $\hat{z}_n$ back into $\hat{T}_n$ and $\hat{V}_n$ [S2].
6.  **SVQ for Geometry:** Apply SVQ to scale $s_n$ and rotation $r_n$ to obtain quantized vectors $\hat{s}_n$ and $\hat{r}_n$ and their corresponding indices [S2].
7.  **Importance Scoring:**
    -   Compute the base importance score $\bar{I}_i$ for Gaussian $i$:
        $$ \bar{I}_i = \begin{cases} \sum_{\rho=1}^{N_R} w_{i,\rho} & \text{if } \exists \rho \in \{1, \dots, N_R\} \text{ s.t. } w_{i,\rho} = \max_j w_{j,\rho} \\ 0 & \text{otherwise} \end{cases} $$
        where $w_{i,\rho}$ is the blending weight of Gaussian $i$ for ray $\rho$, and $N_R$ is the total number of training rays [S2].
    -   Compute the final importance score $I_i$ incorporating local distinctiveness:
        $$ I_i = \bar{I}_i \left( \frac{1}{D} \sum_{j \in N_i^K} \| T_i - T_j \|_1 \right)^\lambda $$
        where $N_i^K$ is the set of $K$-nearest neighbors of Gaussian $i$ (approximated via Morton order sorting) and $\lambda$ is a scaling factor [S2].
8.  **Pruning:** Remove Gaussians with low importance scores using CDF-based thresholding with threshold $\tau$ [S2].
9.  **Training Strategy:**
    -   Train the network parameters (MLPs, Grid, Features) and Codebooks using rendering loss.
    -   In the final 1,000 iterations, freeze the selected codebook indices and fine-tune only the codebook entries using the rendering loss [S2].
10. **Post-processing:**
    -   Apply 16-bit quantization to positions $p$.
    -   Compress positions using G-PCC [S2].
    -   Huffman encode all SVQ indices [S2].
    -   Store all components in a single file with LZMA compression [S2].

### Rendering Equation
During rendering, the color $C(x)$ for a pixel $x$ is computed via alpha compositing of depth-sorted Gaussians $k=1 \dots N(x)$:
$$ C(x) = \sum_{k=1}^{N(x)} c_k \alpha_k(x) \prod_{j=1}^{k-1} (1 - \alpha_j(x)) $$
where $c_k$ is the color derived from the reconstructed SH coefficients, and $\alpha_k(x)$ is the 2D opacity:
$$ \alpha_n(x) = o_n \exp\left(-\frac{1}{2} (x - p'_n)^T \Sigma'_n (x - p'_n)\right) $$
where $p'_n$ and $\Sigma'_n$ are the projected center and covariance matrix of Gaussian $n$ [S2].

### Parameters
-   **Threshold $\tau$:** Controls the number of retained Gaussians. Values: XS=0.96, X=0.98, M=0.99, L=0.999, XL=0.9999 [S2].
-   **Scaling Factor $\lambda$:** Adjusts the influence of local distinctiveness [S2].
-   **Grid Type:** Hash-based grid representation (Instant NGP) [S1].

### Edge Cases and Pitfalls
-   **Neighbor Approximation:** The K-nearest neighbors are approximated by sorting Gaussians in Morton order and selecting adjacent indices. This may fail in highly non-uniform distributions, potentially affecting the accuracy of the local distinctiveness metric [S2].
-   **Index Freezing:** Freezing indices in the final training phase assumes convergence of the quantization assignment. If training is insufficient, indices may not be optimal, leading to reconstruction artifacts [S2].
-   **Precision:** 16-bit quantization of positions may introduce discretization errors, mitigated by G-PCC compression but potentially visible in high-frequency details [S2].

## Operating characteristics
-   **Storage Efficiency:** Achieves over 25× compression compared to standard 3DGS [S1]. On the Deep Blending dataset, it achieves over 28× enhancement in storage efficiency [S1]. In the bicycle scene, OMG-XS achieves over 300× compression relative to 3DGS [S2]. On Mip-NeRF 360, OMG-XS achieves nearly 50% reduction in storage compared to LocoGS-S [S2].
-   **Rendering Speed:** Nearly 40% increase in rendering speed on Deep Blending compared to 3DGS [S1]. General performance shows enhanced rendering speed across various datasets [S1].
-   **Quality:** Maintains comparable PSNR and SSIM to 3DGS and outperforms it in some metrics (e.g., Deep Blending PSNR) [S1, S2].
-   **Complexity:** The use of SVQ reduces computational overhead compared to full Vector Quantization (VQ) by using smaller codebooks [S2]. The hash-based grid provides fast processing speed [S1].
-   **Limitations:** The method relies on the assumption that neighboring Gaussians share similar attributes, which may not hold in highly textured or irregular scenes without sufficient redundancy [S2]. The approximation of neighbors via Morton order may impact pruning accuracy [S2].

## Background
-   **Origin:** Proposed in "Compact 3D Gaussian Representation for Radiance Field" (CVPR 2024) [S1] and "Optimized Minimal 3D Gaussian Splatting" (2024) [S2].
-   **Development:** Developed to address the large memory and storage requirements of 3DGS, which often exceed 1GB for real-world scenes [S1]. It builds on the observation that 3DGS densification produces redundant Gaussians and that spatial redundancy can be exploited for compression [S1, S2].
-   **Usage:** Used for efficient storage and real-time rendering of 3D scenes, particularly in applications where bandwidth or storage is constrained [S1, S2].

## Sources
[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S2] Optimized Minimal 3D Gaussian Splatting, https://doi.org/10.52202/085713-4534, fetched 2026-10-30
