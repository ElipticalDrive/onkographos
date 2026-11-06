# Differences-based Compression (Mensmann)
Also called: Hybrid Scheme (Mensmann et al.), Mensmann hybrid decompression

## Overview
Differences-based Compression (Mensmann) is a hybrid compression scheme that resolves variable-length coding, delta encoding, and brick assembly during decompression [S1][S2]. It sets itself apart from its siblings by implementing individual kernels for each supported bit length to handle the variable-length coding addressing modes, while performing delta encoding by adding calculated values to existing volume data rather than overwriting it [S1][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Compressed volume data (bricks) stored with variable bit depths; pre-computed shift patterns for data shuffling.
*   **Output:** Reconstructed 16-bit voxel values in the volume buffer.

**Data Structures:**
*   **Shift Patterns Table:** A pre-computed table storing data shuffle patterns used to expand compressed data into 16-bit values [S1][S2].
*   **Volume Buffer:** The target memory location where reconstructed values are accumulated [S1][S2].

**Steps:**
1.  **Resolve Variable-Length Coding:** The decompression kernel selects an addressing mode based on the specific bit length of the current brick [S1][S2]. Individual kernels are implemented for each supported bit length to handle these different addressing modes [S1][S2].
2.  **Expand Data:** Using the pre-computed shift patterns table, the compressed data is looked up and shuffled to expand it into 16-bit values [S1][S2]. The overhead of these lookups is negligible [S1][S2].
3.  **Delta Encoding:** The calculated 16-bit value is added to the existing value in the volume buffer [S1][S2]. This step does not overwrite the existing value but accumulates the delta [S1][S2].
4.  **Brick Assembly:** The reconstructed values are assembled into the final brick structure [S1][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompress_mensmann(compressed_brick, bit_length, shift_table, volume_buffer, brick_offset):
    """
    Decompress a single brick using the Mensmann hybrid scheme.
    
    Args:
        compressed_brick: The compressed data for the brick.
        bit_length: The number of bits used to store this brick (determines kernel/addressing mode).
        shift_table: Pre-computed table of shift patterns for expanding data to 16-bit.
        volume_buffer: The target volume memory array.
        brick_offset: The starting index in volume_buffer for this brick.
    """
    # Step 1 & 2: Resolve variable-length coding and expand to 16-bit
    # Individual kernels are used for different bit lengths.
    # Here we simulate the kernel selection and shift pattern lookup.
    
    if bit_length == 8:
        # Kernel for 8-bit bricks
        expanded_values = apply_shift_patterns(compressed_brick, shift_table['8bit'])
    elif bit_length == 16:
        # Kernel for 16-bit bricks
        expanded_values = apply_shift_patterns(compressed_brick, shift_table['16bit'])
    else:
        # Kernel for other supported bit lengths
        expanded_values = apply_shift_patterns(compressed_brick, shift_table[str(bit_length) + 'bit'])
        
    # Step 3: Delta Encoding
    # Add calculated value to existing value in the volume instead of overwriting
    for i, value in enumerate(expanded_values):
        volume_buffer[brick_offset + i] += value
        
    # Step 4: Brick Assembly
    # The values are now in the volume buffer, ready for rendering
    pass

def apply_shift_patterns(data, pattern):
    """
    Apply pre-computed shift patterns to expand compressed data to 16-bit values.
    This function represents the "data shuffle" operation described in the sources.
    """
    # Implementation details depend on the specific bit-length kernel.
    # The sources state that lookups from the shift_table are used.
    # [G] Assuming a simple expansion for demonstration:
    # In reality, this involves specific bit-shifting and masking based on the pattern.
    expanded = []
    for chunk in data:
        # Simulate expansion using the pattern
        # e.g., if pattern specifies shifting chunk by N bits
        expanded.append(chunk << pattern['shift_amount'])
    return expanded
```

**Parameters:**
*   **Bit Lengths:** The specific bit depths supported by the scheme (e.g., 8-bit, 16-bit). The exact set of supported bit lengths is not given in the sources.
*   **Shift Patterns:** Pre-computed tables used for data shuffling.

**Edge Cases and Pitfalls:**
*   **Bit Length Mismatch:** The kernel must match the bit length of the brick; using the wrong addressing mode will result in incorrect data expansion [S1][S2].
*   **Delta Accumulation:** Since delta encoding adds to existing values, the volume buffer must be initialized correctly (e.g., to zero) before the first delta is applied, or the initial base values must be known [G].

## Subtopics

### Variable-Length Coding Kernels
Individual kernels for each supported bit length to handle different addressing modes. Choose when variable bit depths are required. [Link](5723-differences-based-compression.md)

### Delta Encoding
Adding calculated values to existing volume data. Choose when spatial coherence allows for delta representation. [Link](5723-differences-based-compression.md)

### Brick Assembly
Reconstructing the final brick structure. Choose when block-based processing is needed. [Link](5723-differences-based-compression.md)

## Operating characteristics
*   **Time Complexity:** Decompression involves kernel selection, shift pattern lookups, and addition operations. The overhead of lookups is negligible [S1][S2].
*   **Memory Complexity:** Requires storage for the compressed bricks and the pre-computed shift patterns table [S1][S2].
*   **Performance:** The scheme is designed for GPU-based direct volume rendering, leveraging hardware-supported block-coding techniques [S1][S2].
*   **Strengths:** Efficient handling of variable bit depths through specialized kernels; low overhead for data expansion [S1][S2].
*   **Limitations:** Requires multiple kernels for different bit lengths, increasing implementation complexity [S1][S2].
*   **When to Use:** When variable bit depths are necessary and GPU acceleration is available.
*   **When Not to Use:** When fixed-rate block-coding is sufficient or when implementation simplicity is prioritized over compression flexibility [S1][S2].

## Background
The scheme was proposed by Mensmann et al. in [MRH10] [S1][S2]. It is part of the broader field of compressed GPU-based direct volume rendering, building on fixed-rate block-coding techniques like VTC and S3TC [S1][S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
