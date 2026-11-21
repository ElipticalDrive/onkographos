# Inter-Gaussian Context Model


## Overview
The Inter-Gaussian Context Model predicts the attributes of a current Gaussian by leveraging context from previously processed Gaussians, thereby reducing the entropy of the quantized anchor attributes [S1, unofficial]. It sets itself apart from the parent [Gaussian Probability Modeling](2122-gaussian-probability-modeling.md) by explicitly structuring the Gaussian set into sequential batches to progressively enrich the available context, distinguishing it from methods that treat Gaussians as independent or use only local spatial neighbors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of 3D Gaussians to be compressed, split into sequential batches $B_1, \dots, B_k$.
- Output: Predicted mean $\mu_g$ and variance for the attributes of the current batch, used to compute the probability distribution for entropy coding.

**Data Structures:**
The Gaussian set is partitioned into $k$ batches. The partitioning is defined by proportions $p_1, \dots, p_k$ such that $\sum p_i = 1$. The attributes are typically color ($f_{col}$) and geometric ($f_{geo}$) components [S1, unofficial].

**Steps:**
1. **Partitioning:** Split the total set of Gaussians into $k$ batches $B_1, \dots, B_k$ according to the specified proportions. The default configuration uses 4 splits with proportions $[17\%, 17\%, 33\%, 33\%]$ [S1, unofficial].
2. **Context Initialization (Batch $B_1$):** For the first batch $B_1$, there is no prior Gaussian context. The mean $\mu_g$ is manually set to $0$ as a placeholder [S1, unofficial]. However, the positional encoding $emb(\mu_g)$ is still computed and provided to the context model to serve as a fallback context signal [S1, unofficial].
3. **Context Aggregation (Batches $B_2 \dots B_k$):** For subsequent batches, the context is derived from the attributes of all previously processed Gaussians in $B_1, \dots, B_{i-1}$. This allows the model to learn dependencies between Gaussians [S1, unofficial].
4. **Prediction:** The context model (typically a neural network) takes the aggregated context and the positional encoding of the current Gaussian to predict the mean $\mu_g$ and variance.
5. **Entropy Coding:** The predicted distribution is used to encode the quantized attributes.

**Parameters:**
- **Number of splits ($k$):** Default is 4. Tested extremes include 2 splits (proportions $95\%, 5\%$) [S1, unofficial].
- **Split Proportions:** Default is $[17\%, 17\%, 33\%, 33\%]$. The strategy ensures an even distribution of Gaussians across 3D space, but allocates fewer Gaussians to early batches to account for their lack of context [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The placeholder $\mu_g=0$ in $B_1$ is a necessary approximation; removing the inter-Gaussian context model entirely results in significantly larger file sizes, proving that even this weak context is valuable [S1, unofficial].
- **Convergence:** Increasing the number of splits beyond 4 yields negligible size reduction, suggesting diminishing returns in context enrichment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inter_gaussian_context_model(gaussians, num_splits=4, proportions=None):
    """
    Compress Gaussians using inter-Gaussian context.
    
    Args:
        gaussians: List of Gaussian objects with attributes (pos, color, etc.)
        num_splits: Number of batches to split Gaussians into.
        proportions: List of float proportions summing to 1.0.
                     Default: [0.17, 0.17, 0.33, 0.33]
    
    Returns:
        Compressed bitstream
    """
    if proportions is None:
        if num_splits == 4:
            proportions = [0.17, 0.17, 0.33, 0.33]
        elif num_splits == 2:
            proportions = [0.95, 0.05]
        else:
            # Fallback for other split counts, not specified in sources
            proportions = [1.0 / num_splits] * num_splits

    # Split Gaussians into batches
    batches = []
    start_idx = 0
    for prop in proportions:
        end_idx = start_idx + int(len(gaussians) * prop)
        batches.append(gaussians[start_idx:end_idx])
        start_idx = end_idx

    context_buffer = [] # Stores processed attributes from previous batches
    bitstream = []

    for i, batch in enumerate(batches):
        batch_bits = []
        for gaussian in batch:
            # Determine context
            if i == 0:
                # First batch: no inter-Gaussian context
                # Placeholder mean
                mu_g = 0.0
                # Positional encoding still provides context
                pos_enc = positional_encoding(gaussian.position)
            else:
                # Subsequent batches: use context from previous batches
                # Context aggregation logic (e.g., mean, variance, or learned embedding)
                # of all gaussians in context_buffer
                context = aggregate_context(context_buffer)
                mu_g = predict_mean(context, gaussian.position)
                pos_enc = positional_encoding(gaussian.position)

            # Predict distribution parameters
            # The context model (e.g., MLP) takes pos_enc and context
            # to predict mean and variance for the current attribute
            mean, variance = context_model(pos_enc, context if i > 0 else None)
            
            # Quantize and encode
            quantized_attr = quantize(gaussian.attribute, mean, variance)
            bits = encode(quantized_attr, mean, variance)
            batch_bits.append(bits)
            
            # Add to context buffer for future batches
            context_buffer.append(gaussian)

        bitstream.extend(batch_bits)
        
    return bitstream
```

## Subtopics

### Intra-Gaussian Context Model
Predicts attributes within a single Gaussian by splitting them into sequential chunks ($C_1 \to C_4$) rather than across Gaussians [S1, unofficial]. Context is derived from a learnable MLP operating in latent space [S1, unofficial]. Unlike the inter-Gaussian model, where batches have equal average information, intra-Gaussian chunks have unequal information distribution due to the learnable MLP latent space [S1, unofficial]. $C_1$ lacks intra-Gaussian context and receives minimal information allocation to save bits, while $C_2$ receives the most information as it plays a pivotal role in predicting subsequent chunks ($C_3, C_4$) [S1, unofficial]. The default number of chunks is 4 [S1, unofficial].

## Operating characteristics
- **Time and Memory Complexity:** Not explicitly given in the sources. The method involves splitting data and aggregating context, which adds overhead compared to independent coding.
- **Measured Performance:** On the DL3DV-GS dataset ($\lambda=1e-4$), 4 splits yield 27.440 MB with a PSNR of 29.264 dB [S1, unofficial].
- **Scaling:** Increasing splits from 3 to 4 reduces size by 0.227 MB. Beyond 4 splits, size reduction is negligible (e.g., 5 splits yield 27.379 MB) [S1, unofficial].
- **Strengths:** Significantly reduces file size compared to removing the inter-Gaussian context model entirely. The positional encoding $emb(\mu_g)$ provides effective fallback context even when $\mu_g$ is a placeholder [S1, unofficial].
- **Limitations:** The first batch ($B_1$) lacks inter-Gaussian context, leading to less accurate probability estimates and higher bits per parameter for $B_1$ compared to subsequent batches [S1, unofficial]. Extreme split strategies (e.g., 95%, 5%) perform poorly [S1, unofficial].

## Background
The Inter-Gaussian Context Model is part of the Fast Feedforward 3D Gaussian Splatting Compression (FCGS) method, published as a conference paper at ICLR 2025 [S1, unofficial]. It builds on the concept of Gaussian Probability Modeling by introducing structured context dependencies between Gaussians to improve compression efficiency [S1, unofficial].

## Sources
[S1] Fast Feedforward 3D Gaussian Splatting Compression, https://doi.org/10.48550/arxiv.2410.08017, fetched 2026-10-26 (unofficial)
