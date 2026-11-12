# Mixture-of-Experts Neural Fields

Also called: MoE-INR

## Overview
Mixture-of-Experts Neural Fields (MoE-INR) is a neural representation for scientific volume data that partitions the spatial domain into distinct subdomains, each modeled by a specialized expert decoder [S1, unofficial]. It solves the problem of modeling complex, large-scale scientific datasets with high fidelity and compression by allowing local experts to specialize in specific geometric or physical features, while a shared encoder maintains global boundary continuity [S1, unofficial]. It distinguishes itself from independent per-block INRs by guaranteeing continuity across subdomain boundaries through the shared encoder, and from single-network INRs by avoiding the capacity bottleneck of a single model trying to represent heterogeneous data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x} = (x, y, z)$ representing points in the volume.
*   **Output:** Field values (e.g., density, temperature) associated with $\mathbf{x}$.

**Data Structures:**
1.  **Shared Encoder:** A neural network $E(\cdot)$ that extracts global features from the input coordinates.
2.  **Policy Network (Router):** A neural network $R(\cdot)$ that computes routing probabilities for each expert.
3.  **Expert Decoders:** A set of $K$ independent neural networks $\{D_1, D_2, ..., D_K\}$, where each $D_k$ specializes in a spatially distinct subdomain. In the cited implementation, $K=7$ [S1, unofficial].

**Parameters:**
*   $K$: Number of experts. Typical value: 7 [S1, unofficial].
*   Pretraining: Required for the policy network using voxel-clustering labels [S1, unofficial].

**Steps:**
1.  **Feature Extraction:** Pass the input coordinate $\mathbf{x}$ through the shared encoder to obtain a feature representation $f = E(\mathbf{x})$. This ensures that all experts share a common understanding of the global context, maintaining boundary continuity [S1, unofficial].
2.  **Routing:** Pass the input coordinate $\mathbf{x}$ through the policy network $R(\mathbf{x})$ to determine the probability distribution over the $K$ experts. The policy network is pre-trained using voxel-clustering labels to prevent collapse [S1, unofficial].
3.  **Expert Selection:** Select the expert $k$ with the highest probability (or sample based on probabilities) from the distribution.
4.  **Decoding:** Pass the input coordinate $\mathbf{x}$ (and potentially the shared features $f$) to the selected expert decoder $D_k$ to predict the field value.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def moe_inr_forward(x, shared_encoder, policy_network, experts):
    """
    x: 3D coordinate tensor of shape (..., 3)
    shared_encoder: Neural network for global feature extraction
    policy_network: Neural network for routing probabilities
    experts: List of K neural network decoders
    """
    # Step 1: Extract shared features for boundary continuity
    shared_features = shared_encoder(x)
    
    # Step 2: Route coordinates to experts
    # Policy network outputs probabilities for each expert
    routing_probs = policy_network(x)
    
    # Select the expert with the highest probability
    # In training, this might involve sampling or straight-through estimator
    expert_index = torch.argmax(routing_probs, dim=-1)
    
    # Step 3: Decode using the selected expert
    # Note: The exact input to the expert decoder (x vs shared_features) 
    # is not fully specified in the source, but typically experts take x.
    # We assume the expert takes the original coordinate x.
    selected_expert = experts[expert_index]
    
    # Forward pass through the selected expert
    # If the expert also uses shared features, they would be concatenated here.
    # Based on [S1, unofficial], experts specialize in subdomains, implying 
    # they take spatial coordinates.
    output = selected_expert(x)
    
    return output
```

**Edge Cases and Pitfalls:**
*   **Router Collapse:** Without pretraining the policy network with voxel-clustering labels, the router collapses to a single expert, degenerating the model into a conventional INR [S1, unofficial].
*   **Training Time:** The method requires significant training time, typically 8–50 hours [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Performance:** Achieves high PSNR with high compression ratios. On combustion data, it achieves 60.24 dB PSNR at 1,951× compression. On ionization data, it achieves 50.73 dB PSNR at 24,878× compression [S1, unofficial].
*   **Training Time:** High computational cost for training, ranging from 8 to 50 hours [S1, unofficial].
*   **Comparison:** Outperforms CoordNet, NeurComp, and SIREN on time-varying datasets up to 23.44 GB with compression ratios from 733 to 10,000 [S1, unofficial].
*   **Strengths:** High compression ratios, high fidelity (PSNR), boundary continuity via shared encoder.
*   **Limitations:** Long training time, risk of router collapse if not pre-trained.

## Background
MoE-INR was introduced as a mixture-of-experts approach for neural representations of scientific volume data [S1, unofficial]. It addresses the limitations of single-network INRs and independent per-block INRs by combining specialization with continuity [S1, unofficial]. The method was evaluated on time-varying scientific datasets such as combustion and ionization data [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
