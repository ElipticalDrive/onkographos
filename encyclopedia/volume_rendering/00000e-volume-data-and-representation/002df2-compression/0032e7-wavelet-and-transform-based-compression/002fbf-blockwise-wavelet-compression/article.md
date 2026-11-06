# Blockwise Wavelet Compression

Also called: Blockwise wavelet

## Overview
Blockwise wavelet compression is a data compression technique that decomposes 3D volumetric data into small, equal-sized blocks to enable independent compression and decompression along all three axes [S1]. It distinguishes itself from global wavelet methods by localizing the transform, which facilitates parallel processing and localized detail extraction for rendering [S1].

## How it works
**Inputs:** A 3D volumetric dataset $V$.
**Outputs:** A compressed bitstream or a set of compressed blocks.

**Data Structures:**
The volume is partitioned into a grid of non-overlapping 3D blocks. Each block $B$ has dimensions $N_x \times N_y \times N_z$, where $N_x, N_y, N_z$ are typically powers of two to support recursive wavelet decomposition [S1].

**Steps:**
1.  **Partitioning:** Split the input 3D data $V$ into small blocks of equal size [S1].
2.  **Transform:** Apply a 3D wavelet transform to each block individually. This involves applying separable 1D filter banks along the x, y, and z axes [S1].
3.  **Compression:** Compress the resulting wavelet coefficients for each block. This typically involves quantization and entropy coding [S1].
4.  **Decompression:** To reconstruct, decompress the coefficients and apply the inverse 3D wavelet transform to each block individually [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blockwise_wavelet_compress(volume, block_size):
    """
    Compress a 3D volume using blockwise wavelet decomposition.
    
    Args:
        volume: 3D array of shape (D, H, W)
        block_size: Tuple (dz, dy, dx) defining the size of each block
        
    Returns:
        List of compressed block data
    """
    compressed_blocks = []
    D, H, W = volume.shape
    
    # Iterate over the volume in blocks
    for z in range(0, D, block_size[0]):
        for y in range(0, H, block_size[1]):
            for x in range(0, W, block_size[2]):
                # Extract block, handling boundaries if necessary
                z_end = min(z + block_size[0], D)
                y_end = min(y + block_size[1], H)
                x_end = min(x + block_size[2], W)
                
                block = volume[z:z_end, y:y_end, x:x_end]
                
                # Apply 3D wavelet transform
                # This is a separable transform: apply 1D wavelet along each axis
                coeffs = wavelet_transform_3d(block)
                
                # Compress coefficients (quantize + entropy code)
                compressed_data = compress_coefficients(coeffs)
                
                compressed_blocks.append(compressed_data)
                
    return compressed_blocks

def blockwise_wavelet_decompress(compressed_blocks, block_size, output_shape):
    """
    Reconstruct a 3D volume from compressed blocks.
    
    Args:
        compressed_blocks: List of compressed block data
        block_size: Tuple (dz, dy, dx)
        output_shape: Tuple (D, H, W) of the original volume
        
    Returns:
        Reconstructed 3D volume
    """
    volume = np.zeros(output_shape)
    idx = 0
    
    for z in range(0, output_shape[0], block_size[0]):
        for y in range(0, output_shape[1], block_size[1]):
            for x in range(0, output_shape[2], block_size[2]):
                z_end = min(z + block_size[0], output_shape[0])
                y_end = min(y + block_size[1], output_shape[1])
                x_end = min(x + block_size[2], output_shape[2])
                
                # Decompress coefficients
                coeffs = decompress_coefficients(compressed_blocks[idx])
                
                # Apply inverse 3D wavelet transform
                block = inverse_wavelet_transform_3d(coeffs)
                
                # Place block back into volume
                volume[z:z_end, y:y_end, x:x_end] = block
                idx += 1
                
    return volume
```

**Edge Cases:**
*   **Boundary Conditions:** If the volume dimensions are not divisible by the block size, the last blocks along each axis will be smaller. The implementation must handle these partial blocks correctly during both compression and decompression [S1].
*   **Precision:** Wavelet transforms can introduce precision errors. Using orthogonal or biorthogonal wavelets with sufficient precision is recommended to avoid artifacts [S1].

## Subtopics

### [Hierarchical Wavelet Compression](6108-hierarchical-wavelet-compression.md)
Extends block-based compression by using a hierarchical wavelet representation, allowing for multi-resolution access.

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of voxels, $O(N)$, where $N$ is the total number of voxels, as each voxel is processed once per transform level [S1]. Memory complexity is also $O(N)$ for storing the coefficients [S1].
*   **Performance:** The blockwise approach allows for parallel processing of blocks, which can accelerate compression and decompression on multi-core or GPU architectures [S1].
*   **Strengths:** Enables localized detail extraction and parallel processing [S1].
*   **Limitations:** May introduce blocking artifacts at block boundaries if not handled carefully [S1].

## Background
Blockwise wavelet compression was proposed by Nguyen et al. as a method to split 3D data into small blocks for individual compression and decompression [S1]. Guthe et al. later employed block-based wavelets to compress datasets and extract levels of detail on the fly [S1]. Xie et al. further extended this concept by introducing hierarchical wavelet representation [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
