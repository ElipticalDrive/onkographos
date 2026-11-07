# Haar Wavelet Compression

Also called: 3D Haar Wavelet Compression, 3D Haar Wavelet Compression with Random Access, Ihm and Park method, New Coder, Wavelet-Based 3D Volume Compression with Fast Random Access

## Overview
Haar Wavelet Compression is a lossy compression scheme for 3D volume data designed to achieve high compression ratios while supporting fast random access to individual voxels, enabling interactive visualization of large datasets [S1, unofficial][S2]. It differs from traditional wavelet coders (like SPIHT or zerotree-based methods) by using a block-based data structure with fixed-length codes and spatial coherence flags, avoiding variable-length entropy coding that hinders random access [S2]. It also differs from pure 3D wavelet methods by treating the volume as a sequence of 2D slices, applying temporal prediction to remove inter-slice correlation before performing 2D spatial wavelet decomposition, which reduces computational cost compared to full 3D transforms [S2].

## How it works
The method partitions the volume into independent blocks, applies a hierarchical Haar wavelet transform, thresholds and quantizes coefficients, and encodes them using a structure optimized for direct memory access [S1, unofficial][S2].

### Inputs and Outputs
- **Input**: A 3D volume dataset $V$ of size $W \times H \times D$, where voxel values are typically 12-bit integers (range $[0, 2^{12}-1]$) [S1, unofficial].
- **Output**: Compressed bitstream consisting of coefficient values and metadata allowing reconstruction of any voxel without decoding the entire volume [S1, unofficial][S2].

### Data Structures
The compressed data is organized into two main streams and one metadata chunk per block:
1.  **`one-byte` array**: Stores quantized coefficients that fit in 1 byte (values $[0, 255]$) [S1, unofficial].
2.  **`two-byte` array**: Stores quantized coefficients requiring 2 bytes [S1, unofficial].
3.  **Cell Information Chunk**: A $4 \times 4 \times 4$ block of 1-bit flags and offset information. Each bit corresponds to a "cell" (a $4 \times 4 \times 4$ subblock of the original $16 \times 16 \times 16$ block). If the bit is 1, the cell contains non-zero coefficients; if 0, all coefficients in that cell are zero [S1, unofficial]. Offsets point to the location of the cell's coefficients in the `one-byte` or `two-byte` arrays [S1, unofficial].

### Parameters
- **Block Size**: $16 \times 16 \times 16$ voxels [S1, unofficial].
- **Subblock Size**: $4 \times 4 \times 4$ voxels (called "cells") [S1, unofficial].
- **Wavelet**: 2-tap Haar wavelet [S2].
- **Decomposition Levels**: 2 levels [S2].
- **Slice Prediction Distance ($F$)**: $F = 2^k$ for faster processing, where $k$ is chosen based on compression/quality trade-offs [S2].
- **Quantization Range**: Non-zero coefficients are quantized to $[0, 255]$ [S2].

### Steps

1.  **Partitioning**: Divide the volume into $16 \times 16 \times 16$ blocks [S1, unofficial]. Within each block, identify $4 \times 4 \times 4$ subblocks (cells) [S1, unofficial].
2.  **Temporal Prediction (Inter-slice)**: Divide the volume into 2D slices along the z-direction [S2]. Apply temporal prediction to remove correlation between slices. This involves using forward and backward R-slices (reference slices) to predict I-slices (intra-coded slices) [S2]. The distance between R-slices is $F = 2^k$ [S2].
3.  **Spatial Wavelet Transform**: Apply a 2D Haar wavelet transform to each slice to handle spatial redundancy [S2]. The Haar wavelet is chosen for its fast reconstruction speed (2-tap filter) [S2]. Two levels of decomposition are applied [S2].
4.  **Thresholding**: Set wavelet coefficients below a certain threshold to zero to increase sparsity [S1, unofficial][S2].
    -   The threshold is determined per-slice to maintain a target Peak Signal-to-Noise Ratio (PSNR) [S2].
    -   Coefficients in each slice are sorted in ascending order of magnitude. Coefficients are zeroed until the desired PSNR is reached [S2].
    -   PSNR is calculated using the global maximum of the entire volume:
        $$ \text{PSNR} = 10 \log_{10} \left( \frac{\max(x_i)^2}{\text{MSE}} \right) $$
        where $x_i$ is the original image value [S2].
    -   *Note*: Global sorting is impractical for large volumes; per-slice sorting is used as a practical approximation [S2].
