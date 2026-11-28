# Mixture-of-Experts Implicit Neural Representation

Also called: MoE-INR

## Overview
Mixture-of-Experts Implicit Neural Representation (MoE-INR) is an architecture for time-varying volumetric data compression that automates the subdivision of spatiotemporal fields and dynamically assigns input coordinates to specialized expert networks [S1]. It solves the limitations of conventional INRs (which struggle with complex features using a single network) and grid-based INRs (which suffer from boundary discontinuity artifacts due to manual decomposition) by using a shared encoder to maintain cross-boundary continuity while allowing experts to specialize in distinct subdomains [S1]. Unlike siblings like [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md) which may use different routing or ensemble strategies, MoE-INR specifically targets scientific visualization datasets with a unified framework that can emulate conventional, grid-based, or ensemble INRs by adjusting the number of experts and pre-training schemes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A spatiotemporal coordinate $\mathbf{x} = (x, y, z, t)$ and a set of ground truth voxel values $v$ for training.
*   **Output:** A predicted scalar voxel value $\hat{v}$ and a probability distribution over $k$ experts.

**Data Structures and Layout:**
The architecture consists of three components defined by the number of initial neurons $M$ and the number of experts $k$ [S1].

1.  **Policy Network:**
    *   Structure: Fully connected (FC) layers with $\sin(\cdot)$ activation, followed by a final FC layer with Softmax activation.
    *   Parameters: $4M + M + M^2 + M + 9M \times k + k$ [S1].
    *   Function: Outputs a probability vector $\mathbf{p} \in \mathbb{R}^k$.

2.  **Shared Encoder:**
    *   Structure: Positional encoding (learnable Fourier transformation) $\rightarrow$ FC layers with $\sin(\cdot)$ activation $\rightarrow$ Residual block with bottleneck.
    *   Parameters: $4M + M + 8M^2 + 4M + 32M^2 + 8M + 36M^2 + 12M$ [S1].
    *   Function: Maps $\mathbf{x}$ to a high-dimensional compact representation $\mathbf{h}$.

3.  **Expert Decoders:**
    *   Structure: $k$ parallel networks. Each consists of FC layers with $\sin(\cdot)$ activation, followed by a final FC layer with no activation.
    *   Parameters per expert: $8M + 1$. Total: $k \times (8M + 1)$ [S1].
    *   Function: Maps $\mathbf{h}$ to a scalar value.

**Numbered Steps:**

1.  **Pre-training (Initialization):**
    *   Optimize only the Policy Network and Shared Encoder.
    *   **Supervised Pre-training:** Generate ground truth assignments $y_c$ for coordinates using one of four methods:
        *   *Random partition:* Random expert assignment.
        *   *Spatial partition:* $k$ non-overlapping spatial blocks.
        *   *Temporal partition:* $k$ non-overlapping time intervals.
        *   *Voxel clustering:* $k$-means clustering on voxels.
        *   Loss: Cross-entropy $\mathcal{L}_{pre} = -\frac{1}{k} \sum_{i=1}^k \log \left( \frac{e^{p_i}}{\sum_{j=1}^k e^{p_j}} \right)$ where $p_i$ is the logit for expert $i$ [S1].
    *   **Unsupervised Pre-training (Load Balancing):**
        *   Loss: $\mathcal{L}_{lb} = \frac{1}{N^2} \sum_{j=1}^k c_j s_j$, where $c_j$ is the count of coordinates assigned to expert $j$, $s_j$ is the sum of assignment probabilities for expert $j$, and $N$ is total coordinates [S1].
    *   *Note:* Voxel clustering yields the best results; random partition and load balancing perform worse than no pre-training [S1].

