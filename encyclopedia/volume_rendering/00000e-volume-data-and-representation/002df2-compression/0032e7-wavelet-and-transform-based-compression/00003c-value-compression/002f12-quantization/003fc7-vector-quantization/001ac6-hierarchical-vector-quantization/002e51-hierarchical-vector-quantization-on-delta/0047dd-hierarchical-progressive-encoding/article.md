# Hierarchical Progressive Encoding

Also called: HPC Encoding, Multi-resolution Residual Encoding

## Overview
Hierarchical Progressive Encoding (HPC) is a compression method for dynamic volumetric video that exploits temporal coherence by encoding feature grids as residuals relative to the previous frame across multiple spatial resolutions [S1]. It differs from static volume compression by segmenting sequences into Groups of Features (GoFs) and using forward-referenced residuals, enabling adaptive streaming where lower-resolution details are transmitted first to satisfy varying bandwidth conditions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of dynamic scene feature grids. The sequence is segmented into GoFs of $N$ frames. Each frame $t$ has a feature grid $F_t$ decomposed into $L$ resolution levels $F_{tl}$ for $l \in [1, L]$.
*   **Output:** A compressed bitstream $B_t$ consisting of level-specific bitstreams $B_{tl}$.

**Data Structures:**
*   **Feature Grid ($F$):** A multi-resolution representation where $F = \{F_l \mid l \in [1, L]\}$. Higher index $l$ corresponds to higher resolution and finer detail [S1].
*   **Residual Grid ($R$):** The difference between the current frame's features and the previous frame's features at the same resolution level.
*   **Bitstream ($B$):** A collection of encoded residuals, $B_t = \{B_{tl} \mid l \in [1, L]\}$ [S1].

**Parameters:**
*   $L$: Number of resolution levels. Typically set to 6 in experimental setups [S1].
*   $q$: Quantization parameter. Used to scale residuals to enhance precision and facilitate uint8 conversion [S1].
*   $N$: Number of frames per GoF. Typically set to 20 in experimental setups [S1].

**Steps:**
1.  **Decomposition:** Decompose the full feature grid $F_t$ of the current frame into $L$ resolution levels: $F_t = \{F_{tl} \mid l \in [1, L]\}$ [S1].
2.  **Residual Calculation:** For the first frame in a GoF, store the full $L$-level feature grid as the reference. For subsequent frames $t > 1$ in the GoF, calculate the residual $R_{tl}$ at each level $l$ by subtracting the previous frame's features from the current frame's features:
    $$ R_{tl} = F_{tl} - F_{t-1,l} $$
    This establishes frame-to-frame residuals based on forward references [S1].
3.  **Quantization and Offset:** To facilitate compression into a uint8 format, the residuals must be non-negative. Calculate the minimum value of the scaled residuals for level $l$ at frame $t$:
    $$ \text{min\_val}_{tl} = \min(q \cdot R_{tl}) $$
    Shift the scaled residuals to be non-negative by subtracting this minimum:
    $$ \text{shifted}_{tl} = q \cdot R_{tl} - \text{min\_val}_{tl} $$
    The parameter $q$ expands the data range to enhance quantization precision [S1].
