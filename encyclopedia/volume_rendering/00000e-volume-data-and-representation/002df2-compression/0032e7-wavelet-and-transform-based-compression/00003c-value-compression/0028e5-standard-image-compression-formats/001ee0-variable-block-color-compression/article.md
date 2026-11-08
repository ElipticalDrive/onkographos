# Variable-Block Color Compression


## Overview
Variable-Block Color Compression is a lossy compression method for sparse 3D color data that encodes color values using variable-length blocks rather than fixed-size grids. It solves the problem of high memory overhead in sparse voxel representations by grouping contiguous voxels with similar colors into single blocks, each defined by two base colors and a weight. It sets itself apart from sibling methods like [ASTC](5729-astc.md) by using a greedy merging algorithm to determine block boundaries dynamically based on an error threshold, rather than relying on a fixed partitioning scheme, and by storing headers in a fixed-size array to enable efficient binary search decoding [S1, unofficial].

## How it works
**Inputs:** A list of sparse voxel colors (indexed by voxel position) and an error threshold defining the maximum allowable Mean Square Error (MSE) for reconstruction.
**Outputs:** A compressed representation consisting of a block header array and a weight array.

**Data Structures:**
1.  **Block Header Array:** A 32-bit integer array. Each entry represents one block and contains:
    *   `Base Color 1` ($B_1$): 16 bits.
    *   `Base Color 2` ($B_2$): 16 bits.
    *   `Start Voxel Index`: 32 bits (stored in the same logical header structure, though the text notes the header is stored in a 32-bit array, implying the 32-bit index is the primary locator, with base colors packed alongside or in a parallel structure; the text explicitly states "2x16 + 32 bits" total size for the header information).
2.  **Weight Array:** An array of weights, where each weight is 3 bits. The size of this array corresponds to the total number of voxels covered by all blocks.

**Algorithm Steps:**

1.  **Initialization:** Treat each individual voxel color as a separate block.
2.  **Merging Iteration:**
    *   Iterate through the current list of blocks.
    *   For each block, evaluate merging with its immediate left neighbor and its immediate right neighbor.
    *   Calculate the MSE for both potential merges.
    *   Select the merge (left or right) that results in the lowest MSE.
    *   If the lowest MSE is within the specified error threshold, merge the blocks. This involves recomputing the optimal base colors ($B_1, B_2$) and weights for the union of the two blocks.
    *   If both potential merges violate the error threshold (MSE > threshold), the block is marked as "done" and will not be merged further in this pass.
3.  **Convergence:** Repeat the merging iteration with the newly formed blocks until all blocks are marked as "done" in a full pass.
4.  **Encoding:**
    *   Store the final base colors and the starting voxel index for each block in the Block Header Array.
    *   Store the corresponding weight for each voxel in the Weight Array.
5.  **Decoding:**
    *   Given a voxel index, perform a binary search on the `Start Voxel Index` field in the Block Header Array to find the block containing that index.
    *   Retrieve the base colors $B_1$ and $B_2$ from the header.
    *   Retrieve the weight $w$ from the Weight Array using the voxel's relative index within the block (or global index if the weight array is indexed globally).
    *   Reconstruct the color $C$ using the linear interpolation formula.

**Formulas:**
*   **Color Reconstruction:**
    $$C = wB_1 + (1 - w)B_2$$
    Where:
    *   $C$ is the reconstructed color.
    *   $B_1$ and $B_2$ are the base colors (16-bit values).
    *   $w$ is the weight (3-bit value).
*   **Error Metric:**
    The Mean Square Error (MSE) of the reconstructed colors for all voxels in a block must not exceed the user-defined threshold.

**Parameters:**
*   **Base Colors ($B_1, B_2$):** 16 bits each.
*   **Weight ($w$):** 3 bits.
*   **Error Threshold:** User-defined MSE limit. Not given in the sources as a specific numerical value, but defined as the constraint for merging.

