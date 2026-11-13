# Neural Implicit Network Compression

Also called: Coordinate-based MLP Compression, SIREN-based INR Compression, neurcomp

## Overview
Neural Implicit Network Compression (INR) replaces explicit volumetric grids with a coordinate-based neural network that maps spatial coordinates directly to field values, serving as simultaneously compressed storage and a queryable data source [S1, unofficial]. Unlike traditional lossy compressors (e.g., SZ, ZFP) that produce bitstreams requiring full decoding before visualization, INRs allow direct evaluation at arbitrary resolutions [S1, unofficial]. The method sets itself apart by leveraging the continuous nature of neural networks to achieve high compression ratios on scientific volumes, often outperforming tensor decomposition methods at extreme compression levels, though it typically lacks the interactive rendering speeds of cached grid-based approaches [S1, unofficial].

## How it works
**Inputs:** A volumetric dataset $V$ defined on a 3D grid (e.g., $N^3$ points) and a target compression ratio or network size constraint.
**Outputs:** A trained neural network (typically a Multi-Layer Perceptron or MLP) with quantized weights, representing the field $f(x, y, z)$.

**Data Structures:**
1.  **Network Weights:** A set of matrices and bias vectors for the MLP layers. In the baseline method (neurcomp), these are quantized to 9 bits using k-means clustering [S1, unofficial].
2.  **Training Data:** A set of sampled coordinate-value pairs $\{(x_j, y_j, z_j), v_j\}$ derived from the original volume $V$ [S1, unofficial].

**Algorithm Steps:**
1.  **Sampling:** Sample spatial coordinates from the volume. For static volumes, uniform sampling or entropy-based adaptive sampling (concentrating on high-information blocks) may be used [S1, unofficial].
2.  **Encoding:** Apply positional encoding $\gamma(x)$ to input coordinates to help the network capture high-frequency details [S1, unofficial].
3.  **Network Forward Pass:** Pass encoded coordinates through a coordinate-based MLP. The baseline method uses a SIREN (Sinusoidal Representation Network) architecture with residual connections [S1, unofficial].
    *   Let $W_l$ be the weight matrix and $b_l$ be the bias vector for layer $l$.
    *   For layer $l$, the output $h_l$ is computed as:
        $$ h_l = \sin(W_l h_{l-1} + b_l) $$
        where $h_0 = \gamma(x)$ is the encoded input [S1, unofficial].
4.  **Loss Calculation:** Minimize the reconstruction error between the network output and the sampled volume values. An optional gradient regularization term can be added to improve isosurface fidelity, though this may slightly reduce PSNR [S1, unofficial].
    $$ \mathcal{L} = \| f(x) - V(x) \|^2 + \lambda \| \nabla f(x) - \nabla V(x) \|^2 $$
    where $\lambda$ is a hyperparameter balancing reconstruction and gradient fidelity [S1, unofficial].
5.  **Training:** Train the network using gradient descent on a GPU (e.g., V100). Training time varies by volume size and complexity, typically requiring hours per volume for the baseline method [S1, unofficial].
6.  **Quantization:** After training, quantize the network weights to reduce storage size. The baseline method uses k-means clustering to quantize weights to 9 bits [S1, unofficial].

**Parameters:**
*   **Network Depth/Width:** Not specified in the baseline notes; typically small for efficient compression.
*   **Quantization Bits:** 9 bits for weights in the baseline neurcomp method [S1, unofficial].
*   **Gradient Regularization Weight ($\lambda$):** Optional; improves isosurface fidelity at the cost of PSNR [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Interactive Rendering:** The baseline method could not render interactively from the volume due to the computational cost of evaluating the network for every ray [S1, unofficial].
*   **Training Time:** Training can be slow (hours per volume on a V100 GPU) [S1, unofficial].
*   **Precision:** Quantization introduces error; 9-bit quantization is a trade-off between size and fidelity [S1, unofficial].

## Subtopics

### Time-varying INR compression via Laplacian pyramid
A method that decomposes time-varying volumes using a Laplacian pyramid with per-block MLPs to improve decoding speed [S1, unofficial]. It achieves decoding speeds 13–30× faster than neurcomp on datasets up to $1,728^3$ [S1, unofficial].

### MVNet
A method for compressing multivariate data by formulating it as multi-task learning, using a Residual SIREN with a $v$-dimensional output layer (where $v$ is the number of physical variables) and sharing all intermediate parameters across variables [S1, unofficial]. It preserves inter-variable statistical dependencies (Pearson correlation and mutual information) better than traditional compressors. On a 100-variable climate dataset (1568 MB), it achieved 49.68 dB PSNR at 1298:1 compression, outperforming TTHRESH (48.62 dB at 343:1) [S1, unofficial]. It has only been tested on static datasets [S1, unofficial].

### KD-INR
A two-stage pipeline for compressing time-varying data. Stage 1 compresses each timestep independently with a small INR with bottleneck layers, using entropy-based adaptive sampling to concentrate training on high-information blocks. Stage 2 merges all per-timestep teacher models into a single student via offline knowledge distillation (an $O(n)$ procedure) [S1, unofficial]. On time-varying datasets up to 23.44 GB (argon bubble, 150 timesteps), it outperformed CoordNet, NeurComp, and SIREN at compression ratios from 733 to 10,000, with per-timestep compression requiring 1–6 minutes on a single GPU [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is significant, requiring hours per volume for the baseline neurcomp method on a V100 GPU [S1, unofficial]. Variants like KD-INR reduce per-timestep training to 1–6 minutes [S1, unofficial].
*   **Memory Complexity:** Storage is determined by the size of the quantized network weights. For neurcomp, 9-bit weight quantization is used [S1, unofficial].
*   **Performance:** On scalar fields up to $512^3$, neurcomp outperformed tensor decomposition (TTHRESH) at high compression ratios, achieving 47.9 dB PSNR at 545:1 compression on the Rayleigh-Taylor dataset [S1, unofficial].
*   **Strengths:** High compression ratios; continuous representation allows arbitrary resolution; queryable data source.
*   **Limitations:** Slow training; lack of interactive rendering in baseline methods; potential for slight PSNR reduction when using gradient regularization [S1, unofficial].
*   **When to use:** When storage is constrained and interactive rendering is not required, or when continuous representation is beneficial.
*   **When not to use:** When interactive rendering speed is critical or when training time is prohibitive.

## Background
Neural Implicit Network Compression for scientific volumes was introduced by neurcomp [102, unofficial], which used a coordinate-based SIREN MLP with residual connections and 9-bit weight quantization via k-means clustering [S1, unofficial]. It was the first INR compression method for scientific volumes [S1, unofficial]. The method addresses the strain on storage systems and I/O pipelines caused by full-resolution scientific volumes [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
