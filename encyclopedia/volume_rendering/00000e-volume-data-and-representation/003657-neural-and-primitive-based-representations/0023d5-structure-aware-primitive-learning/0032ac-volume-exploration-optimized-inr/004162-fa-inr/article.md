# FA-INR

Also called: Feature-Adaptive Explorable INR, Feature-Adaptive INR

## Overview
FA-INR is a structure-aware primitive learning method for volume exploration that replaces the fixed spatial encoding of Explorable INR with a learnable cross-attention mechanism and Mixture-of-Experts (MoE) routing [S1, unofficial]. It solves the problem of adapting neural representations to complex, spatially varying patterns in scientific simulation data by concentrating model capacity where it is needed most, while maintaining the ability to query continuous parameter spaces [S1, unofficial]. Unlike its parent Explorable INR, FA-INR does not propagate uncertainty estimates, limiting its use in safety-critical workflows [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ and simulation parameters $\mathbf{p} \in \mathbb{R}^k$.
- **Output:** Field value $y$ (and potentially feature vectors for attention).

**Data Structures:**
1.  **Memory Bank:** A set of learnable feature vectors $\mathcal{M} = \{\mathbf{m}_1, \dots, \mathbf{m}_N\}$, where $N$ is the bank size. Each $\mathbf{m}_i \in \mathbb{R}^d$.
2.  **Mixture-of-Experts:** A set of expert networks $\{E_1, \dots, E_K\}$, where each $E_j$ is a small neural network (e.g., MLP) mapping features to output values.
3.  **Gate Network:** A routing network $G(\cdot)$ that outputs weights for the experts.

**Method Steps:**
1.  **Hybrid Spatial Encoding:** Combine 3D and 2D feature grids with per-parameter embeddings to create an initial feature representation $\mathbf{f}_{init}$ [S1, unofficial]. This captures spatial detail without the memory cost of full multi-dimensional grids.
2.  **Cross-Attention:** Use $\mathbf{f}_{init}$ as a query to attend to the Memory Bank $\mathcal{M}$. Compute attention weights $\alpha_i$ for each memory vector $\mathbf{m}_i$ based on similarity between $\mathbf{f}_{init}$ and $\mathbf{m}_i$. Aggregate the attended features into $\mathbf{f}_{att}$ [S1, unofficial].
3.  **MoE Routing:** Pass $\mathbf{f}_{att}$ through the Gate Network $G$ to compute expert weights $\mathbf{w} = [w_1, \dots, w_K]$ [S1, unofficial].
4.  **Expert Evaluation:** Compute the output of each expert $E_j(\mathbf{f}_{att})$.
5.  **Output Aggregation:** Combine expert outputs using the gate weights: $y = \sum_{j=1}^K w_j E_j(\mathbf{f}_{att})$ [S1, unofficial].

**Parameters:**
-   Memory Bank Size $N$: Not given in the sources.
-   Number of Experts $K$: Not given in the sources.
-   Feature Dimension $d$: Not given in the sources.

**Edge Cases and Pitfalls:**
-   The method assumes Gaussian-distributed outputs for uncertainty propagation in Explorable INR; FA-INR removes this but provides no uncertainty estimates, which is a gap for safety-critical workflows [S1, unofficial].
-   Precision issues may arise if the attention mechanism fails to distinguish between similar spatial patterns in complex regions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fa_inr_forward(x, p, memory_bank, experts, gate_network, feature_grids):
    """
    x: Spatial coordinates (3,)
    p: Simulation parameters (k,)
    memory_bank: List of learnable vectors (N, d)
    experts: List of expert networks (K)
    gate_network: Routing network
    feature_grids: Hybrid 3D/2D grids
    """
    # Step 1: Hybrid Spatial Encoding
    # Combine 3D and 2D feature grids with per-parameter embeddings
    # [G] Assuming feature_grids provides a function to sample features at x
    f_3d = sample_3d_grid(feature_grids.grid_3d, x)
    f_2d = sample_2d_grid(feature_grids.grid_2d, x)
    f_p = parameter_embedding(p)
    f_init = concatenate([f_3d, f_2d, f_p])

    # Step 2: Cross-Attention over Memory Bank
    # Compute attention weights between f_init and memory_bank
    # [G] Assuming standard scaled dot-product attention
    queries = f_init.unsqueeze(0)  # (1, d)
    keys = memory_bank           # (N, d)
    values = memory_bank         # (N, d)
    
    # Attention scores
    scores = dot_product(queries, keys.transpose(-2, -1)) / sqrt(d)
    alpha = softmax(scores, dim=-1)  # (1, N)
    
    # Attended features
    f_att = alpha @ values         # (1, d)
    f_att = f_att.squeeze(0)       # (d,)

    # Step 3: MoE Routing
    # Compute expert weights
    w = gate_network(f_att)        # (K,)
    w = softmax(w, dim=-1)         # (K,)

    # Step 4 & 5: Expert Evaluation and Aggregation
    y = 0.0
    for j in range(len(experts)):
        expert_output = experts[j](f_att)
        y += w[j] * expert_output
    
    return y
```

## Subtopics

### [Explorable INR](6823-volume-exploration-optimized-inr.md)
The parent method using fixed spatial encoding and probabilistic affine forms for uncertainty. Choose when uncertainty quantification is required.

### [SurroFlow](6824-surroflow.md)
Uses normalizing flows for bidirectional prediction. Choose when invertible mapping between parameters and data is needed.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** On three datasets (MPAS-Ocean with 11.8 M unstructured grid points, Nyx at $512^3$, and CloverLeaf3D at $128^3$), FA-INR outperformed Explorable INR by 5–10% in PSNR [S1, unofficial].
-   **Model Size:** Uses 1.5–6.7× fewer parameters (1.10–9.65 M vs. 7.38–14.73 M) compared to Explorable INR [S1, unofficial].
-   **Inference Speed:** 1.70 s inference per field on an A100 GPU [S1, unofficial].
-   **Strengths:** Concentrates capacity in spatially complex regions via MoE and attention; more parameter-efficient than Explorable INR.
-   **Limitations:** Provides no uncertainty estimates, limiting applicability in safety-critical scientific workflows [S1, unofficial].

## Background
FA-INR was introduced as an improvement over Explorable INR [16] to address the limitations of fixed spatial encoding in adapting to complex spatial patterns [S1, unofficial]. It builds on the concept of continuous parameter-space exploration but replaces the static encoding with dynamic, learnable mechanisms [S1, unofficial]. The method is cited in the context of deep learning methods for scientific volume representations [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
