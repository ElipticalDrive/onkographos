# Hierarchical Progressive Encoding

Also called: HPC Encoding, Multi-resolution Residual Encoding

## Overview
Hierarchical Progressive Encoding (HPC) is a method for compressing dynamic volumetric video by exploiting temporal coherence and multi-resolution spatial redundancy. It segments video sequences into Groups of Features (GoFs) and represents each frame's feature grid as a residual relative to the previous frame's grid, decomposed across multiple resolution levels [S1]. Unlike methods that transmit full radiance fields for every frame, HPC allows for adaptive rendering by selecting a subset of resolution levels based on bandwidth or computational constraints, enabling scalable quality adjustments under fluctuating network conditions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of dynamic volumetric video frames, represented as feature grids $F$.
*   **Output:** A compressed bitstream $B$ containing entropy-encoded residuals.

**Data Structures:**
*   **Feature Grid ($F$):** Decomposed into $L$ resolution levels, $F = \{F_l \mid l \in [1, L]\}$. As the index $l$ increases, the resolution of $F_l$ increases, capturing finer details [S1].
*   **Group of Features (GoF):** A segment of $N$ frames. The first frame in a GoF is stored as a full $L$-level reference. Subsequent frames are stored as residuals [S1].
*   **Bitstream ($B_t$):** For frame $t$, the bitstream is a collection of level-specific bitstreams: $B_t = \{B_{tl} \mid l \in [1, L]\}$ [S1].

**Parameters:**
*   $L$: Number of resolution levels. In the source experiments, $L=6$ [S1].
*   $q$: Quantization parameter. Multiplying residuals by $q$ expands the data range to enhance quantization precision. Adjusting $q$ trades off between reconstruction quality and model storage [S1].
*   $N$: Number of frames per GoF. In the source experiments, $N=20$ [S1].

**Steps:**
1.  **Decomposition:** Decompose the large feature grid $F$ into $L$ different resolutions: $F = \{F_l \mid l \in [1, L]\}$ [S1].
2.  **Reference Storage:** For the first frame in a GoF, store the full $L$-level feature grid as the reference [S1].
3.  **Residual Calculation:** For subsequent frames $t$ in the GoF, calculate the residual $R_t$ relative to the previous frame's reconstructed feature grid $F_{t-1}$:
    $$ F_t = F_{t-1} + R_t $$
    This implies $R_t = F_t - F_{t-1}$ [S1].
4.  **Quantization and Offset:** To facilitate compression into uint8 format, residuals are converted to non-negative values. The quantized residual at level $l$ for frame $t$ is computed as:
    $$ Q(q \cdot R_{tl}) - Q(q \cdot \min(R_l)) $$
    where $Q$ represents the quantization operation, and $\min(R_l)$ is the minimum value of the residual at level $l$ [S1].
5.  **Entropy Encoding:** Encode the quantized residuals at each level $l$ separately using a dedicated entropy encoder $E_l$. This layered approach accounts for the unique statistical properties of residuals at each resolution [S1]:
    $$ B_{tl} = E_l \left( Q(q \cdot R_{tl}) - Q(q \cdot \min(R_l)) \right) $$
    The total bitstream for frame $t$ is $B_t = \{B_{tl} \mid l \in [1, L]\}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_progressive_encode(feature_grids, L, q, N):
    """
    Encodes a sequence of feature grids using Hierarchical Progressive Encoding.
    
    Args:
        feature_grids: List of feature grids for N frames. Each grid is a list of L levels.
        L: Number of resolution levels.
        q: Quantization parameter.
        N: Number of frames in the Group of Features (GoF).
        
    Returns:
        bitstream: Dictionary mapping frame index to a list of L bitstreams.
    """
    bitstream = {}
    previous_reconstructed_grids = None
    
    for t in range(N):
        current_grids = feature_grids[t]
        
        if t == 0:
            # First frame: store full reference
            # In practice, this might be encoded as a special header or full residual
            # For this implementation, we treat it as residual from zero or store directly
            # The source states: "store the full L-level feature grid as the reference"
            # We assume the reference is stored separately or as the first bitstream entry
            reference_grids = current_grids
            bitstream[t] = [encode_level(level, q) for level in current_grids]
        else:
            # Subsequent frames: encode residuals
            residuals = []
            for l in range(L):
                # Calculate residual: R_t = F_t - F_{t-1}
                # Note: F_{t-1} is the reconstructed grid from previous step
                residual = current_grids[l] - previous_reconstructed_grids[l]
                residuals.append(residual)
            
            # Quantize and offset for non-negative uint8
            encoded_levels = []
            for l in range(L):
                residual = residuals[l]
                min_val = min(residual.flatten())
                
                # Quantize: Q(q * R)
                # Offset: subtract Q(q * min(R))
                quantized = quantize(q * residual)
                offset = quantize(q * min_val)
                
                # Convert to non-negative for entropy coding
                non_negative_quantized = quantized - offset
                
                # Encode with dedicated entropy encoder E_l
                bitstream_entry = entropy_encode(non_negative_quantized, encoder_id=l)
                encoded_levels.append(bitstream_entry)
            
            bitstream[t] = encoded_levels
            
            # Update previous reconstructed grids for next iteration
            # F_t = F_{t-1} + R_t
            # Since we encoded R_t, we must reconstruct F_t for the next frame's residual calc
            previous_reconstructed_grids = []
            for l in range(L):
                # Reconstruct residual from bitstream (for internal consistency in encoding loop)
                # In a real system, this would be done during decoding, but for encoding
                # we keep the ground truth F_t to compute next residual.
                # However, the source implies recursive reconstruction:
                # F_t = {F_{l,t-1} + R_{l,t} | l in [1,L]}
                # So we use the original current_grids as the "reconstructed" version
                # because the encoder has access to the source data.
                previous_reconstructed_grids.append(current_grids[l])
                
    return bitstream

