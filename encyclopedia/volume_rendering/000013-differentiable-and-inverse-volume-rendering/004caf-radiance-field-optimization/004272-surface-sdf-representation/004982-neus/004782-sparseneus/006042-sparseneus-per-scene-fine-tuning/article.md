# SparseNeuS Per-Scene Fine-Tuning

Also called: Fine-Tuning Networks, Per-Scene Optimization, SparseNeuS, SparseNeuS Network Architecture and Sampling

## Overview
SparseNeuS Per-Scene Fine-Tuning is a method for refining a neural surface reconstruction from sparse input views by optimizing the geometry encoding volume and SDF network directly for a specific scene [S1]. It differs from the generic SparseNeuS inference by discarding the 2D feature extraction and 3D sparse CNN networks, replacing the CNN-based blending network with a lightweight MLP, and employing a consistency-aware color loss to handle occlusions and noise [S1]. This approach leverages the initialization from the generic network inference to fast converge to a high-quality surface [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images, camera poses, and the pre-trained generic SparseNeuS model parameters (specifically the fine-level geometry encoding volume and the SDF network $f_\theta$) [S1].
*   **Outputs:** Optimized fine-level geometry encoding volume, optimized SDF network $f_\theta$, and the reconstructed 3D surface [S1].

**Data Structures:**
*   **Geometry Encoding Volume:** A 3D grid storing latent features for spatial locations.
*   **SDF Network ($f_\theta$):** An MLP with 4 hidden layers and hidden size 256 [S1].
*   **Blending Network ($f_c'$):** A tiny MLP with 3 hidden layers and hidden size 256 [S1].
*   **Positional Encoding:** Applied to 3D locations (6 frequencies) and view directions (4 frequencies) [S1].

**Numbered Steps:**

1.  **Initialization:** Load the pre-trained generic SparseNeuS model. Discard the 2D feature extraction network and the 3D sparse CNN networks [S1]. Keep the fine-level geometry encoding volume and the SDF network $f_\theta$ [S1].
2.  **Network Replacement:** Replace the CNN-based blending network used in the generic setting with a tiny MLP network $f_c'$ [S1]. This MLP outputs blending weights for a query 3D position $q$ [S1].
3.  **Sampling:** Adopt a hierarchical sampling strategy similar to NeuS [S1]. Sample 64 coarse points and 64 fine points along each query ray [S1].
4.  **Blending Weight Calculation:** For each query 3D position $q$, compute the blending weights $\{w_i^q\}_{i=0}^{N-1}$ using the tiny MLP $f_c'$ [S1]. The inputs to $f_c'$ are:
    *   $PE(q)$: Positional encoding of the 3D position $q$.
    *   $PE(d)$: Positional encoding of the ray direction $d$.
    *   $n_q$: Surface normal at $q$.
    *   $s(q)$: Predicted SDF value at $q$.
    *   $M(q)$: Interpolated geometry encoding feature at $q$ from the fine-level volume [S1].
    
    The formula is:
    $$ \{w_i^q\}_{i=0}^{N-1} = f_c'(PE(q), PE(d), n_q, s(q), M(q)) $$
    where $N$ is the number of input images [S1].
5.  **Loss Computation:** Compute the total loss, which includes the consistency-aware color loss and other terms (e.g., sparseness loss) [S1].
    *   **Consistency-Aware Color Loss:** This loss automatically detects regions lacking consistent projections across views (due to occlusion or noise) and excludes them from optimization [S1]. The loss is defined as:
        $$ \mathcal{L}_{color} = \sum_{r \in \mathbb{R}} O(r) \cdot \mathcal{D}_{pix}(C(r), \tilde{C}(r)) + \sum_{r \in \mathbb{R}} O(r) \cdot \mathcal{D}_{pix}(C(r), \tilde{C}(r)) $$
        *(Note: The source text for the full formula is truncated, but it involves an observation mask $O(r)$, rendered color $C(r)$, and target color $\tilde{C}(r)$ [S1].)*
    *   **Sparseness Loss:** Includes a scaling parameter $\tau$ [S1].
6.  **Optimization:** Directly optimize the fine-level geometry encoding volume and the SDF network $f_\theta$ using the computed losses [S1].

**Parameters:**
*   **SDF Network:** 4 hidden layers, hidden size 256 [S1].
*   **Blending Network ($f_c'$):** 3 hidden layers, hidden size 256 [S1].
*   **Positional Encoding:** 6 frequencies for 3D locations, 4 frequencies for view directions [S1].
*   **Sampling:** 64 coarse samples, 64 fine samples [S1].
*   **Loss Weights:** $\alpha = 0.1$, $\beta = 0.02$ for total loss [S1].
*   **SDF Scaling ($\tau$):** 100 for sparseness loss [S1].
*   **Consistency-Aware Color Loss:** $\lambda_0 = 0.01$, $\lambda_1 = 0.015$ by default [S1].
*   **Data Preparation:** Threshold $\tau = 10$ for detecting invalid black regions in images, followed by dilation and erosion [S1].

**Edge Cases and Pitfalls:**
*   **Ratio Tuning:** The ratio $\lambda_0 / \lambda_1$ in the consistency-aware color loss affects surface completeness vs. cleanliness. Decreasing the ratio keeps more regions, while increasing it excludes more regions for cleaner surfaces [S1].
*   **Noise and Occlusion:** Regions with inconsistent projections across views (due to occlusion or image noise) can lead to inaccurate and distorted surfaces if not handled by the consistency-aware loss [S1].
*   **Background:** Images may contain large black backgrounds and considerable noise, requiring threshold-based denoising and mask processing [S1].

## Operating characteristics
*   **Time Complexity:** Faster than generic SparseNeuS inference due to the replacement of the CNN-based blending network with a smaller MLP [S1].
*   **Memory Complexity:** Reduced memory footprint compared to generic SparseNeuS as the 2D feature extraction and 3D sparse CNN networks are discarded during fine-tuning [S1].
*   **Performance:** Fast convergence to high-quality surfaces due to initialization from the generic network inference [S1].
*   **Strengths:** Handles sparse views effectively; consistency-aware loss improves robustness to noise and occlusion; lightweight blending network speeds up optimization [S1].
*   **Limitations:** Requires per-scene optimization, which may be slower than single-shot inference methods; performance depends on the quality of the generic initialization [S1].
*   **When to Use:** When high-quality surface reconstruction is needed from sparse views and per-scene optimization is feasible [S1].
*   **When Not to Use:** When real-time inference is required or when input views are dense (other methods may be more efficient) [S1].

## Background
SparseNeuS was introduced to address the challenge of neural surface reconstruction from sparse views [S1]. The per-scene fine-tuning method builds upon the generic SparseNeuS framework, which uses a hybrid representation of geometry encoding volume and learnable image features [S1]. The method was developed to improve the quality and speed of surface reconstruction by leveraging the initialization from the generic model and optimizing it for specific scenes [S1]. It is used in applications requiring high-fidelity 3D reconstruction from limited camera views, such as in the DTU dataset evaluations [S1].

## Sources
[S1] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
