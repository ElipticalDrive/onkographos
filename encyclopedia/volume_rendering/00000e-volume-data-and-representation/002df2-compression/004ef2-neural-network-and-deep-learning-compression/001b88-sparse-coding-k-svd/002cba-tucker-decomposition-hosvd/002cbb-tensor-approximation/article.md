# Tensor Approximation

Also called: HOOI-based Compression, Tensor Decomposition (TD), Tucker Approximation, Tucker Decomposition, Tucker Decomposition for Data Compression, Tucker Tensor Approximation, Tucker formulation, Tucker model

## Overview
Tensor Approximation (TA) is a framework for data reduction and feature extraction that transforms multidimensional datasets into compact representations via tensor decomposition and reconstructs them via tensor reconstruction [S1, unofficial][S2, unofficial]. It solves the storage and rendering bottleneck of massive volume datasets by factorizing data into a core tensor and factor matrices, achieving high compression ratios while preserving specific structural features [S1, unofficial][S2, unofficial]. Unlike standard Level-of-Detail reduction that typically causes blurring, rank-reduced TA removes small-scale features to highlight specific structural features at different spatial scales [S3]. It differs from wavelet transforms by capturing non-axis aligned features more effectively with the same coefficient count, though wavelets may still outperform TA in reconstructing overall statistical distributions at coarse resolutions [S3, S4, S5].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3rd-order tensor $A \in \mathbb{R}^{I_1 \times I_2 \times I_3}$ representing a volume block or entire volume [S3][S7, unofficial].
*   **Outputs:** A core tensor $B \in \mathbb{R}^{R_1 \times R_2 \times R_3}$ and three factor matrices $U^{(1)} \in \mathbb{R}^{I_1 \times R_1}$, $U^{(2)} \in \mathbb{R}^{I_2 \times R_2}$, $U^{(3)} \in \mathbb{R}^{I_3 \times R_3}$, where $R_n < I_n$ [S3][S7, unofficial].

**Data Structures:**
*   **Tensor $A$:** A 3D array of voxel values with elements $a_{i_1 i_2 i_3}$ [S3][S7, unofficial].
*   **Core Tensor $B$:** A 3D array of interaction coefficients with elements $b_{r_1 r_2 r_3}$ [S3][S7, unofficial].
*   **Factor Matrices $U^{(n)}$:** 2D matrices where columns are learned basis vectors for mode $n$, with elements $u_{i_n r_n}^{(n)}$ [S3][S7, unofficial].
*   **Quantization:** Core tensor coefficients use 8-bit logarithmic encoding, and basis matrix coefficients use 16-bit linear encoding [S5, S6].

**Definitions:**
*   **Tensor Order:** The number of modes (dimensions) [S1, unofficial].
*   **Fiber:** A vector obtained by fixing all indices except one [S1, unofficial].
*   **Unfolding (Matricization):** Reorganizing a tensor into a matrix for computation, typically using backward cyclic unfolding [S1, unofficial].
*   **Multilinear Rank:** The tuple $(R_1, R_2, R_3)$ defining the core tensor dimensions [S3].

