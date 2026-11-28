# Palette Compression

Also called: Block-based palette compression, Variable-length palette compression

## Overview
Palette compression is a variable-length compression scheme for the attribute array in decoupled Sparse Voxel Octrees (SVOs) that reduces memory footprint by exploiting spatial coherence and attribute redundancy [S1]. It differs from fixed-size array storage by replacing global material indices with local palette indices, allowing the bit-width per voxel to adapt to the local entropy of the attribute data [S1].

## How it works
**Inputs and Outputs**
*   **Input:** An indexed attribute array $A^e = \{m_0, \dots, m_{\Lambda-1}\}$, where each $m_i$ is a global index into a global material array $M$ [S1]. $\Lambda$ is the total number of entries, equal to the voxel count in the original SVO [S1].
*   **Output:** An array of blocks $\{B_0, \dots, B_{\gamma-1}\}$, an array of palettes $\{P_0, \dots, P_{\rho-1}\}$, and a block directory [S1].

**Data Structures**
1.  **Global Material Array ($M$):** Stores all $\lambda$ unique attributes in the scene as $M = \{a^e_0, \dots, a^e_{\lambda-1}\}$ [S1].
2.  **Block Directory:** An array ordered by starting node index to indicate where blocks start and their corresponding palette, enabling binary search [S1]. Each entry indicates the start index of a block and the index of its associated palette [S1].
3.  **Block ($B$):** A contiguous sub-segment of the attribute array that stores local indices into its associated palette [S1].
4.  **Palette ($P$):** An array of unique global indices into $M$ required to retrieve all attributes present in its associated block(s) [S1].

**Parameters**
*   $\Lambda$: Total number of entries in the attribute array [S1].
*   $\lambda$: Number of unique voxel attributes, typically orders of magnitude smaller than $\Lambda$ [S1].
*   $\omega$: Bit-width for local indices in a block, determined by the number of unique entries in the block ($|P| \le 2^\omega$) [S1].

**Algorithm Steps**
1.  **Global Indexing:** Construct $M$ containing all unique attributes from the original attribute array and replace $A$ with the indexed version $A^e$ [S1].
2.  **Block Partitioning:** Partition $A^e$ into blocks using a two-phase greedy heuristic to minimize memory usage [S1].
3.  **Palette Creation:** For each block, identify the unique global indices present and create a palette $P$ containing these indices [S1].
4.  **Index Replacement:** Replace each global index $m$ in the block with a local index into $P$ [S1]. The local index requires $\omega$ bits, where $\omega = \lceil \log_2 |P| \rceil$ [S1].
5.  **Directory Construction:** Create the block directory with entries sorted by the starting index of the block in $A^e$ [S1].
6.  **Storage:** Store the final representation as the array of blocks and the array of palettes, allowing palettes to be shared by multiple blocks [S1].