4.  **Entropy Encoding:** Encode the shifted, quantized residuals using a dedicated entropy encoder $E_l$ specific to resolution level $l$. Separate encoders are used for each level to account for unique statistical properties:
    $$ B_{tl} = E_l(\text{shifted}_{tl}) $$
    The total bitstream for frame $t$ is the union of all level bitstreams: $B_t = \{B_{tl} \mid l \in [1, L]\}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_progressive_encode(feature_grids, L, q, N):
    """
    Encodes a sequence of feature grids using Hierarchical Progressive Encoding.
    
    Args:
        feature_grids: List of feature grids, where each grid is a list of L resolution levels.
                       feature_grids[t][l] is the feature grid at time t, level l.
        L: Number of resolution levels.
        q: Quantization parameter.
        N: Number of frames per Group of Features (GoF).
        
    Returns:
        bitstreams: List of bitstreams for each frame. 
                    bitstreams[t] is a list of L bitstreams, one per resolution level.
    """
    bitstreams = []
    prev_frame_features = None
    
    for t, frame_features in enumerate(feature_grids):
        # Start of a new GoF
        if t % N == 0:
            # Store full reference for the first frame of the GoF
            # In a real implementation, this might be handled by a specific header or flag.
            # For the residual encoding logic, we treat the first frame as having zero residuals
            # relative to itself, or simply store the full grid. 
            # The source states: "For the first frame in a GoF, store the full L-level feature grid as the reference."
            # And "For frame t, Ft is reconstructed by adding together the residuals and the features from the previous frame".
            # This implies the first frame's data is the base.
            # We will encode zero residuals for the first frame, or handle it as a special case.
            # However, the formula R_t = F_t - F_{t-1} requires F_{t-1}.
            # Let's assume the first frame of the GoF is encoded as a full reference, 
            # and subsequent frames use residuals.
            
            # If it's the very first frame of the entire sequence or GoF, we might need to store the full grid.
            # The source says: "Within these groups, take the first group for example G1 = {F1, R2, R3 ... RN}"
            # This implies F1 is stored fully.
            pass 

        current_bitstream_for_frame = []
        
        # If it's the first frame of a GoF, we store the full features.
        # The source implies R1 is not needed, F1 is the reference.
        # For simplicity in this generic implementation, if t % N == 0, we might encode F_t directly 
        # or assume R_t = 0 and F_{t-1} is known. 
        # Let's follow the residual logic for t > 0 within a GoF.
        
        if t % N == 0:
            # First frame of GoF: Store full features.
            # This is a special case. The bitstream would contain the raw features or a special marker.
            # For the purpose of the residual formula, we can consider this as R_t = F_t and F_{t-1} = 0,
            # but the source says "F1" is stored.
            # We will encode the features directly or as a special "keyframe" block.
            # Since the prompt asks for the method from the text, and the text focuses on residuals for t>1:
            # We will assume the first frame is handled as a full reference.
            # Let's encode it as a special case or just store F_t.
            # To keep the pseudocode consistent with the residual formula for subsequent frames:
            # We can set prev_frame_features to None or a zero grid.
            
            # Actually, the source says: "G1 = {F1, R2, R3 ... RN}".
            # So for t=0 (first frame), we encode F1.
            # For t>0, we encode R_t.
            
            # Encoding F1:
            # The source doesn't specify the exact encoding for F1 other than "store the full L-level feature grid".
            # We will assume it is quantized and entropy encoded similarly, or stored raw.
            # Given the focus on residuals, we'll assume F1 is encoded using the same pipeline 
            # but with R_t = F_t (assuming F_{t-1} = 0).
            
            # Let's implement the residual encoding for t > 0, and handle t=0 separately.
            pass

        # Calculate residuals for levels 1 to L
        residuals = []
        for l in range(L):
            if t % N == 0:
                # First frame of GoF: Residual is the feature itself (assuming previous is 0)
                # Or simply, we store F_t. Let's assume R_t = F_t for consistency with the formula R_t = F_t - F_{t-1} if F_{t-1}=0.
                current_residual = frame_features[l]
            else:
                # Subsequent frames: Residual is difference from previous frame
                current_residual = frame_features[l] - prev_frame_features[l]
            
            residuals.append(current_residual)
        
        # Encode each level
        for l in range(L):
            R_tl = residuals[l]
            
            # Quantization and Offset
            # Multiply by q
            scaled_residuals = q * R_tl
            
            # Find minimum to make non-negative
            min_val = min(scaled_residuals.flatten()) # Assuming flatten for min operation
            
            # Shift to non-negative
            shifted_residuals = scaled_residuals - min_val
            
            # Convert to uint8 (implied by source "facilitate compression into the uint8 format")
            # We assume the values are within a range that fits uint8 after shifting, 
            # or we clamp them. The source doesn't specify clamping, just "converted into non-negative values".
            # We will assume the encoder handles the data type.
            
            # Entropy Encoding
            # E_l is the l-th entropy encoder
            bitstream_l = entropy_encode(shifted_residuals, encoder=E_l)
            
            current_bitstream_for_frame.append(bitstream_l)
            
        bitstreams.append(current_bitstream_for_frame)
        prev_frame_features = frame_features
        
    return bitstreams

def entropy_encode(data, encoder):
    # Placeholder for the specific entropy encoder implementation (e.g., range encoder)
    # The source mentions "range encoder[42]"
    return encode_with_range_encoder(data)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The choice of $q$ is critical. A larger $q$ improves quantization precision but increases the dynamic range, potentially requiring more bits or causing overflow if not handled correctly [S1].
*   **Boundaries:** The minimum value calculation $\min(q \cdot R_{tl})$ must be transmitted or derived at the decoder to reverse the shift. The source shows the decoder uses $Q(q \cdot \min(R_{tl}))$ in the reconstruction formula, implying the minimum value (or the quantized version of it) is part of the bitstream or metadata [S1].
*   **Degenerate Input:** If the scene does not change, residuals are zero, leading to high compression. If the scene changes drastically, residuals may be large, reducing compression efficiency [S1].

## Subtopics

### [Hierarchical Progressive Decoding](12109-hierarchical-progressive-decoding.md)
The decoding phase of ReRF, recovering feature grids from the bitstream using the hierarchical residual structure. It differs by interpreting the encoded residuals and offsets to reconstruct the original multi-resolution grids.

### [Residual Radiance Fields — ReRF](12529-residual-radiance-fields-rerf.md)
The broader framework for dynamic scene modeling that utilizes HPC for compact storage and streaming of long-duration dynamic scenes. It differs by providing the overall architecture for motion and residual grids.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Tested on the DNA-Rendering dataset (2048x2448, 48 views) using an Intel Xeon W-2245 CPU and RTX 3090 GPU [S1].
*   **Scaling:** The method allows for scalable quality adjustments by transmitting fewer resolution levels for lower bandwidths, satisfying volumetric video viewing under various network conditions [S1].
*   **Strengths:** Effectively reduces temporal redundancy in extended sequences; offers scalable quality adjustments; maintains high photorealism and sharpness regardless of frame count [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When streaming dynamic volumetric video with fluctuating bandwidth conditions and high temporal coherence [S1].
*   **When not:** Not given in the sources.

## Background
Hierarchical Progressive Encoding was introduced as part of the ReRF framework for volumetric video compression [S1]. It is described as the first framework for hierarchical progressive coding in this context [S1]. The method leverages the temporal coherence of dynamic scenes by encoding residuals across multiple resolutions [S1].

## Sources
[S1] HPC: Hierarchical Progressive Coding Framework for Volumetric Video, https://doi.org/10.1145/3664647.3681107, fetched 2026-10-28
