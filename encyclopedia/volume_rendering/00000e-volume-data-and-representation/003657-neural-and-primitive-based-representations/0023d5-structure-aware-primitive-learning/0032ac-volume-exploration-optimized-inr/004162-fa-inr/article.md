# FA-INR

Also called: Explorable INR

## Overview
FA-INR is a structure-aware primitive learning method for volume exploration that replaces Explorable INR's fixed spatial encoding with a learnable cross-attention mechanism over a memory bank and Mixture-of-Experts (MoE) routing to concentrate capacity in spatially complex regions [S1, unofficial]. Unlike Explorable INR, FA-INR provides no uncertainty estimates, which limits its applicability in safety-critical scientific workflows [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ and simulation parameters $\mathbf{p}$.
- **Output:** Field value $y$.

**Data Structures:**
1. **Memory Bank:** A set of learnable feature vectors $\mathcal{M} = \{\mathbf{m}_1, \dots, \mathbf{m}_N\}$, where $N$ is the bank size [G].
2. **Mixture-of-Experts:** A set of expert networks $\{E_1, \dots, E_K\}$, where each $E_j$ maps features to output values [G].
3. **Gate Network:** A routing network $G(\cdot)$ that outputs weights for the experts [G].

**Method Steps:**
1. **Query Construction:** Combine positional encoding of $\mathbf{x}$ and a per-parameter embedding of $\mathbf{p}$ to form a query vector [G].
2. **Cross-Attention:** Use the query to compute attention weights over $\mathcal{M}$ and aggregate the attended features [G].
3. **MoE Routing:** Pass the aggregated features through $G$ to compute expert weights [G].
4. **Expert Evaluation & Aggregation:** Compute the output of each expert on the aggregated features and combine them using the computed weights [G].

**Parameters:**
- Memory Bank Size $N$: Not given in the sources.
- Number of Experts $K$: Not given in the sources.
- Feature Dimension $d$: Not given in the sources.

**Edge Cases and Pitfalls:**
- The method provides no uncertainty estimates, unlike Explorable INR which assumes Gaussian-distributed outputs for uncertainty propagation [S1, unofficial].
- Precision issues may arise if the attention mechanism fails to distinguish between similar spatial patterns in complex regions [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fa_inr_forward(x, p, memory_bank, experts, gate_network):
    # x: spatial coordinates (3,)
    # p: simulation parameters (k,)
    # memory_bank: list of learnable vectors (N, d)
    # experts: list of K expert networks
    # gate_network: routing network

    # [G] Standard positional encoding for spatial coordinates
    pos_enc = [math.sin(x[i]) for i in range(len(x))]
    # [G] Per-parameter embedding (simplified as identity for demonstration)
    param_emb = list(p)
    # [G] Concatenate to form query
    query = pos_enc + param_emb

    # Cross-attention over memory bank
    # [G] Scaled dot-product attention
    d = len(query)
    scores = [sum(query[i] * memory_bank[j][i] for i in range(d)) / math.sqrt(d) for j in range(len(memory_bank))]
    max_score = max(scores)
    attention_weights = [math.exp(s - max_score) for s in scores]
    total = sum(attention_weights)
    attention_weights = [w / total for w in attention_weights]
    attended_features = [sum(attention_weights[j] * memory_bank[j][i] for j in range(len(memory_bank))) for i in range(d)]

    # MoE routing
    gate_logits = gate_network(attended_features)
    max_logit = max(gate_logits)
    expert_weights = [math.exp(g - max_logit) for g in gate_logits]
    total_w = sum(expert_weights)
    expert_weights = [w / total_w for w in expert_weights]

    # Expert evaluation and aggregation
    output = 0.0
    for j in range(len(experts)):
        output += expert_weights[j] * experts[j](attended_features)
    return output
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On three datasets (MPAS-Ocean with 11.8 M unstructured grid points, Nyx at $512^3$, and CloverLeaf3D at $128^3$), FA-INR outperformed Explorable INR by 5–10% in PSNR [S1, unofficial].
- **Model Size:** Uses 1.5–6.7× fewer parameters (1.10–9.65 M vs. 7.38–14.73 M) compared to Explorable INR [S1, unofficial].
- **Inference Speed:** 1.70 s inference per field on an A100 GPU [S1, unofficial].
- **Strengths:** Concentrates capacity in spatially complex regions via MoE and attention; more parameter-efficient than Explorable INR [S1, unofficial].
- **Limitations:** Provides no uncertainty estimates, limiting applicability in safety-critical scientific workflows [S1, unofficial].

## Background
FA-INR was introduced to address the limitations of fixed spatial encoding in Explorable INR [S1, unofficial]. It replaces static encoding with dynamic, learnable mechanisms to better adapt to complex spatial patterns in scientific simulation data [S1, unofficial]. The method is discussed in the context of deep learning methods for scientific volume representations [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
