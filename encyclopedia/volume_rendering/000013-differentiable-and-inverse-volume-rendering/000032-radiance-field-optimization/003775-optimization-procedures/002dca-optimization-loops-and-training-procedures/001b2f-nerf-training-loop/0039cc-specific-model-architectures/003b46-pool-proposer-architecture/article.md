# Pool proposer architecture

Also called: Pooling-based proposer

## Overview
The Pool proposer is a trainable network component in NeRF-ID that generates fine-grained sampling positions from coarse features along a ray [S1, unofficial]. It solves the problem of adaptive ray sampling by learning to propose additional points where the radiance field likely contains high-frequency details, improving reconstruction quality over uniform or heuristic sampling [S1, unofficial]. It is distinguished from sibling architectures like MLPMix by its use of a bottleneck fully connected layer and sinusoidal positional encoding summation, whereas MLPMix uses a different mixing strategy [S1, unofficial]. Compared to the Transformer proposer, the Pool proposer offers more stable training and better performance on complex scenes, though it exhibits higher variance than MLPMix [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A set of $N_c = 64$ coarse features extracted from the coarse sampling stage [S1, unofficial].
*   Output: A set of $N_f = 128$ proposed fine sampling positions (and associated features) [S1, unofficial].

**Data Structures:**
*   Coarse features: Vector of size $N_c \times D$, where $D$ is the feature dimension (implied by the bottleneck layer input size).
*   Bottleneck Layer: A single fully connected (FC) layer with dimensions $256 \times 256$ [S1, unofficial].
*   Positional Encoding: Sinusoidal encodings of the coarse sample positions [S1, unofficial].

**Steps:**
1.  **Feature Processing:** The $N_c$ coarse features are processed. The most expensive operations occur before average pooling, applied to each of the $N_c$ features [S1, unofficial].
2.  **Bottleneck Application:** A single fully connected layer of size $256 \times 256$ is applied to the processed features [S1, unofficial].
3.  **Positional Encoding Summation:** Sinusoidal positional encodings of the coarse sample positions are computed and summed with the features [S1, unofficial].
4.  **Proposal Generation:** The network outputs the $N_f = 128$ fine proposals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pool_proposer(coarse_features, coarse_positions):
    """
    Generates fine sampling proposals from coarse features.
    
    Args:
        coarse_features: Tensor of shape (Nc, D), where Nc=64.
        coarse_positions: Tensor of shape (Nc, 3), 3D coordinates of coarse samples.
        
    Returns:
        fine_proposals: Tensor of shape (Nf, 3), proposed fine positions.
        fine_features: Tensor of shape (Nf, D), associated features.
    """
    Nc = 64
    Nf = 128
    
    # Step 1: Process coarse features (operations before pooling)
    # Note: The exact pre-pooling operations are not detailed in the source,
    # but they are applied to each of the Nc features.
    processed_features = process_coarse_features(coarse_features)
    
    # Step 2: Apply bottleneck fully connected layer (256x256)
    # Assuming processed_features has dimension 256
    bottleneck_output = apply_fc_layer(processed_features, weight_256x256, bias_256)
    
    # Step 3: Compute sinusoidal positional encodings for coarse positions
    pos_encodings = sinusoidal_encoding(coarse_positions)
    
    # Step 4: Sum positional encodings with features
    # This step combines spatial information with feature information
    combined_features = bottleneck_output + pos_encodings
    
    # Step 5: Generate proposals
    # The source does not specify the exact mechanism for generating Nf=128
    # outputs from the combined features. It implies a transformation occurs here.
    # [G] Assuming a final linear projection or similar mechanism to map to Nf outputs.
    fine_proposals, fine_features = generate_proposals(combined_features, Nf)
    
    return fine_proposals, fine_features

def sinusoidal_encoding(positions):
    """
    Computes standard NeRF-style sinusoidal positional encodings.
    """
    # Standard implementation: scale, sin/cos, concatenate
    # This is a standard operation in NeRF literature, not specific to the proposer
    # but cited as part of the proposer's input processing.
    pass

def process_coarse_features(features):
    """
    Placeholder for operations applied before average pooling.
    The source states these are the most expensive operations.
    """
    pass

def apply_fc_layer(input_tensor, weight, bias):
    """
    Applies a fully connected layer.
    """
    return input_tensor @ weight + bias

def generate_proposals(features, Nf):
    """
    Generates Nf proposals from the processed features.
    The exact method is not specified in the source.
    """
    pass
```

**Parameters:**
*   $N_c = 64$: Number of coarse samples [S1, unofficial].
*   $N_f = 128$: Number of fine proposals [S1, unofficial].
*   Bottleneck size: $256 \times 256$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Blind Proposer:** Ignoring input coarse features leads to poor performance, showing that a degenerate strategy of always outputting the same proposals is insufficient [S1, unofficial].
*   **No-position Proposer:** Ignoring the positions of coarse samples along the ray performs badly on Blender scenes because it is insensitive to feature ordering (e.g., producing the same proposals for two colinear rays of opposite directionality). However, it remains competitive on LLFF-NeRF scenes, which are forward-facing [S1, unofficial].

## Subtopics

### MLPMix Proposer
MLPMix is a variant proposer architecture that is preferred over the Transformer proposer due to the latter's inferior performance on specific scenes (Blender: Ficus, Blender: Lego, LLFF-NeRF: T-Rex) and unstable training curves [S1, unofficial]. MLPMix and Pool are somewhat on par in general performance, but Pool has higher variance and is significantly worse on Blender: Materials and Blender: Mic scenes [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The most expensive operations in Pool and MLPMix are those applied to each of the $N_c = 64$ coarse features before average pooling [S1, unofficial].
*   The bottleneck is the single fully connected layer of size $256 \times 256$ [S1, unofficial].
*   The additional application of this FC layer on $N_c = 64$ vectors is negligible compared to the NeRF backbone, which applies 8 such layers on a total of $8 \times (N_c + N_c + N_f) = 2048$ vectors [S1, unofficial].
*   The Transformer proposer is somewhat more expensive due to the lack of pooling operations, requiring heavy constraints to keep costs reasonable [S1, unofficial].

**Measured Performance:**
*   Pool and MLPMix proposers tend to be the best performing, but Pool has higher variance [S1, unofficial].
*   Pool is significantly worse on Blender: Materials and Blender: Mic scenes compared to MLPMix [S1, unofficial].
*   The Transformer proposer is inferior due to significantly worse performance on Blender: Ficus, Blender: Lego, and LLFF-NeRF: T-Rex, along with unstable training curves [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Stable training compared to Transformer; negligible overhead compared to the main NeRF backbone [S1, unofficial].
*   **Limitations:** Higher variance than MLPMix; poor performance on specific complex scenes (Blender: Materials, Blender: Mic) [S1, unofficial].

**When to Use:**
*   Use when stability is preferred over the slight edge MLPMix might offer on average, and when the specific scenes do not include Blender: Materials or Blender: Mic.
*   Avoid if the Transformer proposer's instability is a concern and MLPMix is not available.

## Background
The Pool proposer architecture was introduced in "NeRF in detail: Learning to sample for view synthesis" by Mildenhall et al. (2021) [S1, unofficial]. It is part of the NeRF-ID framework, which builds on the original NeRF [37] and aims to improve sampling strategies for view synthesis. The architecture was designed to be computationally efficient, with the bottleneck layer being negligible compared to the main NeRF backbone [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