5.  **Quantization**: Quantize the remaining non-zero coefficients to the interval $[0, 255]$ using a uniform scalar quantizer [S2]. Coefficients are grouped into blocks $G$ of at most 64 coefficients [S2].
    -   For each block $G$, determine parameters:
        $$ A = \max_{c \in G}(c), \quad B = \min_{c \in G}(c) $$
    -   Quantize each coefficient $C \in G$ to a byte $x$:
        $$ x = \left\lfloor 255 \times \frac{C - B}{A - B} + 0.5 \right\rfloor $$
    -   Store $A$ and $B$ as 16-bit integers and $x$ as a byte [S2].
    -   Reconstructed value $\tilde{C}$ from quantized $x$:
        $$ \tilde{C} = \left\lfloor \frac{A - B}{255} \times x + 0.5 \right\rfloor + B $$
6.  **Encoding**:
    -   Enumerate cells within each $16 \times 16 \times 16$ block.
    -   If a cell contains any non-zero coefficients, tag it with a 1 in the Cell Information Chunk and store its coefficients in the `one-byte` or `two-byte` arrays based on size [S1, unofficial].
    -   If a cell contains only zero coefficients, tag it with 0 [S1, unofficial].
    -   Use flags and offsets in the Cell Information Chunk to map cell indices to addresses in the coefficient arrays [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def haar_wavelet_compress(volume, target_psnr, k):
    """
    Compress a 3D volume using Haar Wavelet Compression with random access.
    
    Args:
        volume: 3D numpy array of shape (W, H, D), dtype uint16 (12-bit values).
        target_psnr: Desired PSNR in dB.
        k: Exponent for slice prediction distance F = 2^k.
        
    Returns:
        compressed_data: Dictionary containing 'one_byte', 'two_byte', 'cell_info', 'block_params'.
    """
    W, H, D = volume.shape
    block_size = 16
    cell_size = 4
    
    # Initialize storage
    one_byte_stream = []
    two_byte_stream = []
    cell_info = [] # List of (flag, offset_to_one_byte, offset_to_two_byte)
    block_params = [] # Store A, B for quantization blocks
    
    # 1. Partition into blocks
    for z in range(0, D, block_size):
        for y in range(0, H, block_size):
            for x in range(0, W, block_size):
                # Extract block
                block = volume[z:z+block_size, y:y+block_size, x:x+block_size]
                
                # 2. Temporal Prediction (Simplified: Assume slices are processed in order)
                # Note: Full implementation requires managing R-slices and I-slices across blocks.
                # Here we assume block-level processing for clarity, but prediction happens at slice level.
                # For this pseudocode, we skip the complex slice prediction logic and assume 
                # the input 'block' is already prediction-corrected or prediction is handled externally.
                
                # 3. Spatial Wavelet Transform (2D Haar, 2 levels)
                # Apply 2D Haar wavelet to each slice in the block
                transformed_block = apply_2d_haar_2_levels(block)
                
                # 4. Thresholding
                # Determine threshold per slice to meet PSNR
                # This requires calculating MSE and PSNR for each slice
                thresholded_block = threshold_slices(transformed_block, target_psnr, volume)
                
                # 5. Quantization and Encoding
                # Process cells (4x4x4 subblocks)
                cell_flags = []
                current_one_byte_offset = len(one_byte_stream)
                current_two_byte_offset = len(two_byte_stream)
                
                for cz in range(0, block_size, cell_size):
                    for cy in range(0, block_size, cell_size):
                        for cx in range(0, block_size, cell_size):
                            cell = thresholded_block[cz:cz+cell_size, cy:cy+cell_size, cx:cx+cell_size]
                            
                            # Check if cell has non-zero coefficients
                            if np.any(cell != 0):
                                cell_flags.append(1)
                                
                                # Group coefficients for quantization (max 64)
                                coeffs = cell.flatten()
                                coeffs = coeffs[coeffs != 0] # Remove zeros if any slipped through
                                
                                # Quantize to [0, 255]
                                A = np.max(coeffs)
                                B = np.min(coeffs)
                                
                                # Store A, B for this block (assuming one block per 64 coeffs for simplicity)
                                # In reality, A and B are stored per quantization group G
                                block_params.append({'A': A, 'B': B})
                                
                                # Quantize
                                quantized = np.floor(255 * (coeffs - B) / (A - B) + 0.5).astype(np.uint8)
                                
                                # Store in streams
                                one_byte_stream.extend(quantized)
                                
                                # Note: If coefficients required 2 bytes, they would go to two_byte_stream
                                # For 12-bit input and 2 levels, most fit in 1 byte after quantization
                            else:
                                cell_flags.append(0)
                                
                # Store cell info
                cell_info.append({
                    'flags': cell_flags,
                    'offset_one_byte': current_one_byte_offset,
                    'offset_two_byte': current_two_byte_offset
                })
                
                # Update offsets for next cell
                # (In a real implementation, offsets are cumulative)
                
    return {
        'one_byte': np.array(one_byte_stream, dtype=np.uint8),
        'two_byte': np.array(two_byte_stream, dtype=np.uint16),
        'cell_info': cell_info,
        'block_params': block_params
    }

def apply_2d_haar_2_levels(block):
    """
    Apply 2D Haar wavelet transform with 2 decomposition levels.
    """
    # Level 1
    # Transform rows
    for z in range(block.shape[0]):
        for y in range(block.shape[1]):
            row = block[z, y, :]
            block[z, y, :] = haar_transform_1d(row)
    # Transform columns
    for z in range(block.shape[0]):
        for x in range(block.shape[1]):
            col = block[z, :, x]
            block[z, :, x] = haar_transform_1d(col)
            
    # Level 2 (on the approximation subband)
    # This is a simplified representation; actual implementation handles subbands carefully
    # ...
    return block

def haar_transform_1d(signal):
    """
    1D Haar wavelet transform.
    """
    n = len(signal)
    if n % 2 != 0:
        signal = np.append(signal, signal[-1]) # Pad if necessary
    n = len(signal)
    approx = (signal[0::2] + signal[1::2]) / 2
    detail = (signal[0::2] - signal[1::2]) / 2
    return np.concatenate([approx, detail])

def threshold_slices(transformed_block, target_psnr, original_volume):
    """
    Threshold coefficients per slice to achieve target PSNR.
    """
    # This is a placeholder for the complex PSNR calculation logic
    # Iterate over each slice in the block
    # Sort coefficients, zero out small ones, calculate PSNR
    # Repeat until PSNR >= target_psnr
    # Use global max of original_volume for PSNR calculation
    pass
```

### Edge Cases and Pitfalls
-   **Global Sorting Impracticality**: Sorting all coefficients globally for thresholding is too slow for large volumes. The method uses per-slice sorting, which is not globally optimal but provides a reasonable solution [S2].
-   **Precision**: Original voxel values are 12-bit. Averages fall in $[0, 2^{12}]$ and details in $[-(2^{12}-1)/2, (2^{12}-1)/2]$ [S1, unofficial]. Quantization to 8-bit introduces loss, but the uniform scalar quantizer minimizes distortion [S2].
-   **Boundaries**: The block-based approach requires volumes to be divisible by block size ($16 \times 16 \times 16$). Padding may be needed for non-divisible dimensions [G].

## Operating characteristics
-   **Time Complexity**: The use of the 2-tap Haar wavelet and 2 decomposition levels ensures fast reconstruction. Data reconstruction time is imperceptible compared to rendering tasks [S1, unofficial].
-   **Memory Complexity**: The block-based structure with flags allows random access to any voxel by decoding only the relevant cell and its parent nodes, avoiding decoding the entire volume [S1, unofficial][S2].
-   **Compression Ratio**: Compression ratios of up to 28 were reported with good fidelity [S2]. Other methods like Laplacian Pyramid achieved ~10, and 2D wavelet subband coding achieved 20-60 but lacked fast random access [S2].
-   **Strengths**: Fast random access, high compression ratios, interactive visualization support [S1, unofficial][S2].
-   **Limitations**: Lossy compression; artifacts may appear at high compression factors (e.g., factor 5 showed blocking and contouring artifacts in some comparisons, though Ihm and Park reported good fidelity at factor 28) [S2]. The per-slice thresholding is not globally optimal [S2].

## Background
The method was presented by Insung Ihm and Sanghun Park [S1, unofficial][S2]. It was developed to address the limitations of previous wavelet compression schemes that either supported high compression ratios or fast random access, but not both [S2]. It builds on the idea of treating 3D volume data as a sequence of 2D slices, borrowing techniques from video compression such as temporal prediction [S2]. The use of Haar wavelets was chosen for their computational efficiency [S2].

## Sources
[S1] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S2] Wavelet Based 3D Compression for Very Large Volume Data Supporting Fast Random Access, https://www.brics.dk/RS/99/34/index.html, fetched 2026-11-04
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