def quantize(data, q):
    """Quantize data by multiplying by q and rounding."""
    return np.round(data * q)

def entropy_encode(data, encoder_id):
    """Encode data using the specific entropy encoder for the level."""
    # Implementation depends on the specific entropy coder (e.g., range encoder)
    # [S1] mentions range encoder [42]
    return range_encoder(data)
```

## Subtopics

### Hierarchical Progressive Decoding
A decoding process that recovers feature grids from a bitstream and allows adaptive rendering by selecting a subset of resolution levels based on bandwidth or computational constraints [S1].
*   **Definition:** Recovers quantized residuals from the bitstream $B_t$ using an entropy decoder $D$ [S1].
*   **Steps:**
    1.  Decode the bitstream $B_t$ using entropy decoder $D$ to recover quantized residuals [S1].
    2.  Reconstruct the feature grid $\hat{F}_t$ at level $l$ by adding the recovered residual $\hat{R}_{tl}$ to the previous frame's reconstructed feature grid $\hat{F}_{l,t-1}$:
        $$ \hat{R}_t = \frac{D(B_t) + Q(q \cdot \min(R_t))}{q} $$
        $$ \hat{F}_t = \hat{F}_{t-1} + \hat{R}_t $$
        Note: The source formula (6) shows the recovery of the residual $\hat{R}_t$ from the decoded bitstream and the stored minimum value [S1].
    3.  Render the scene by passing the reconstructed feature grids $\hat{F}_l$ through a global MLP $\Phi$ to obtain color $c$ and density $\sigma$, then perform volume rendering [S1].
*   **Adaptive Rendering:** By selecting a smaller level $l$, the decoder can choose to only receive and decode lower-resolution data, tailoring the effect to the device's computational capacity [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. The encoding involves per-level entropy encoding, which is generally linear in the number of voxels/features per level.
*   **Memory Complexity:** Not explicitly given. The method requires storing the previous frame's reconstructed grid for residual calculation.
*   **Measured Performance:** The source does not provide specific compression ratios or bitrate numbers for HPC, but compares it to other methods in Section 5.2 [S1].
*   **Scaling:** The method scales with the number of resolution levels $L$ and the GoF size $N$. Increasing $L$ improves detail but increases bitrate. Increasing $N$ increases temporal redundancy exploitation but may reduce prediction accuracy if motion is large.
*   **Strengths:** Reduces temporal redundancy by using frame-to-frame residuals. Allows progressive streaming and adaptive quality based on bandwidth. Single model outputs features at different bitrates [S1].
*   **Limitations:** Requires recursive reconstruction, meaning errors can propagate if decoding is not error-free. The source does not discuss error resilience.
*   **When to use:** Volumetric video delivery under fluctuating network conditions or when device computational capacity is limited [S1].
*   **When not to use:** Real-time applications requiring low latency where recursive decoding overhead is prohibitive, or scenes with high temporal discontinuity where residual prediction is ineffective.

## Background
*   **Origin:** HPC was introduced in the paper "HPC: Hierarchical Progressive Coding Framework for Volumetric Video" presented at MM’24 (October 28 - November 1, 2024, Melbourne, Australia) [S1].
*   **History:** It is described as the first framework for hierarchical progressive coding for volumetric video [S1].
*   **Usage:** Designed for dynamic volumetric video compression, addressing the inefficiency of transmitting entire radiance fields for each frame [S1].

## Sources
[S1] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
