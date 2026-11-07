# Pool proposer architecture

Also called: Pooling-based proposer

## Overview
The Pool proposer is a trainable network component used in NeRF-ID to generate fine sampling positions from coarse features [S1, unofficial]. It solves the problem of adaptive ray sampling by learning to propose new points based on the density and color features of existing coarse samples [S1, unofficial]. It is distinguished from the MLPMix proposer by its use of average pooling operations, which results in higher variance and significantly worse performance on specific scenes like Blender: Materials and Blender: Mic, though it remains competitive on average [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: $N_c = 64$ coarse features, each associated with a 3D position along the ray [S1, unofficial].
*   Output: $N_f = 128$ new sampling positions (proposals) [S1, unofficial].

**Data Structures:**
*   Coarse features: A tensor of shape $(N_c, D)$, where $D$ is the feature dimension (typically 256 in the NeRF backbone context) [S1, unofficial].
*   Bottleneck Layer: A single fully connected (FC) layer with dimensions $256 \times 256$ [S1, unofficial].

**Steps:**
1.  **Positional Encoding:** Compute sinusoidal positional encodings for the $N_c$ coarse 3D positions [S1, unofficial].
2.  **Feature Summation:** Sum the sinusoidal positional encodings with the coarse features [S1, unofficial].
3.  **Bottleneck Projection:** Pass the summed features through the single bottleneck fully connected layer ($256 \times 256$) [S1, unofficial].
4.  **Pooling:** Apply average pooling operations on the processed features [S1, unofficial].
5.  **Proposal Generation:** The output of the pooling operations yields the $N_f = 128$ proposed positions [S1, unofficial].

**Parameters:**
*   $N_c = 64$: Number of coarse samples [S1, unofficial].
*   $N_f = 128$: Number of proposed fine samples [S1, unofficial].
*   Bottleneck size: $256 \times 256$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The Pool proposer exhibits higher variance compared to MLPMix [S1, unofficial].
*   It performs significantly worse on scenes with complex materials or small details (e.g., Blender: Materials, Blender: Mic) [S1, unofficial].
*   Ignoring input coarse features (Blind proposer) or ignoring positions (No-position proposer) leads to poor performance, indicating the necessity of both feature and spatial information [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pool_proposer(coarse_features, coarse_positions):
    """
    Generates fine sampling proposals from coarse features and positions.
    
    Args:
        coarse_features: Tensor of shape (Nc, D), where Nc=64, D=256.
        coarse_positions: Tensor of shape (Nc, 3), 3D coordinates of coarse samples.
        
    Returns:
        proposals: Tensor of shape (Nf, 3), where Nf=128.
    """
    Nc = 64
    Nf = 128
    D = 256
    
    # Step 1: Sinusoidal positional encoding
    # Assuming a standard sinusoidal PE function that returns a vector of size D
    pos_encodings = sinusoidal_positional_encoding(coarse_positions, D)
    
    # Step 2: Sum encodings with features
    summed_features = coarse_features + pos_encodings
    
    # Step 3: Bottleneck FC layer (256x256)
    # W is a matrix of shape (D, D)
    bottleneck_output = linear(summed_features, W)
    
    # Step 4: Average pooling
    # The source mentions "operations before the average pooling" and "pooling operations".
    # In the context of NeRF-ID, this typically involves pooling over the Nc dimension
    # to aggregate information, potentially followed by expansion or further processing
    # to generate Nf positions. The exact mapping from pooled features to Nf positions
    # is not detailed beyond "pooling operations" in the provided text.
    # We assume a standard average pooling over the batch/feature dimension.
    pooled_features = average_pool(bottleneck_output)
    
    # Step 5: Generate proposals
    # The source states the output is Nf=128 proposals.
    # The exact transformation from pooled features to 3D positions is not specified
    # in the provided notes. [G] Assuming a final linear layer or similar mechanism
    # to map the pooled representation to 3D coordinates is required to complete
    # the implementation, but this detail is not in the sources.
    proposals = generate_proposals(pooled_features, Nf)
    
    return proposals
```

## Subtopics

### MLPMix Proposer
The MLPMix proposer is an alternative to the Pool proposer that uses MLP-based mixing instead of pooling [S1, unofficial]. It is preferred over the Transformer proposer because the Transformer shows significantly worse performance on scenes like Blender: Ficus, Blender: Lego, and LLFF-NeRF: T-Rex, along with unstable training curves [S1, unofficial]. MLPMix and Pool are somewhat on par in general performance, but MLPMix is more stable [S1, unofficial].

### Transformer Proposer
The Transformer proposer uses self-attention mechanisms to generate proposals [S1, unofficial]. It is constrained heavily to manage computational cost due to the lack of pooling operations [S1, unofficial]. It is inferior to Pool and MLPMix due to significantly worse performance on specific scenes and unstable training curves [S1, unofficial].

## Operating characteristics
*   **Complexity:** The most expensive operations are those applied to each of the $N_c = 64$ coarse features before average pooling [S1, unofficial]. The bottleneck is the single $256 \times 256$ FC layer [S1, unofficial].
*   **Performance:** Pool and MLPMix tend to be the best performing proposers, but Pool has higher variance [S1, unofficial]. Pool is significantly worse on Blender: Materials and Blender: Mic [S1, unofficial].
*   **Comparison:** The Transformer proposer is more expensive due to the lack of pooling operations [S1, unofficial].

## Background
The Pool proposer is part of the NeRF-ID method, which extends the original NeRF architecture by learning to sample [S1, unofficial]. The architecture of the coarse and fine networks is identical to the original NeRF [S1, unofficial]. The proposer architectures were introduced to improve view synthesis by adaptively sampling rays [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