2.  **Training (Joint Optimization):**
    *   Optimize Policy Network, Shared Encoder, and all Expert Decoders jointly.
    *   **Forward Pass:**
        1.  Compute $\mathbf{p} = \text{Softmax}(G(\mathbf{x}))$ using the Policy Network.
        2.  Compute $\mathbf{h} = E_{shared}(\mathbf{x})$ using the Shared Encoder.
        3.  Compute $\hat{v}_j = E_j(\mathbf{h})$ for all $j=1 \dots k$ using Expert Decoders.
    *   **Loss Calculation:**
        *   Weighted MSE: $\mathcal{L}_{train} = \sum_{j=1}^k p_j ||\hat{v}_j - v||^2$, where $v$ is the ground truth voxel value [S1].
    *   **Inference:**
        *   Select expert $j^* = \arg\max_j p_j$.
        *   Output $\hat{v} = E_{j^*}(\mathbf{h})$ [S1].

**Parameters:**
*   $M$: Number of initial neurons (user-specified).
*   $k$: Number of experts. Optimal value is 7 for tested datasets (e.g., Ionization PD, Vortex) [S1].
*   **Parameter Allocation:** The "Larger Encoder" strategy is optimal, allocating ~96.9% of parameters to the shared encoder, ~0.8% to experts, and ~2.3% to the policy network (for Vortex dataset, CR 5,065) [S1].

**Pitfalls and Edge Cases:**
*   **Too many experts:** Setting $k > 7$ degrades performance (e.g., PSNR drops) because the policy network struggles to correctly classify coordinates, leading to misclassifications [S1].
*   **Bad Pre-training:** Random partition and load balancing pre-training fail to produce meaningful assignments, resulting in inferior performance compared to no pre-training [S1].
*   **Boundary Discontinuity:** Avoided by using a shared encoder; grid-based INRs without shared encoders suffer from block artifacts [S1].

## Operating characteristics
*   **Complexity:**
    *   **Memory:** Scales linearly with the number of experts $k$. Memory Usage (MU) for Ionization (PD) dataset (CR 10,078) ranges from 8,177 MB ($k=2$) to 21,745 MB ($k=9$) [S1].
    *   **Time:** Compression Time (CT) increases with $k$. For Ionization (PD), CT ranges from 11.83 hours ($k=2$) to 25.36 hours ($k=9$) [S1].
*   **Performance:**
    *   Outperforms conventional INRs (single network) and grid-based INRs (manual decomposition) in PSNR and LPIPS across various compression ratios (CR) [S1].
    *   On Vortex dataset (CR 5,065, $k=7$, Larger Encoder): Best PSNR/LPIPS achieved [S1].
    *   On Ionization (PD) dataset (CR 10,078, $k=7$): PSNR 58.76 dB, LPIPS 0.024 [S1].
    *   On Ionization (T) dataset (CR 5,033, $k=7$, Voxel Clustering Pre-training): PSNR 56.71 dB, LPIPS 0.047 [S1].
*   **Strengths:**
    *   Automates irregular subdivisions without explicit storage of mapping.
    *   Shared encoder ensures boundary continuity.
    *   Unified framework accommodating conventional, grid-based, and ensemble INR types [S1].
*   **Limitations:**
    *   Conventional INRs (single encoder/decoder) have lower computational cost for visualization than MoE-INR with $k>1$ due to fewer matrix multiplications, but MoE-INR with $k=1$ reduces to conventional INR [S1].
    *   Ensemble INRs (e.g., RMDSRN) decode a single coordinate using multiple experts and average results; MoE-INR typically uses single expert selection but can function as an ensemble INR if configured to average [S1].

## Background
The mixture-of-experts (MoE) paradigm was originally proposed by Nowlan and Hinton [26], applying supervision to a system of separate networks where each handles a subset of training samples and specializes in a different part of the data space [S1]. MoE-INR was developed to address the limitations of existing INRs for scientific data compression, specifically the inability of conventional INRs to preserve complex features and the boundary discontinuity issues of grid-based INRs [S1]. It is designed for time-varying volumetric data compression and representation in scientific visualization, tested on datasets including argon bubble, combustion, ionization, Tangaroa, and vortex [S1].

## Sources
[S1] MoE-INR: Implicit Neural Representation with Mixture-of-Experts for ..., https://academicweb.nd.edu/~cwang11/papers/vis25-moeinr.pdf, fetched 2026-10-29