**Algorithm Steps:**
1.  **Tensor Decomposition:** Input volume tensor $A$. Compute the Tucker decomposition using Higher-Order SVD (HOSVD) or Higher-Order Orthogonal Iteration (HOOI) [S2, unofficial][S3]. HOSVD performs matrix SVD along every data mode after unfolding [S2, unofficial]. HOOI uses Alternating Least Squares (ALS) to directly obtain a reduced-rank decomposition, avoiding suboptimal truncation from HOSVD [S2, unofficial][S3]. Truncate to target multilinear ranks $(R_1, R_2, R_3)$ for data reduction [S3]. In multiresolution hierarchies, this runs locally per brick [S3, S5].
2.  **Quantization:** Apply tensor-specific quantization to the floating-point coefficients of $B$ and $U^{(n)}$ to reduce memory footprint [S5, S6]. Use 8-bit logarithmic encoding for $B$ and 16-bit linear encoding for $U^{(n)}$ [S5, S6].
3.  **Tensor Reconstruction:** Reconstruct the volume from $B$ and $U^{(n)}$ using one of three methods:
    *   **Method A: Progressive Reconstruction (Sum of Rank-One Tensors):** Iterate over all core indices $r_1, r_2, r_3$. Compute the weighted outer product of the corresponding column vectors $u_{r_1}^{(1)}, u_{r_2}^{(2)}, u_{r_3}^{(3)}$ scaled by $b_{r_1 r_2 r_3}$. Sum all weighted subtensors to form $\tilde{A}$ [S3]. Formula: $\tilde{A} = \sum_{r_1} \sum_{r_2} \sum_{r_3} b_{r_1 r_2 r_3} \cdot u_{r_1}^{(1)} \cdot u_{r_2}^{(2)} \cdot u_{r_3}^{(3)}$ [S3].
    *   **Method B: Voxel-wise Reconstruction:** For each voxel $(i_1, i_2, i_3)$, compute the sum over all core indices of the product of the core coefficient and the corresponding factor matrix coefficients [S3]. Formula: $a_{i_1 i_2 i_3} = \sum_{r_1} \sum_{r_2} \sum_{r_3} b_{r_1 r_2 r_3} \cdot u_{i_1 r_1}^{(1)} \cdot u_{i_2 r_2}^{(2)} \cdot u_{i_3 r_3}^{(3)}$ [S3].
    *   **Method C: Tensor Times Matrix (TTM) / n-mode Product:** Reconstruct by sequentially applying n-mode products along every mode [S3]. The n-mode product is defined as $Y = X \times_n U \iff Y^{(n)} = U X^{(n)}$, where $X^{(n)}$ is the mode-$n$ unfolding of $X$ [S3]. Implementation: (1) Unfold $B$ along mode 1 to $B^{(1)}$. (2) Compute $M_1 = U^{(1)} B^{(1)}$. (3) Unfold $M_1$ along mode 2 to $M_1^{(2)}$. (4) Compute $M_2 = U^{(2)} M_1^{(2)}$. (5) Unfold $M_2$ along mode 3 to $M_2^{(3)}$. (6) Compute $\tilde{A}^{(3)} = U^{(3)} M_2^{(3)}$. (7) Fold $\tilde{A}^{(3)}$ back into a 3rd-order tensor $\tilde{A}$ [S3]. In GPU implementations, each kernel thread computes one voxel using this linear complexity approach [S3, S5].

**Parameters:**
*   **Multilinear Ranks $(R_1, R_2, R_3)$:** User-defined parameters controlling compression ratio and feature retention [S3]. Typical values: $R_n = I_n / 2$ (e.g., rank 8 for $16^3$ blocks) [S3].
*   **Block Size:** TA is often applied to blocks (bricks) of size $I_1 \times I_2 \times I_3$ (e.g., $16^3$) within a multiresolution hierarchy [S3, S5].

**Edge Cases and Pitfalls:**
*   **Block Boundaries:** At low ranks, block boundaries become apparent as artifacts [S3]. This is inherent to brick-based lossy compression [S3].
    *   *Mitigation:* Interblock interpolation by sampling neighboring bricks or deferred filtering, at the cost of higher rendering time [S3].
*   **Feature Loss:** High rank reduction removes small-scale features. While this can highlight larger structures (e.g., growth prisms in dental enamel), it may obscure fine details [S3].
*   **Statistical Distribution:** TA may not reconstruct the overall statistical distribution of the dataset as well as wavelets at coarse resolutions [S5].

## Subtopics

### [Hierarchical Tensor-based Transform](5369-hierarchical-tensor-based-transform.md)
A compression scheme that recursively partitions the input domain into smaller blocks, defining a truncated tensor-product basis for each block. It represents the volume as a summation of incomplete tensor approximations across multiple resolution levels, using residual tensors passed from higher levels. Choose this when a multiscale hierarchical structure is required for the entire domain.

### [Tensor Times Matrix (TTM) Reconstruction](7404-tensor-times-matrix-ttm-reconstruction.md)
A GPU-accelerated method for reconstructing volume bricks from their Tucker decomposition by sequentially applying n-mode products. It reduces per-voxel complexity from cubic $O(R^3)$ to linear $O(R)$ with respect to rank $R$. Choose this for high-performance GPU rendering where memory pressure from storing the full core tensor in threads is a bottleneck.

