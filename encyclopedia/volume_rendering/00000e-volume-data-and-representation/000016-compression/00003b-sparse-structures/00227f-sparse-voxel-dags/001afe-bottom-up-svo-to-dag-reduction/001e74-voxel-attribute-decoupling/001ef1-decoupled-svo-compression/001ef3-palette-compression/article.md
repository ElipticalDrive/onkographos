# Palette Compression

Also called: Block-based palette compression, Variable-length palette compression

## Overview
Palette compression is a lossless compression scheme for the attribute array in decoupled Sparse Voxel Octrees (SVOs) that reduces memory footprint by exploiting spatial coherence and attribute redundancy [S1]. It replaces full attribute storage with variable-length indices into local palettes, which in turn point to a global material array [S1]. Unlike fixed-width indexing, this method adapts the bit-width per block based on the number of unique attributes present, achieving higher compression ratios for homogeneous regions while maintaining fast random access via a block directory [S1].

## How it works
**Inputs:** An uncompressed attribute array $A$ of size $\Lambda$, where $\Lambda$ equals the total voxel count in the original SVO.
**Outputs:** An array of blocks $\{B_0, \dots, B_{\gamma-1}\}$, an array of palettes $\{P_0, \dots, P_{\rho-1}\}$, and a block directory.

**Data Structures:**
1.  **Global Material Array ($M$):** An array of size $\lambda$ storing all unique attributes found in $A$. $\lambda \ll \Lambda$ [S1].
2.  **Indexed Attribute Array ($A^e$):** An array of size $\Lambda$ where each entry $m_i$ is an index into $M$ [S1].
3.  **Block ($B$):** A contiguous sub-segment of $A^e$.
4.  **Palette ($P$):** An array of unique indices into $M$ required to reconstruct all attributes in a specific block. The size of $P$ is $2^\omega$, where $\omega$ is the bit-width [S1].
5.  **Block Directory:** An array of entries ordered by the starting node index of each block. Each entry stores the starting index and the palette ID (or palette data) [S1].

**Algorithm Steps:**

1.  **Global Indexing:** Construct $M$ containing all $\lambda$ unique attributes from $A$. Replace $A$ with $A^e$, where each $m_i \in [0, \lambda-1]$ [S1].
2.  **Block Partitioning (Greedy Heuristic):** Partition $A^e$ into blocks using two phases [S1]:
    *   **Phase 1 (Large Homogeneous Blocks):** Iterate $\omega$ from 0 to 3. Find the largest contiguous block in the current unassigned range that contains exactly $2^\omega$ unique indices. If the memory cost of this block is less than the cost of storing the indices without a palette (i.e., using $\omega+1$ bits), create a palette for this block, replace the indices with local palette indices, and recursively process the remaining unassigned data on both sides of the block. If the criterion fails, increment $\omega$. If $\omega \ge 4$, terminate Phase 1 [S1].
    *   **Phase 2 (Remaining Data):** For the remaining unassigned data, consider blocks starting at the current position for $\omega \in \{0, \dots, 8\}$. Select the block that minimizes the memory per entry metric $S_\omega$. Create a palette, replace indices, and recursively process the remainder [S1].
3.  **Palette Management:** If a block's required palette already exists in the palette array, link the block to it. Otherwise, append the new palette to the array [S1].
4.  **Directory Construction:** Create the block directory. Entries are sorted by the starting index of the block in $A^e$ to enable binary search during decoding [S1].

