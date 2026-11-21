# Hash-based Gating

Also called: Hash-based dispatching, Hash expert selection

## Overview
Hash-based gating is a mechanism for selecting one of $N$ hash experts for a given 3D point $x$ by integrating the dispatching logic directly into the hash table lookup operation [S1]. It solves the efficiency bottleneck of traditional Mixture-of-Experts (MoE) routing by eliminating explicit input feature reorganization and the associated matrix multiplications [S1]. Unlike MLP-based gating used in Switch-NeRF, it reduces training time and memory overhead while maintaining or improving rendering quality [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D point coordinate $x \in \mathbb{R}^3$.
- Output: The index of the selected hash expert $e \in \{0, \dots, N-1\}$ and the corresponding feature representation from that expert.

**Data Structures:**
- $N$ hash experts, where each expert consists of a hash grid with logarithmically spaced base and desired resolutions [S1].
- A gating network that maps $x$ to an expert index. The sources specify the use of an MLP+Norm design for the gating network, contrasting it with a linear gating baseline [S1].
- Hash tables for each expert, following Instant-NGP settings [S1].

**Method:**
1. **Positional Encoding:** Compute the positional encoding $PE(x)$ of the input point $x$ [S1].
2. **Gating Network Forward Pass:** Process $PE(x)$ through a trainable MLP with Layer Normalization (Norm) to produce a routing decision [S1]. The sources note that removing LayerNorm or using a simple linear layer (Linear gating) underperforms due to insufficient capacity [S1].
3. **Hash-Based Dispatching:** Integrate the expert selection into the hash operation. Instead of computing a separate routing vector and gathering features from multiple experts, the gating network's output directly determines which hash table (expert) is queried for the feature representation of $x$ [S1]. This avoids the "gather-scatter" overhead of standard MoE dispatching [S1].
4. **Feature Retrieval:** Retrieve the feature vector from the selected expert's hash grid using the standard hash lookup procedure [S1].
5. **Prediction:** Pass the retrieved feature through prediction heads to estimate density and color [S1].
6. **Loss Computation:** Compute the L2 loss on rendered pixel colors and a gating balance loss $L_b$ to ensure uniform expert utilization [S1]. The weight $\lambda$ for $L_b$ is set to $5 \times 10^{-4}$ [S1].

**Parameters:**
- $N = 8$: Number of hash experts [S1].
- $\lambda = 5 \times 10^{-4}$: Weight for the balance loss $L_b$ [S1].
- Resolutions: Logarithmically spaced base and desired resolutions for the hash grids [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Half-precision training and testing are used, similar to Mega-NeRF [S1].
- **Capacity:** Linear gating fails due to insufficient network capacity; the MLP+Norm design is required [S1].
- **Balance:** Without the balance loss $L_b$, expert utilization may become skewed [S1].

## Subtopics

### [Hash-Based Dispatching](17538-hash-based-dispatching.md)
A specific implementation detail where the expert selection is integrated into the hash lookup, avoiding explicit feature reorganization and matrix multiplications, thereby reducing memory and latency compared to MLP-based dispatching.

## Operating characteristics
**Time and Memory Complexity:**
- The method reduces training time and memory usage compared to MLP-based gating and standard MoE dispatching [S1].
- On the Mega-NeRF dataset, Hash-based dispatching (Switch-NeRF++) uses 2857M test memory vs 2981M for MoE dispatching, and 8707M train memory vs 10386M for MoE [S1].
- Training time is 10.7 hours vs 15.2 hours for MoE dispatching [S1].
- Rendering time is 6.65s vs 13.5s for MoE dispatching [S1].

**Scaling:**
- The method scales well to large-scale scenes (e.g., UrbanBIS) [S1].
- It outperforms Instant-NGP (INGP) with less training time, demonstrating great scalability [S1].

**Strengths and Limitations:**
- **Strengths:** Higher quality images with cleaner and sharper details; superior efficiency compared to Mega-NeRF, Switch-NeRF, and INGP [S1].
- **Limitations:** Requires careful tuning of the balance loss weight $\lambda$ [S1].

**When to use:**
- Use for large-scale neural radiance fields where training efficiency and memory usage are critical, and high-quality rendering is required [S1].

## Background
Originated in Switch-NeRF++ [S1]. The method builds on Instant-NGP hash encoding settings and addresses the scalability issues of large-scale NeRF methods like Mega-NeRF [S1]. It was developed to provide a more efficient alternative to MLP-based gating and standard MoE dispatching in the context of heterogeneous mixture of scene experts [S1].

## Sources
[S1] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
