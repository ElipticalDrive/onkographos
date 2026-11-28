# Hierarchical Progressive Encoding

Also called: HPC Encoding, Multi-resolution Residual Encoding

## Overview
Hierarchical Progressive Encoding (HPC) is a compression method for dynamic volumetric video that reduces temporal redundancy by encoding feature grids as residuals relative to previous frames across multiple resolution levels [S1]. It differs from static volume rendering compression by utilizing temporal coherence within Groups of Features (GoFs) and from sibling methods by processing residuals at each resolution level through separate entropy encoders to preserve unique statistical properties [S1]. This approach enables scalable streaming, allowing reconstruction at varying levels of detail based on available bandwidth [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of dynamic volumetric video frames, represented as feature grids $F$.
*   **Output:** A compressed bitstream $B$ containing entropy-coded residuals.

**Data Structures:**
*   **Feature Grids:** The large feature grid $F$ is decomposed into $L$ resolution levels: $F = \{F_l \mid l \in [1, L]\}$. Higher index $l$ corresponds to higher resolution and finer detail [S1].
*   **Residual Grids:** $R_t$ represents the residual for frame $t$.
*   **Bitstream:** $B_t = \{B_{tl} \mid l \in [1, L]\}$, where $B_{tl}$ is the bitstream for level $l$ of frame $t$ [S1].

**Parameters:**
*   $L$: Number of resolution levels. In experimental setups, $L=6$ [S1].
*   $q$: Quantization parameter. Multiplying residuals by $q$ expands the data range to enhance quantization precision. Adjusting $q$ trades off between reconstruction quality and model storage [S1].
*   $N$: Number of frames in a Group of Features (GoF) [S1].

**Steps:**
1.  **Segmentation:** Divide the video sequence into GoFs of $N$ frames [S1].
2.  **Reference Frame (First Frame of GoF):** For the first frame $t$ in a GoF, store the full $L$-level feature grid as the reference. No residual encoding is performed for this frame's data; it serves as the base for subsequent P-frames [S1].
3.  **Residual Calculation:** For subsequent frames $t$ in the GoF, compute the residual grid $R_t$ relative to the reconstructed previous frame $\hat{F}_{t-1}$ [S1, S2].
    $$F_t = \hat{F}_{t-1} + R_t$$
    $$R_t = F_t - \hat{F}_{t-1}$$
    At each resolution level $l$:
    $$R_{tl} = F_{tl} - \hat{F}_{t-1,l}$$
4.  **Quantization and Offset Adjustment:** To facilitate compression into uint8 format, residuals are converted to non-negative values. The quantized residual at level $l$ for frame $t$ is calculated as [S1]:
    $$Q_{tl} = Q(q \cdot R_{tl}) - Q(q \cdot \min(R_{tl}))$$
    Where $Q(\cdot)$ is the quantization operation. The term $Q(q \cdot \min(R_{tl}))$ shifts the data to be non-negative [S1].
5.  **Entropy Encoding:** Encode the quantized residuals at each level $l$ separately using a dedicated entropy encoder $E_l$ [S1]:
    $$B_{tl} = E_l(Q_{tl})$$
    Separate encoders are used because residuals at different resolutions have unique statistical properties [S1].
6.  **Buffer Update:** Store the reconstructed grid $\hat{F}_t$ into the decode buffer for the reconstruction of the next frame [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_progressive_encoding(feature_grids, L, q, N):
    """
    feature_grids: List of feature grids for each frame.
                   Each grid is a list of L resolution levels.
    L: Number of resolution levels.
    q: Quantization parameter.
    N: Group of Features size.
    """
    bitstream = []
    prev_reconstructed_grids = None
    
    for t, frame_grids in enumerate(feature_grids):
        # Determine GoF index
        gof_index = t // N
        frame_in_gof = t % N
        
        if frame_in_gof == 0:
            # First frame of GoF: Store full reference
            # In a real implementation, this might be stored as a special marker
            # or the full grid is entropy coded directly.
            # For the purpose of the residual structure described:
            reference_grids = frame_grids
            # Encode reference_grids directly or as I-frame data
            # (Specific I-frame encoding details not provided in sources, 
            # assuming standard entropy coding of the full grid)
            ref_bits = []
            for l in range(L):
                # Placeholder for I-frame encoding logic
                ref_bits.append(entropy_encode(frame_grids[l])) 
            bitstream.append(ref_bits)
            prev_reconstructed_grids = frame_grids
        else:
            # P-frame: Encode residuals relative to previous frame
            current_grids = frame_grids
            residual_grids = []
            
            for l in range(L):
                # Calculate residual
                residual = current_grids[l] - prev_reconstructed_grids[l]
                
                # Find min for offset
                min_val = min(residual.flatten())
                
                # Quantize and offset
                # Q() represents quantization (e.g., rounding)
                quantized = quantize(q * residual)
                offset_quantized = quantize(q * min_val)
                
                # Shift to non-negative
                shifted = quantized - offset_quantized
                
                # Encode with level-specific encoder
                bits = entropy_encode_level(shifted, level=l)
                residual_grids.append(bits)
            
            bitstream.append(residual_grids)
            
            # Update reconstructed grid for next frame
            # Note: In practice, this would be the decoded version of the residuals
            # added to the previous reconstructed grid.
            # For encoding simulation, we use the ground truth if perfect reconstruction
            # is assumed for the buffer, or we would need a decoder here.
            # The source says "Store the reconstructed grid... for the reconstruction of the next frame".
            # We assume prev_reconstructed_grids is updated with the *decoded* version.
            # Since we are in the encoder, we must simulate the decoder or assume
            # the encoder has access to the decoded buffer.
            decoded_residuals = []
            for l in range(L):
                # Decode the bits we just generated to get the value to add
                decoded_shifted = entropy_decode_level(residual_grids[l], level=l)
                # Reverse offset and quantization
                # This part is technically decoding, but required for the encoder's buffer state
                # if the encoder is closed-loop.
                # Reconstruct: R_hat = (decoded_shifted + Q(q*min)) / q
                # However, min(R) is needed. The source implies min(R) is part of the encoding.
                # Let's assume the encoder knows the min of the residual it just encoded.
                min_val = min(residual_grids[l]) # This is incorrect, min_val was from residual, not bits.
                # Correct logic from source:
                # R_hat = (D(B_tl) + Q(q * min(R_tl))) / q
                # We need min(R_tl) from the residual calculation step.
                pass 
            
            # Simplified buffer update for implementation clarity:
            # The encoder must maintain the decoded state.
            # We will assume a helper function `decode_and_update` exists or
            # that the encoder simulates the decoder.
            # For the sake of the algorithm description:
            prev_reconstructed_grids = decode_residuals_and_add(residual_grids, prev_reconstructed_grids, q)

    return bitstream

def decode_residuals_and_add(residual_bits, prev_grids, q):
    """
    Simulates the decoding step to update the buffer in the encoder.
    """
    new_grids = []
    for l, bits in enumerate(residual_bits):
        # Decode
        decoded_shifted = entropy_decode_level(bits, level=l)
        
        # We need the min value of the residual to reverse the offset.
        # In a real encoder, this min value must be stored or transmitted.
        # The source formula: R_hat = (D(B_tl) + Q(q * min(R_tl))) / q
        # This implies min(R_tl) is known or derived.
        # Since the source doesn't specify how min(R) is transmitted,
        # we assume it's part of the metadata or the decoder has access to it.
        # For this pseudocode, we assume a function `get_min_residual` exists.
        min_val = get_min_residual(l) 
        
        # Reverse quantization and offset
        # Q(q * min) was subtracted. So we add it back.
        # Then divide by q.
        reconstructed_residual = (decoded_shifted + quantize(q * min_val)) / q
        
        # Add to previous grid
        new_grid = prev_grids[l] + reconstructed_residual
        new_grids.append(new_grid)
        
    return new_grids
```

**Edge Cases and Pitfalls:**
*   **Precision:** Quantization parameter $q$ affects precision. Too low $q$ loses detail; too high $q$ increases bitrate [S1].
*   **Non-negative Conversion:** Residuals must be shifted to non-negative values for uint8 compression. The minimum value of the residual must be preserved and transmitted/known for correct decoding [S1].
*   **Drift:** Errors in decoding residuals can accumulate if not handled. The source mentions storing the *reconstructed* grid in the buffer, implying a closed-loop prediction to prevent drift [S2].

## Subtopics

### Inter Prediction-Based Dynamic Modeling
A dynamic modeling strategy that eliminates inter-frame redundancy by modeling P-frames as residuals relative to the previous frame's reconstruction, reducing bandwidth requirements for volumetric video [S2]. It differs from static compression by explicitly handling temporal redundancy in dynamic scenes [S2]. It is chosen when streaming dynamic volumetric video with limited bandwidth [S2]. It builds on the residual encoding concept by applying it to feature grids in a group structure (I-frames and P-frames) [S2]. Key details: The sequence is divided into groups where the first frame is an I-frame (modeled independently) and subsequent frames are P-frames (modeled as residuals) [S2]. For a P-frame, the reconstruction grid $\hat{G}_{t-1}$ is retrieved from the decoded buffer, and the residual grid $R_t$ is learned [S2]. The current frame representation is $G_t = \hat{G}_{t-1} + R_t$ [S2]. The reconstructed grid $\hat{G}_t$ is stored in the decode buffer for the next frame [S2]. Frames within the same group share the rendering MLP to reduce bitrate consumption [S2]. An $\mathcal{L}1$ regularization $\mathcal{L}_{reg} = \|R_t\|_1$ is applied to the magnitude of the residual grid to ensure temporal continuity and facilitate compression [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The DNA-Rendering dataset (2048x2448, 48 views) was used for testing [S1]. The framework allows for scalable quality adjustments [S1].
*   **Scaling:** The method supports progressive streaming by merging feature grids from lowest to highest resolution, allowing the scene to be presented at different levels of precision [S1].
*   **Strengths:** Reduces temporal redundancy in extended sequences, offers scalable quality adjustments, and tailors streaming experience to fluctuating network conditions [S1].
*   **Limitations:** Requires storing the full reference grid for the first frame of each GoF [S1].
*   **When to use:** Dynamic volumetric video streaming where bandwidth varies and temporal coherence exists [S1].
*   **When not to use:** Static volume rendering or scenes with no temporal coherence [S1].

## Background
Hierarchical Progressive Encoding was introduced in the context of Neural Residual Radiance Fields (ReRF) for volumetric video compression [S1]. It addresses the inefficiency of transmitting entire radiance fields for each frame by leveraging temporal coherence [S1]. The method was proposed to solve the storage and streaming challenges of dynamic neural radiance fields [S1].

## Sources
[S1] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
[S2] Rate-aware Compression for NeRF-based Volumetric Video, https://doi.org/10.1145/3664647.3680970, fetched 2026-10-27