**Memory Formula:**
The memory cost for a block $B$ spanning indices $m_i$ to $m_j$ with bit-width $\omega$ is [S1]:
$$ \text{MEMORY}(\{m_i, \dots, m_j\}, \omega) = (j - i + 1) \cdot \omega + 2^\omega \cdot \lceil \log_2 \lambda \rceil + \text{size}(\text{directory entry}) $$
Where:
*   $(j - i + 1) \cdot \omega$: Bits for the block indices.
*   $2^\omega \cdot \lceil \log_2 \lambda \rceil$: Bits for the palette entries (each entry is an index into $M$).
*   $\text{size}(\text{directory entry})$: Overhead for the directory record [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def palette_compression(attribute_array_A):
    # Step 1: Global Indexing
    unique_attrs = set(attribute_array_A)
    M = list(unique_attrs) # Global Material Array
    lambda_val = len(M)
    # Map original attributes to indices in M
    attr_to_idx = {attr: i for i, attr in enumerate(M)}
    A_e = [attr_to_idx[attr] for attr in attribute_array_A]
    Lambda = len(A_e)
    
    blocks = []
    palettes = []
    palette_map = {} # Maps tuple(sorted_indices) -> palette_index
    
    # Helper to calculate memory cost
    def memory_cost(start_idx, end_idx, omega):
        num_entries = end_idx - start_idx + 1
        palette_bits = (1 << omega) * math.ceil(math.log2(lambda_val))
        dir_entry_size = 8 # Assumed size for directory entry, e.g., 4 bytes start + 4 bytes palette_ref
        return num_entries * omega + palette_bits + dir_entry_size

    # Helper to create palette
    def create_palette(block_indices):
        unique_indices = sorted(list(set(block_indices)))
        # Check if palette already exists
        key = tuple(unique_indices)
        if key in palette_map:
            return palette_map[key]
        new_palette_idx = len(palettes)
        palettes.append(unique_indices)
        palette_map[key] = new_palette_idx
        return new_palette_idx

    # Phase 1: Greedy Heuristic for small omega
    def find_large_blocks(start, end, omega_limit=4):
        if start > end:
            return
        omega = 0
        while omega < omega_limit:
            # Find largest block with 2^omega unique indices
            # This requires scanning for the longest segment with <= 2^omega unique values
            # Simplified logic: brute force search for largest valid block
            best_block = None
            best_len = 0
            
            # Note: The source describes finding the "largest block" with exactly 2^omega unique indices.
            # In practice, this implies finding the longest contiguous subarray with cardinality <= 2^omega.
            # We approximate this by checking segments.
            
            # Optimization: The source says "brute-force fashion". 
            # We implement a scan to find the longest segment with unique count <= 2^omega.
            max_unique = 1 << omega
            i = start
            while i <= end:
                # Expand j from i
                j = i
                current_unique = set()
                while j <= end and len(current_unique) < max_unique:
                    current_unique.add(A_e[j])
                    j += 1
                # If we hit exactly max_unique or ran out, check length
                # The source says "exactly 2^omega", but usually implies <= for compression gain.
                # However, strict reading: "block with 2^omega unique m".
                # If len(current_unique) < max_unique, we can potentially extend or it's smaller.
                # Let's assume we take the segment [i, j-1] if it has <= max_unique unique.
                
                current_len = j - i
                if current_len > best_len:
                    best_len = current_len
                    best_block = (i, j - 1)
                
                # Move i forward. To find ALL large blocks, we might need to slide.
                # But the source implies finding THE largest block in the current range.
                i += 1
            
            if best_block is None:
                break
                
            b_start, b_end = best_block
            b_indices = A_e[b_start : b_end + 1]
            unique_in_block = len(set(b_indices))
            
            # Check memory criterion
            # Cost with palette vs Cost without (using omega+1 bits for indices)
            # Source: MEMORY(B, omega) < (l - k + 1) * (omega + 1)
            # Note: (l-k+1) is the block length.
            cost_with_palette = memory_cost(b_start, b_end, omega)
            cost_without_palette = (b_end - b_start + 1) * (omega + 1)
            
            if cost_with_palette < cost_without_palette:
                palette_idx = create_palette(b_indices)
                blocks.append((b_start, b_end, palette_idx, omega))
                # Replace indices in A_e with local palette indices
                for k in range(b_start, b_end + 1):
                    val = A_e[k]
                    # Find local index
                    local_idx = palettes[palette_idx].index(val)
                    A_e[k] = local_idx # Store local index temporarily? 
                    # Actually, we store the compressed block. A_e is modified to hold local indices?
                    # The source says "replace the material indices m with indices into this palette".
                    # This implies A_e is overwritten.
                
                # Recurse on remainders
                find_large_blocks(start, b_start - 1, omega_limit)
                find_large_blocks(b_end + 1, end, omega_limit)
                return
            else:
                omega += 1
        
        # If Phase 1 fails for this range, go to Phase 2
        find_remaining_blocks(start, end)

    # Phase 2: Remaining Data
    def find_remaining_blocks(start, end):
        if start > end:
            return
        
        best_block = None
        best_omega = -1
        best_cost_per_entry = float('inf')
        
        # Consider omega from 0 to 8
        for omega in range(9):
            max_unique = 1 << omega
            # Find largest block starting at 'start' with <= max_unique unique indices
            i = start
            current_unique = set()
            j = start
            while j <= end and len(current_unique) < max_unique:
                current_unique.add(A_e[j])
                j += 1
            
            # Block is [start, j-1]
            b_start = start
            b_end = j - 1
            b_len = b_end - b_start + 1
            
            if b_len == 0:
                continue
                
            cost = memory_cost(b_start, b_end, omega)
            cost_per_entry = cost / b_len
            
            if cost_per_entry < best_cost_per_entry:
                best_cost_per_entry = cost_per_entry
                best_block = (b_start, b_end)
                best_omega = omega
        
        if best_block:
            b_start, b_end = best_block
            b_indices = A_e[b_start : b_end + 1]
            palette_idx = create_palette(b_indices)
            blocks.append((b_start, b_end, palette_idx, best_omega))
            
            for k in range(b_start, b_end + 1):
                val = A_e[k]
                local_idx = palettes[palette_idx].index(val)
                A_e[k] = local_idx
                
            find_remaining_blocks(b_end + 1, end)

    # Initial Call
    find_large_blocks(0, Lambda - 1)
    
    # Construct Directory
    # Directory entries ordered by starting node index
    directory = []
    for b_start, b_end, p_idx, omega in blocks:
        directory.append({
            'start': b_start,
            'palette_id': p_idx,
            'omega': omega,
            'length': b_end - b_start + 1
        })
    directory.sort(key=lambda x: x['start'])
    
    return blocks, palettes, directory
```

**Edge Cases and Pitfalls:**
*   **Combinatorial Complexity:** Finding the optimal block partitioning is NP-hard. The greedy heuristic is an approximation [S1].
*   **Phase 1 Infeasibility:** Phase 1 becomes computationally expensive for large $\omega$. It is terminated when $\omega \ge 4$ (16 unique indices) [S1].
*   **Precision:** The memory formula assumes fixed sizes for directory entries and material indices. Variations in $\lambda$ affect $\lceil \log_2 \lambda \rceil$ [S1].

## Subtopics

### [Palette-based compression for chunked discrete voxel data](3130-palette-based-compression-for-chunked-discrete-voxel-data.md)
A variant for discrete voxel data (e.g., Minecraft) that uses `BlockStorage` and variable-bit-length indices. It balances memory usage by using fewer bits for low-entropy regions and more for high-entropy regions, similar to GIF palettes.

## Operating characteristics
*   **Time Complexity:** The compression phase is dominated by the greedy heuristic. Phase 1 scans for large blocks, which is $O(\Lambda^2)$ in the worst case without optimization, but limited by $\omega < 4$. Phase 2 is $O(\Lambda)$ as it processes sequentially [S1].
*   **Memory Complexity:** The compressed size depends on the spatial coherence. For highly coherent scenes, $\lambda \ll \Lambda$ and many blocks have small $\omega$, leading to significant reduction. The overhead includes the global material array $M$, the block directory, and the palettes [S1].
*   **Performance:** Allows fast random access because the block directory supports binary search, and palettes are small enough to fit in cache [S1].
*   **Strengths:** High compression ratio for scenes with spatially coherent attributes (e.g., large areas of same material). Decouples geometry from attributes [S1].
*   **Limitations:** Compression is lossless but computationally intensive during encoding. Random access requires two lookups (directory then palette) [S1].

## Background
Palette compression was introduced in the context of geometry and attribute compression for voxel scenes, specifically as part of a decoupled SVO compression scheme [S1]. It leverages the observation that voxel attributes often exhibit spatial coherence and redundancy, similar to color palettes in image formats like GIF [S1]. The method was developed to address the large memory footprint of uncompressed attribute arrays in high-resolution voxel scenes [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