**Memory Formula**
The memory cost for a block $B$ spanning indices $m_i$ to $m_j$ with bit-width $\omega$ is:
$$ \text{MEMORY}(\{m_i, \dots, m_j\}, \omega) = (j - i + 1) \cdot \omega + 2^\omega \cdot \lceil \log_2 \lambda \rceil + \text{size}(\text{directory entry}) $$
Where:
*   $(j - i + 1) \cdot \omega$: Bits for the block indices [S1].
*   $2^\omega \cdot \lceil \log_2 \lambda \rceil$: Bits for the palette using worst-case size $2^\omega$ entries, each $\lceil \log_2 \lambda \rceil$ bits [S1].
*   $\text{size}(\text{directory entry})$: Overhead for the block directory entry [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def palette_compression(Ae, lambda_unique):
    """
    Compresses the indexed attribute array Ae using palette compression.
    
    Args:
        Ae: List of global indices into the material array.
        lambda_unique: Total number of unique attributes (lambda).
        
    Returns:
        blocks: List of blocks, where each block is a list of local indices.
        palettes: List of palettes, where each palette is a list of global indices.
        directory: List of tuples (start_index, palette_index).
    """
    blocks = []
    palettes = []
    directory = []

    def memory_cost(i, j, omega):
        return (j - i + 1) * omega + (1 << omega) * math.ceil(math.log2(lambda_unique)) + 1

    def create_palette(block_indices):
        unique = sorted(set(block_indices))
        if unique in palettes:
            return palettes.index(unique), {v: i for i, v in enumerate(unique)}
        palettes.append(unique)
        return len(palettes) - 1, {v: i for i, v in enumerate(unique)}

    def replace_indices(block_indices, local_map):
        return [local_map[m] for m in block_indices]

    def find_largest_block_with_unique(Ae, start, end, max_unique):
        best_k, best_l, best_len = -1, -1, 0
        for k in range(start, end + 1):
            seen = set()
            for l in range(k, end + 1):
                seen.add(Ae[l])
                if len(seen) > max_unique:
                    break
                if len(seen) == max_unique:
                    length = l - k + 1
                    if length > best_len:
                        best_len = length
                        best_k, best_l = k, l
        return best_k, best_l

    def phase1(i, j):
        if j < i:
            return
        omega = 0
        while omega < 4:
            k, l = find_largest_block_with_unique(Ae, i, j, 1 << omega)
            if k == -1:
                omega += 1
                continue
            if memory_cost(k, l, omega) < (l - k + 1) * (omega + 1):
                palette_idx, local_map = create_palette(Ae[k:l+1])
                blocks.append(replace_indices(Ae[k:l+1], local_map))
                directory.append((k, palette_idx))
                phase1(i, k - 1)
                phase1(l + 1, j)
                return
            omega += 1
        phase2(i, j)

    def phase2(i, j):
        if j < i:
            return
        current = i
        while current <= j:
            best_k, best_cost = -1, float('inf')
            for omega in range(9):
                k, _ = find_largest_block_with_unique(Ae, current, j, 1 << omega)
                if k == -1:
                    continue
                cost_per_entry = memory_cost(current, k, omega) / (k - current + 1)
                if cost_per_entry < best_cost:
                    best_cost = cost_per_entry
                    best_k = k
            if best_k == -1:
                best_k = current
            palette_idx, local_map = create_palette(Ae[current:best_k+1])
            blocks.append(replace_indices(Ae[current:best_k+1], local_map))
            directory.append((current, palette_idx))
            current = best_k + 1

    phase1(0, len(Ae) - 1)
    return blocks, palettes, directory
```

**Edge Cases and Pitfalls**
*   **Combinatorial Complexity:** Finding the optimal block partition is a hard combinatorial problem, so a greedy heuristic is used instead [S1].
*   **Phase 1 Infeasibility:** Phase 1 becomes computationally infeasible for large $\omega$, so it terminates when $\omega \ge 4$ (16 unique indices) [S1].
*   **Precision:** The memory formula uses worst-case palette size ($2^\omega$), which wastes memory if the actual number of unique indices is much smaller [S1].

## Subtopics

### [Palette-based compression for chunked discrete voxel data](3130-palette-based-compression-for-chunked-discrete-voxel-data.md)
Compresses discrete voxel volume chunks by decoupling voxel identity from usage, storing types in a sparse palette and indices in a variable-bit-length buffer. It reduces memory to 1–2 bits per voxel in low-entropy regions and supports thousands of block types, as seen in Minecraft 1.13. It builds on Palette Compression by implementing specific bit-packing and dynamic palette management for chunked data.

## Operating characteristics
*   **Time Complexity:** The block partitioning phase is computationally expensive due to the brute-force search for the largest blocks in Phase 1, while Phase 2 is more efficient but still requires scanning [S1].
*   **Memory Complexity:** Reduces memory footprint significantly for scenes with spatial coherence and low attribute entropy, with overhead limited to the block directory and palettes [S1].
*   **Performance:** Allows fast random access via binary search on the block directory [S1].
*   **Strengths:** High compression ratios for scenes with repeated attributes and spatial coherence [S1].
*   **Limitations:** High compression time due to heuristic search, making it unsuitable for scenes with high attribute entropy or random attribute distribution [S1].

## Background
Palette compression was proposed in "Geometry and Attribute Compression for Voxel Scenes" to address the memory requirements of uncompressed attribute arrays in decoupled SVOs [S1]. It leverages the observation that attribute arrays in voxel scenes often exhibit spatial coherence, leading to consecutive similar indices [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
[S2] Minecraft Wiki: Java Edition 1.13, https://minecraft.wiki/w/Java_Edition_1.13/Level_format, fetched 2026-10-27
