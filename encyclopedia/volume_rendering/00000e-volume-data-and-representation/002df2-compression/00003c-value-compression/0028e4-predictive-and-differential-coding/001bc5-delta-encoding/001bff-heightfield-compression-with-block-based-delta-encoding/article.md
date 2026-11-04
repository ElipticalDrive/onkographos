# Heightfield Compression with Block-Based Delta Encoding

Also called: Andujar-inspired compression, Block-based delta encoding

## Overview
This method compresses heightfields by partitioning them into equal-sized blocks, where each block is encoded relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two disk reads: one for the block header and one for the block's data [S1, unofficial]. Unlike its parent method, Delta Encoding, which decouples interval heights in gstacks but lacks efficient random access, this approach balances compression ratio and access time, making it suitable for applications requiring efficient queries or traversals, such as real-time visualization [S1, unofficial].

## How it works
**Inputs:** A heightfield $H$ of dimensions $W \times H$.
**Outputs:** A compressed heightfield buffer containing a header, block descriptors, and encoded difference data.

**Data Structures:**
The compressed data is stored in a linear buffer with the following layout [S1, unofficial]:
1.  **Global Header:** Contains the total number of blocks.
2.  **Block Descriptors:** A sequence of descriptors, one for each block. Each descriptor contains:
    *   `base_value`: The minimum height value in the block.
    *   `num_bits`: The number of bits required to encode the height differences in this block.
    *   `address`: The offset (in bits or bytes) to the start of the block's encoded data in the buffer.
3.  **Encoded Data:** The actual difference values for each block, stored sequentially.

**Parameters:**
*   Block dimensions: $m \times n$. The sources suggest a "relatively small size" to ensure differences remain close to zero, but do not specify exact numeric values [S1, unofficial].
*   Morton encoding is used for both block metadata and height differences to provide spatial coherence during access [S1, unofficial].

**Algorithm Steps:**
1.  Partition the heightfield $H$ into $w \times h$ blocks, where each block $B_{i,j}$ has dimensions $m \times n$.
2.  For each block $B_{i,j}$:
    a.  Identify the lower height value (minimum) in the block as the `base_value`.
    b.  Compute the difference for each element $v$ in the block: $d = v - \text{base\_value}$.
    c.  Determine the minimum number of bits, `num_bits`, required to represent the maximum difference in the block. Since differences are close to zero, this number is significantly smaller than the original bit-depth [S1, unofficial].
    d.  Encode the differences $d$ using `num_bits` per element.
    e.  Store the `base_value`, `num_bits`, and the address of the encoded data in the block descriptor.
3.  Store the global header and the sequence of block descriptors.
4.  Append the encoded difference data for all blocks to the buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_heightfield(H, block_m, block_n):
    """
    Compresses a heightfield H using block-based delta encoding.
    H: 2D array of height values.
    block_m, block_n: dimensions of each block.
    Returns: header, block_descriptors, encoded_data_buffer
    """
    height, width = H.shape
    num_blocks_y = (height + block_m - 1) // block_m
    num_blocks_x = (width + block_n - 1) // block_n
    total_blocks = num_blocks_y * num_blocks_x
    
    header = {"num_blocks": total_blocks}
    block_descriptors = []
    encoded_data_buffer = []
    
    for j in range(num_blocks_y):
        for i in range(num_blocks_x):
            # Extract block, handling boundaries
            y_start = j * block_m
            x_start = i * block_n
            y_end = min(y_start + block_m, height)
            x_end = min(x_start + block_n, width)
            
            block_data = H[y_start:y_end, x_start:x_end]
            
            # Step 2a: Find base value (minimum)
            base_value = np.min(block_data)
            
            # Step 2b: Compute differences
            differences = block_data - base_value
            
            # Step 2c: Determine bit depth
            max_diff = np.max(differences)
            if max_diff <= 0:
                num_bits = 1 # Minimum 1 bit to store 0
            else:
                num_bits = max_diff.bit_length()
            
            # Step 2d: Encode differences
            # Note: In a real implementation, this would pack bits into bytes
            # For this pseudocode, we assume a helper that packs integers
            packed_block = pack_bits(differences, num_bits)
            
            # Record address before appending
            current_address = len(encoded_data_buffer)
            
            # Step 2e: Store descriptor
            block_descriptors.append({
                "base_value": base_value,
                "num_bits": num_bits,
                "address": current_address
            })
            
            # Append to buffer
            encoded_data_buffer.append(packed_block)
            
    return header, block_descriptors, encoded_data_buffer

def decompress_heightfield(header, block_descriptors, encoded_data_buffer, target_shape):
    """
    Decompresses a heightfield using block-based delta encoding.
    """
    H = np.zeros(target_shape, dtype=np.float32) # Or appropriate type
    num_blocks_y = (target_shape[0] + block_m - 1) // block_m
    num_blocks_x = (target_shape[1] + block_n - 1) // block_n
    
    for idx, desc in enumerate(block_descriptors):
        # Calculate block coordinates
        j = idx // num_blocks_x
        i = idx % num_blocks_x
        
        y_start = j * block_m
        x_start = i * block_n
        y_end = min(y_start + block_m, target_shape[0])
        x_end = min(x_start + block_n, target_shape[1])
        
        # Read block data
        packed_block = encoded_data_buffer[desc["address"]]
        differences = unpack_bits(packed_block, desc["num_bits"])
        
        # Reconstruct heights
        block_data = differences + desc["base_value"]
        H[y_start:y_end, x_start:x_end] = block_data
        
    return H
```

**Edge Cases and Pitfalls:**
*   **Boundary Blocks:** Blocks at the edges of the heightfield may be smaller than $m \times n$. The algorithm must handle these partial blocks correctly by adjusting the extraction and reconstruction logic [S1, unofficial].
*   **Precision:** The choice of `num_bits` must be sufficient to represent the maximum difference in the block. If the block is too large, differences may exceed the allocated bits, leading to clipping and reconstruction errors [S1, unofficial].
*   **Access Efficiency:** While random access is improved, it still requires two reads. If the block descriptors are not cached, this overhead can be significant [S1, unofficial].

## Subtopics

### QuadStack Integration
This compression method is specifically designed to work with QuadStacks, where heightfields are associated with specific levels of the QuadStack hierarchy. The buffer includes an index indicating the base level of the QuadStack [S1, unofficial].

### Morton Encoding
The use of Morton order for block metadata and height differences ensures spatial coherence, which is beneficial for cache performance during traversal [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Random access to a specific height value requires two reads: one to the header (to find the block descriptor) and one to the bit field (to read the block data) [S1, unofficial]. Decompression of a single block is linear in the size of the block.
*   **Memory Complexity:** The compressed size depends on the block size and the variance of heights within blocks. Smaller blocks generally lead to better compression if height variation is low, but increase the overhead of block descriptors [S1, unofficial].
*   **Strengths:** Enables efficient random access and traversal, which is critical for real-time visualization and interactive applications [S1, unofficial].
*   **Limitations:** Compression ratio may degrade if height variation within blocks is high. The method is less efficient than standard delta encoding for sequential access, where the latter can decompress the entire dataset in a single pass [S1, unofficial].

## Background
This method was proposed by Andujar et al. as an improvement over standard delta encoding for heightfields in the context of layered datasets [S1, unofficial]. It was introduced in the QuadStack paper, which presents an efficient representation and direct rendering of layered datasets [S1, unofficial]. The method addresses the practical limitations of delta encoding in applications requiring efficient queries or traversals [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