### [Tensor-Train (TT) Decomposition](14059-tensor-train-tt-decomposition.md)
Factorizes a high-dimensional tensor into a sequence of two- and three-dimensional tensors (TT cores), reducing memory complexity from quartic to linear growth with respect to dimensionality. Unlike Tucker, TT cores maintain fixed rank dimensions. Choose this for very high-dimensional data (e.g., 4D volumes) where Tucker's core dimensionality would grow prohibitively.

## Operating characteristics
*   **Time Complexity:**
    *   Decomposition: Depends on the algorithm (HOSVD involves SVDs; HOOI is iterative ALS) [S2, unofficial].
    *   Reconstruction (Progressive): Cubic rank complexity $O(R_1 R_2 R_3)$ [S3].
    *   Reconstruction (TTM/Voxel-wise): Linear rank complexity $O(R)$ with respect to the rank $M$ (assuming symmetric ranks $R_1=R_2=R_3=R$) [S3, S5]. For $R=16$, this is a 256-fold improvement over progressive reconstruction [S3].
*   **Memory Complexity:**
    *   Storage: Proportional to $R_1 R_2 R_3 + I_1 R_1 + I_2 R_2 + I_3 R_3$. Significantly smaller than the original $I_1 I_2 I_3$ when $R_n \ll I_n$ [S3, S5].
*   **Measured Performance:**
    *   Applied to interactive large volume visualization (e.g., Chameleon dataset, $16^3$ blocks, rank 8) [S3].
    *   Allows real-time exploration of massive volumes (e.g., great ape molar, veiled chameleon) at high data reduction ratios [S3].
*   **Strengths:**
    *   Captures non-axis aligned features at different scales better than wavelets with the same number of coefficients [S3, S4].
    *   Rank reduction acts as a feature extractor, highlighting specific structural features (e.g., layered periodic arrangements) rather than just blurring [S3].
    *   Compact model suitable for GPU reconstruction [S3].
*   **Limitations:**
    *   Block boundary artifacts at low ranks [S3].
    *   May not preserve overall statistical distribution as well as wavelets at coarse resolutions [S5].
    *   Decomposition can be computationally expensive during preprocessing [S3].
*   **When to Use:** Interactive visualization of massive microstructural volume datasets where specific feature extraction and high compression are prioritized over exact statistical fidelity [S3, S5].
*   **When Not to Use:** When preserving the exact statistical distribution of the data at coarse resolutions is critical, or when block artifacts are unacceptable without expensive interpolation [S3, S5].

## Background
Tensor Approximation (TA) was introduced as a higher-order extension of Singular Value Decomposition (SVD) and Principal Component Analysis (PCA) for multidimensional data [S1, unofficial][S2, unofficial]. The Tucker model, a common approach for TA, approximates a tensor by a product of a core tensor and factor matrices [S3]. Algorithms like HOSVD (Higher-Order SVD) and HOOI (Higher-Order Orthogonal Iteration) are used to compute these decompositions [S3]. TA was applied to interactive large volume visualization by Suter et al., who introduced local TA for every brick in a multiresolution hierarchy, a tensor-specific quantization scheme, and an optimized GPU reconstruction method [S3, S5]. It is particularly noted for its ability to highlight multiscale features in microstructural data, such as dental enamel growth patterns [S3].

## Sources
[S1] Interactive Multiresolution and Multiscale Visualization of Large Volume Data, https://doi.org/10.2312/diss.20138322, fetched 2026-10-26 (unofficial)
[S2] A Study on the Compression of Voxel Simulation Results Using Tensor Decomposition, http://www.sapub.org/global/showpaperpdf.aspx?doi=10.5923/j.jmea.20160605.01, fetched 2026-10-26 (unofficial)
[S3] PDF Interactive Multiscale Tensor Reconstruction for Multiresolution Volume ..., https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
[S4] Application of Tensor Approximation to Multiscale Volume Feature Representations, https://doi.org/10.2312/pe/vmv/vmv10/203-210, fetched 2026-10-26
[S5] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S6] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S7] PDF Tensor Approximation Properties for Multiresolution and ... - UZH, https://www.ifi.uzh.ch/dam/jcr:00000000-4c65-90f4-0000-00006dc77a03/TA_Poster.pdf, fetched 2026-10-29 (unofficial)
