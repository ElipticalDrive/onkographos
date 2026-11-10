# Sparse Domain Decomposition with Expert Networks

Also called: Neural Sparse Volume Representation, NeuralVDB MoE

## Overview
Sparse Domain Decomposition with Expert Networks (NeuralVDB) is a hybrid method that encodes sparse volumetric data by combining explicit hierarchical tree structures (VDB) with implicit neural representations for voxel values and topology [S1, unofficial]. It solves the problem of high memory consumption in standard sparse volumes by replacing stored values with trained neural networks, enabling high-resolution storage and random access while maintaining compatibility with existing VDB pipelines [S1, unofficial]. Unlike standard neural implicit representations that require normalized input spaces, NeuralVDB deliberately over-fits to the input volume to memorize it, trading statistical generalization for the flexibility to handle arbitrary input domains [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of posed images or a standard VDB volume (scalar, vector, or binary mask) defined over a coordinate span.
- **Output:** A NeuralVDB structure consisting of a sparse tree (e.g., Hash grid) and a set of expert neural networks (MLPs) and gate functions associated with subdomains.

**Data Structures:**
The method uses a hierarchical VDB structure, specifically a Hash grid. The notation `[Hash, L_1, L_2, ...]` defines the tree levels.
- **Tree Structure:** A sparse hash root followed by dense levels with logarithmic sizes. For example, `[Hash, 5, 4, NN(3)]` indicates a Hash grid with 5 levels, where levels 1 and 2 are dense with sizes corresponding to domains $4096^3$ and $128^3$ respectively, and level 0 (leaves) uses neural networks [S1, unofficial].
- **Expert Networks:** Multi-Layer Perceptrons (MLPs) partition the coordinate span into partially overlapping subdomains. Each MLP maps floating-point voxel coordinates $(x, y, z)$ to the relevant value type (scalar, vector, or binary mask) [S1, unofficial].
- **Gate Functions:** Non-learnable clamped tent functions that define the influence of each subdomain, including a halo region for overlap [S1, unofficial].

**Algorithm Steps:**

1.  **Domain Decomposition:** Partition the entire coordinate span of the sparse volume into partially overlapping subdomains [S1, unofficial].
2.  **Network Initialization:** For each subdomain, initialize an MLP.
    - For scalar and vector values, use the MLP as a regression network.
    - For binary masks (indicating active value/child status), use an MLP classifier [S1, unofficial].
3.  **Training:**
    - Optimize the MLPs using a loss function. For regression, use Mean Squared Error (MSE):
      $$L_{MSE}(f, \hat{f}) = \frac{1}{N} \sum_{i=1}^N (f - \hat{f}_i)^2$$
      where $f$ is the target value and $\hat{f}_i$ is the predicted value [S1, unofficial].
    - For SDF data, scale targets to $[-1, 1]$. For density volumes, keep the original range $[0, 1]$ [S1, unofficial].
    - Train concurrently on multiple GPUs by assigning groups of subdomains to each GPU [S1, unofficial].
    - Use Fourier Feature Mappings (FFM) as the activation function for the examples provided [S1, unofficial].
4.  **Inference (Random Access):**
    - Query the VDB tree structure. If the query point falls into a tile (non-voxel), prune the query [S1, unofficial].
    - Otherwise, pass the input coordinates to the gate functions and expert networks [S1, unofficial].
    - Calculate the output $\hat{y}$ as the weighted sum of expert outputs:
      $$\hat{y} = \sum_{k=1}^n G(\mathbf{x})_k E_k(\mathbf{x})$$
      where $n$ is the number of subdomains, $G(\mathbf{x})_k$ is the gate function for subdomain $k$, and $E_k(\mathbf{x})$ is the expert network output [S1, unofficial].
    - Note that gate functions are not learnable [S1, unofficial].
    - A single input coordinate can activate multiple gate functions (up to eight) due to overlapping halos. Average the evaluated values weighted by the gate functions [S1, unofficial].
    - Examine the gate function first to determine which networks to invoke; only perform computation for networks with non-zero gate values [S1, unofficial].

**Parameters:**
- **Maximum active gate functions:** 8 per coordinate [S1, unofficial].
- **Target Error Metrics:**
  - For narrowband level sets (SDF): Intersection of Union (IoU) > 99% [S1, unofficial].
  - For density volumes: Root Mean Squared Error (RMSE) < 0.1 [S1, unofficial].
- **Modified Chamfer Distance (mCD):** Used for level set evaluation:
  $$mCD = \frac{1}{2N_1} \sum_{i=1}^{N_1} SDF_2(\mathbf{v}_{1,i}) + \frac{1}{2N_2} \sum_{i=1}^{N_2} SDF_1(\mathbf{v}_{2,i})$$
  where $\mathbf{v}_1 \in V_1$ and $\mathbf{v}_2 \in V_2$ are sampling points from the ground truth and reconstructed SDFs respectively [S1, unofficial].
- **Temporal Warm-starting:** Initialize the encoder for the current frame using converged weights from the previous frame to boost performance and temporal coherence [S1, unofficial].
- **Multi-GPU Heuristic:** Use multiple GPUs if average active voxels $\times$ number of blocked ranges $\ge 200 \times$ number of GPUs [S1, unofficial].
- **Blocked Range Size:** $512^3$ for job distribution [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Overlapping Halos:** Multiple gate functions may activate for a single coordinate. This is handled by averaging the weighted outputs [S1, unofficial].
- **Load Balancing:** Sub-linear strong-scaling in multi-GPU encoding/decoding due to sequential steps (I/O, decomposition, gathering) and imbalanced subdomains [S1, unofficial].
- **Reconstruction Error:** The method is lossy for values (not topology), resulting in "blobby" error patterns on iso-surfaces [S1, unofficial].

## Subtopics

### [Offline Sequential Reconstruction](8790-offline-sequential-reconstruction.md)
Offline Sequential Reconstruction is a method for decoding a NeuralVDB representation into a standard VDB tree in a single sequential pass, prioritizing memory efficiency over reconstruction time. It is designed for scenarios where the data is stored offline (e.g., on disk) or transmitted over low-bandwidth networks, and the entire volume must be reconstructed into memory. Unlike online random access methods, it does not support immediate point queries without full reconstruction, but it achieves a significantly smaller in-memory footprint by leveraging the compressed neural structure for the lower levels of the VDB hierarchy.

## Operating characteristics
**Time and Memory Complexity:**
- **Memory:** NeuralVDB achieves significant compression ratios. For SDF models, compression ratios up to 61.2 are achieved with `[Hash, 5,NN(4),NN(3)]`. For density volumes, ratios as high as 140.9 (Chameleon model) are possible [S1, unofficial]. The `[Hash, 5, 4,NN(3)]` variant reduces size to 6.268% of the original VDB [S1, unofficial].
- **Random Access:** NeuralVDB random access is more expensive than NanoVDB's nearest neighbor and tri-linear interpolation, but similar to third-order (tri-cubic) interpolation and cheaper than pure neural network predictions [S1, unofficial].
- **Encoding/Decoding:** Sub-linear strong-scaling on multiple GPUs due to sequential steps and load balancing issues [S1, unofficial].

**Measured Performance:**
- **Hardware:** NVIDIA RTX A40 GPUs, AMD EPYC 7502 CPU [S1, unofficial].
- **Rendering:** Bunny model ray-marching times: `[Hash, 5, 4, 3]` zeroth-order 75 ms, first-order 97 ms, third-order 1660 ms; `[Hash, 5, 4,NN(3)]` 1316 ms [S1, unofficial].
- **Temporal Boost:** Performance boosts of 2.4x (LeVeque's Test), 2.5x (Smoke Plume), 1.6x (Ship Breach), 1.2x (Dust Impact), and 3.1x (Tornado) using temporal warm-starting [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** High memory efficiency, compatibility with existing VDB pipelines, fast random access compared to pure neural methods, and support for temporal coherence [S1, unofficial].
- **Limitations:** Lossy compression leads to "blobby" errors; sub-linear scaling on multi-GPU setups; random access is slower than standard sparse interpolation methods [S1, unofficial].

## Background
NeuralVDB was implemented in C++17 using CUDA and PyTorch, based on concepts from NanoVDB and OpenVDB [S1, unofficial]. It addresses the limitations of earlier neural-implicit representations (e.g., DeepSDF) that require normalized input spaces, by over-fitting to the input volume to allow arbitrary inputs [S1, unofficial]. The method distinguishes itself from sparsely-gated Mixture of Experts (MoE) by using non-learnable gate functions [S1, unofficial].

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
