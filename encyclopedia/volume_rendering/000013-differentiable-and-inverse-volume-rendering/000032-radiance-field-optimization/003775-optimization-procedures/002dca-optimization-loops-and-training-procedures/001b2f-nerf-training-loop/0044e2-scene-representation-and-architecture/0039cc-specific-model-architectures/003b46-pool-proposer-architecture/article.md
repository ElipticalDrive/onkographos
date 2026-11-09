# Pool proposer architecture

Also called: Pooling-based proposer

## Overview
The Pool proposer architecture is a trainable neural network component used in NeRF-ID to generate fine-grained sampling positions for volume rendering [S1, unofficial]. It processes coarse features from a preliminary sampling stage to propose new, higher-quality sample locations along a ray [S1, unofficial]. It is distinguished from its sibling [MLPMix Proposer](#mlpmix-proposer) by its use of average pooling and a bottleneck fully connected layer, though it exhibits higher variance and significantly worse performance on specific complex scenes like Blender: Materials and Blender: Mic [S1, unofficial].

## How it works
The Pool proposer refines the set of coarse sample positions $\mathbf{x}_c$ and their associated features $\mathbf{f}_c$ to produce a set of fine sample positions $\mathbf{x}_f$ [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of $N_c = 64$ coarse features $\mathbf{f}_c \in \mathbb{R}^{N_c \times D}$ and their corresponding 3D positions $\mathbf{x}_c \in \mathbb{R}^{N_c \times 3}$ along a ray [S1, unofficial]. The feature dimension $D$ is not explicitly defined in the sources, but the bottleneck layer size is fixed [S1, unofficial].
*   **Output:** A set of $N_f = 128$ proposed fine positions $\mathbf{x}_f \in \mathbb{R}^{N_f \times 3}$ [S1, unofficial].

**Data Structures:**
The architecture assumes a fixed number of coarse samples $N_c = 64$ and fine proposals $N_f = 128$ [S1, unofficial]. The core computational bottleneck is a single fully connected (FC) layer of size $256 \times 256$ [S1, unofficial].

**Steps:**
1.  **Feature Processing:** For each of the $N_c$ coarse features, apply a sinusoidal positional encoding $\gamma(\mathbf{x}_c)$ to the input position [S1, unofficial].
2.  **Summation:** Sum the sinusoidal positional encodings with the coarse features $\mathbf{f}_c$ [S1, unofficial].
3.  **Bottleneck Projection:** Pass the summed features through a single fully connected layer of size $256 \times 256$ [S1, unofficial]. This is the most expensive operation for this proposer [S1, unofficial].
4.  **Position Proposal:** The output of the FC layer is used to generate the $N_f = 128$ fine positions $\mathbf{x}_f$ [S1, unofficial]. The exact mapping from the FC output to the 3D coordinates is not detailed in the sources, but it operates on the processed features [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pool_proposer(coarse_features, coarse_positions):
    """
    Generates fine sample positions using the Pool proposer architecture.
    
    Args:
        coarse_features: Tensor of shape (Nc, D) containing coarse features.
        coarse_positions: Tensor of shape (Nc, 3) containing 3D positions.
        
    Returns:
        fine_positions: Tensor of shape (Nf, 3) containing proposed fine positions.
    """
    Nc = 64
    Nf = 128
    # Note: The exact dimension D and the final mapping to 3D coordinates
    # are not specified in the sources. The following assumes a standard
    # implementation where the FC layer output is reshaped or projected.
    
    # 1. Apply sinusoidal positional encoding to coarse positions
    # gamma(x) includes sin/cos at increasing frequencies
    encoded_positions = sinusoidal_encoding(coarse_positions)
    
    # 2. Sum encodings with features
    # Assumes encoded_positions shape matches coarse_features shape for addition
    # If dimensions differ, a projection step would be required before summation.
    summed_features = coarse_features + encoded_positions
    
    # 3. Apply bottleneck fully connected layer (256x256)
    # The source states this is the most expensive operation [S1, unofficial]
    bottleneck_output = linear_layer(summed_features, weight_matrix_256x256)
    
    # 4. Generate fine positions
    # The source does not specify how bottleneck_output maps to Nf=128 3D points.
    # It implies the proposer outputs positions directly.
    # [G] Assuming a final linear projection to 3D coordinates and expansion to Nf
    # This step is inferred as the source lacks the exact formula for position generation.
    fine_positions = generate_positions(bottleneck_output, Nf)
    
    return fine_positions
```

**Parameters:**
*   $N_c = 64$: Number of coarse samples [S1, unofficial].
*   $N_f = 128$: Number of fine proposals [S1, unofficial].
*   Bottleneck size: $256 \times 256$ fully connected layer [S1, unofficial].

**Edge Cases:**
*   **Blind Proposer:** A variant that ignores input coarse features and outputs constant proposals performs poorly, indicating that feature processing is necessary [S1, unofficial].
*   **No-position Proposer:** A variant that ignores coarse positions performs poorly on Blender scenes (sensitive to feature ordering) but is competitive on LLFF-NeRF (forward-facing scenes) [S1, unofficial].

## Subtopics

### MLPMix Proposer
The MLPMix proposer is an alternative trainable proposer architecture that uses a multi-layer perceptron mixing mechanism instead of pooling [S1, unofficial]. It is preferred over the Transformer proposer due to the Transformer's inferior performance on scenes like Blender: Ficus, Blender: Lego, and LLFF-NeRF: T-Rex, as well as unstable training curves [S1, unofficial]. MLPMix and Pool are generally on par in performance, but Pool has higher variance [S1, unofficial].

## Operating characteristics
*   **Complexity:** The most expensive operation is the $256 \times 256$ fully connected layer applied to each of the $N_c = 64$ coarse features [S1, unofficial]. However, this cost is considered negligible compared to the NeRF backbone, which applies similar FC layers to $8 \times (N_c + N_c + N_f) = 2048$ vectors [S1, unofficial].
*   **Performance:** Pool and MLPMix proposers tend to be the best performing architectures [S1, unofficial]. However, the Pool proposer has higher variance and is significantly worse on specific scenes: Blender: Materials and Blender: Mic [S1, unofficial].
*   **Comparison:** The Transformer proposer is inferior to both Pool and MLPMix due to worse performance on specific scenes and unstable training [S1, unofficial].

## Background
The Pool proposer architecture was introduced in the context of NeRF-ID (NeRF in detail), a study on learning to sample for view synthesis [S1, unofficial]. It is part of a family of trainable proposer architectures designed to improve upon the fixed sampling strategies of the original NeRF [S1, unofficial]. The architecture was evaluated against Transformer and MLPMix proposers using PSNR and SSIM metrics on various datasets including Blender and LLFF-NeRF [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