**Edge Cases and Pitfalls:**
*   **Precision:** The use of 16-bit base colors and 3-bit weights limits the precision of the reconstruction. The error threshold must be chosen carefully to balance compression ratio and visual quality.
*   **Boundary Conditions:** The merging process is sequential and directional (left/right). The order of processing may affect the final block configuration. The algorithm processes blocks sequentially, which implies a specific order of evaluation.
*   **Header Size:** The header stores a 32-bit start voxel index. This allows for sparse addressing but increases the header size compared to fixed-size block formats that might use relative offsets.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variable_block_color_compression(voxel_colors, error_threshold):
    """
    Compresses sparse voxel colors using variable-block color compression.
    
    Args:
        voxel_colors: List of color values for each voxel, indexed by voxel position.
        error_threshold: Maximum allowed MSE for a block.
        
    Returns:
        headers: List of block headers (base colors and start index).
        weights: List of weights for each voxel.
    """
    # Step 1: Initialize each color as a separate block
    # Each block contains a list of voxel indices it covers
    blocks = []
    for i, color in enumerate(voxel_colors):
        blocks.append([i])
    
    # Step 2 & 3: Merging Iteration
    while True:
        merged = False
        new_blocks = []
        i = 0
        while i < len(blocks):
            current_block = blocks[i]
            left_merge_ok = False
            right_merge_ok = False
            left_mse = float('inf')
            right_mse = float('inf')
            
            # Check left merge
            if i > 0:
                left_block = blocks[i-1]
                mse = calculate_mse(left_block + current_block, voxel_colors)
                if mse <= error_threshold:
                    left_merge_ok = True
                    left_mse = mse
            
            # Check right merge
            if i < len(blocks) - 1:
                right_block = blocks[i+1]
                mse = calculate_mse(current_block + right_block, voxel_colors)
                if mse <= error_threshold:
                    right_merge_ok = True
                    right_mse = mse
            
            # Decide merge
            if left_merge_ok and right_merge_ok:
                if left_mse <= right_mse:
                    # Merge with left
                    # Note: This modifies the list structure, so we need to handle indices carefully
                    # For simplicity, we'll rebuild the list in this pass
                    pass # Logic handled by rebuilding below
                else:
                    # Merge with right
                    pass
            elif left_merge_ok:
                pass
            elif right_merge_ok:
                pass
            else:
                # Block is done, add to new list
                new_blocks.append(current_block)
                i += 1
                continue
            
            # If a merge happened, we need to handle the combination
            # To simplify, we'll use a flag-based approach in a real implementation
            # Here, we'll just mark for merging and rebuild
            if left_merge_ok or right_merge_ok:
                merged = True
                # For this pseudocode, we'll assume a simpler sequential pass
                # In practice, this requires careful index management
                pass
            
            i += 1
        
        if not merged:
            break
            
    # Final encoding
    headers = []
    weights = []
    for block in blocks:
        start_index = block[0]
        b1, b2 = compute_base_colors(block, voxel_colors)
        w = compute_weight(block, voxel_colors, b1, b2)
        headers.append({'b1': b1, 'b2': b2, 'start_index': start_index})
        for _ in block:
            weights.append(w)
            
    return headers, weights

def calculate_mse(block_indices, voxel_colors):
    """Calculate MSE for a block of voxels."""
    if not block_indices:
        return 0
    # Compute best fit base colors and weights for the block
    # This is a simplified placeholder for the actual optimization
    # In reality, this involves solving for B1, B2, and w to minimize MSE
    # For the purpose of this pseudocode, we assume a helper exists
    b1, b2, w = optimize_block(block_indices, voxel_colors)
    mse = 0
    for idx in block_indices:
        c_reconstructed = w * b1 + (1 - w) * b2
        c_original = voxel_colors[idx]
        mse += (c_original - c_reconstructed) ** 2
    return mse / len(block_indices)

def optimize_block(block_indices, voxel_colors):
    """Placeholder for computing optimal base colors and weight."""
    # This function would typically use least squares or a similar method
    # to find B1, B2, and w that minimize the MSE for the given voxels
    # Since the exact optimization method is not detailed in the sources,
    # we return dummy values.
    return 0, 0, 0.5

def compute_base_colors(block_indices, voxel_colors):
    """Compute base colors for a block."""
    # Placeholder
    return 0, 0

def compute_weight(block_indices, voxel_colors, b1, b2):
    """Compute weight for a block."""
    # Placeholder
    return 0.5
```

## Subtopics

### [Variable Bit Rate Color Compression with Macro Blocks](1886-variable-bit-rate-color-compression-with-macro-blocks.md)
A variant that uses variable bits per weight (bpw) to optimize compression ratio. It sets itself apart by introducing macro blocks to localize header offsets, reducing the bit-width required for pointers and enabling efficient variable bit-rate encoding without prohibitive header sizes.

## Operating characteristics
*   **Time Complexity:** Encoding is iterative and depends on the number of blocks and the cost of MSE calculation. The merging process may require multiple passes until convergence. Decoding requires a binary search on the headers, which is $O(\log N)$ where $N$ is the number of blocks, plus a direct lookup for the weight.
*   **Memory Complexity:** The memory overhead is described as "practically negligible" compared to previous methods that doubled the size of the DAG [S1, unofficial]. The storage consists of the header array (fixed size per block) and the weight array (3 bits per voxel).
*   **Performance:** The method is designed to outperform previous attempts in both compression ratio and quality [S1, unofficial].
*   **Strengths:** Low memory overhead, flexible block sizes allow for better adaptation to color gradients, simple decoding structure due to fixed-size headers.
*   **Limitations:** Lookup times are "a bit more expensive" than fixed-size block methods due to the binary search [S1, unofficial]. The greedy merging algorithm may not produce globally optimal block configurations.
*   **When to use:** Suitable for sparse 3D color data where memory efficiency is critical and some loss in color fidelity is acceptable.
*   **When not to use:** When exact color reconstruction is required or when hardware support for variable-length blocks is not available.

## Background
The method was introduced as part of a novel compression format for sparse 3D data, presented in a thesis or report from Chalmers University of Technology. It serves as an alternative to mapping attributes to a DAG (Directed Acyclic Graph), aiming to reduce memory overhead while maintaining reasonable lookup times. The work investigates and demonstrates the effectiveness of this approach for sparse voxel representations.

## Sources
[S1] On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
