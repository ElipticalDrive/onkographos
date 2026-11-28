# NLR Baseline


## Overview
The NLR Baseline implements the Neural Lumigraph Representation (NLR) method for novel view synthesis [S1, unofficial]. The source does not specify the exact network architecture or how it differs from standard NeRF MLPs [S1, unofficial]. It serves as a reference for the original NLR formulation before meta-learning enhancements [S1, unofficial].

## How it works
The method builds on the core differentiable volume rendering pipeline defined in [NeRF Training Loop](1522-training-loop-setup.md) [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of calibrated camera poses and corresponding RGB images [S1, unofficial].
*   **Output:** A trained neural network model capable of rendering novel views [S1, unofficial].

**Data Structures:**
The source does not specify the exact internal memory layout or bit-packing of the NLR network weights or intermediate tensors [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Sampling:** Generate rays from camera poses as defined in the parent pipeline [S1, unofficial].
2.  **Network Query:** Evaluate the NLR neural network at sample points along the ray to obtain volume density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].
3.  **Volume Rendering:** Compute the predicted pixel color $\hat{\mathbf{C}}(\mathbf{r})$ using the discretized integral:
    $$ \hat{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance and $\delta_i$ is the distance between samples [S1, unofficial].
4.  **Loss Computation:** Minimize the difference between rendered and ground truth images, typically using Mean Squared Error [S1, unofficial].
5.  **Optimization:** Update network parameters using gradient-based optimization [S1, unofficial].

**Parameters:**
*   **Resolution:** Training and testing views are processed at 800 × 600 resolution [S1, unofficial].
*   **Training Time:** Approximately 2.21 seconds per iteration [S1, unofficial].
*   **Rendering Time:** Approximately 0.025 seconds per frame [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Convergence Speed:** NLR iterations take 2.21 seconds, which is slower than NeRF (0.40 sec/iteration) and IDR (0.2 sec/iteration) [S1, unofficial]. The source does not state that the NLR baseline requires fewer iterations to converge; this property is attributed to NLR++/MetaNLR++ variants due to larger ray-batch sizes optimizing entire images instead of randomly selected rays [S1, unofficial].
*   **Implementation Access:** The baseline implementation was accessed directly from the original authors [S1, unofficial].

## Subtopics

### MetaNLR Baseline
MetaNLR wraps the NLR implementation with an outer meta-learning loop using Reptile to learn the initialization of the shape and color networks. It differs from the baseline by performing meta-training and meta-testing on the same scenes and specializing to test scenes via inner-loop updates on training/testing views at 800 × 600 resolution. It is used when rapid scene adaptation is required. See [MetaNLR Baseline](11349-meta-nlr-baseline.md) for details.

## Operating characteristics
*   **Time Complexity:** Training iteration time is 2.21 seconds [S1, unofficial]. This is slower than NeRF (0.40 sec/iteration) and IDR (0.2 sec/iteration) [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Rendering time is 0.025 seconds per frame [S1, unofficial].
*   **Scaling:** NLR++/MetaNLR++ variants require significantly larger ray-batch sizes, leading to longer iteration times (7.69 sec/iteration with shape optimization) but fewer total iterations to converge because they optimize for multiple entire images instead of randomly selected rays [S1, unofficial].
*   **Strengths:** Not given in the sources.
*   **Limitations:** Slower per-iteration training time compared to NeRF and IDR [S1, unofficial].

## Background
The source does not provide publication dates or original authors beyond referencing [3] for NLR and [10] for Reptile [S1, unofficial]. The baseline implementation was accessed directly from the NLR authors [S1, unofficial]. MetaNLR applies an outer Reptile meta-learning loop to learn the initialization of the shape and color networks [S1, unofficial].

## Sources
[S1] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
