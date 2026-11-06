# Multi-resolution Brick Encoding

Also called: brick encoding, resolution pyramid encoding

## Overview
Multi-resolution Brick Encoding is a lossless compression technique for segmentation volumes that exploits homogeneous label neighborhoods to compactly encode label data and boundaries [S1]. It builds a per-brick resolution pyramid to generate a sequence of operations and a label palette, forming the foundational representation for Compressed Segmentation Volume (CSV) compression [S1, 4985-compressed-segmentation-volume.md]. Unlike flat storage, it separates the label values from the structural assignment logic, enabling efficient entropy coding [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A segmentation volume brick of size $b \times b \times b$ labels, where $b = 2^N$ and $N \in \mathbb{N}$. Input volumes are padded to multiples of $b$ in each dimension [S1].
*   **Output:** A palette of labels (integer values) and a sequence of operations (one per node in the pyramid) that reconstruct the brick [S1].

**Data Structures:**
1.  **Resolution Pyramid:** An explicit octree-like structure with $N+1$ levels, indexed $L_0$ to $L_N$.
    *   Level $L_l$ stores $(b/2^l)^3$ nodes [S1].
    *   Level $L_0$ is the finest, storing $b^3$ labels (the original brick) [S1].
    *   Level $L_N$ is the coarsest, storing $1^3 = 1$ node (the root) [S1].
    *   Each voxel in $L_l$ overlaps $2^3=8$ voxels in $L_{l-1}$ [S1].
2.  **Palette:** A list of unique or duplicate label integers [S1].
3.  **Operation Sequence:** A list of commands corresponding to the traversal order of the pyramid nodes [S1].

**Parameters:**
*   $b$: Brick side length, a power of 2.
*   $N = \log_2 b$: Number of coarsening steps.

**Algorithm Steps:**

1.  **Build Pyramid:** Construct the resolution pyramid explicitly.
    *   Initialize $L_0$ with the input brick labels [S1].
    *   For each level $l$ from 1 to $N$:
        *   For each node in $L_l$, identify the corresponding $2 \times 2 \times 2$ block of 8 voxels in $L_{l-1}$ [S1].
        *   Assign the label of the node in $L_l$ to be the **most frequent label** among those 8 voxels [S1].
    *   *Note:* This step requires approximately 14% additional temporary memory for the explicit pyramid storage [S1].

2.  **Traverse and Encode:** Process the pyramid nodes to generate the operation sequence and palette.
    *   **Traversal Order:** Start from the root node (coarsest level $L_N$). Process nodes coarse-to-fine (from $L_N$ down to $L_0$). Within each level, process nodes along the Morton Z-curve [S1].
    *   **Operation Generation:** For each node in the traversal order, determine the operation that assigns its label. The operations define how labels are assigned, such as reading from the palette or reusing a parent/neighbor label [S1].
        *   **Root Node ($L_N$):** The label of the root is the first entry in the palette. The operation is typically "Read Palette" (or similar, referencing the first palette index) [S1].
        *   **Subsequent Nodes:** The specific operation depends on the label assignment strategy. Common operations include:
            *   $P_0$: Read a new label from the palette.
            *   $R_p$: Reuse the label of the parent node.
            *   $R_{\{x,y,z\}}$: Reuse the label of a specific neighbor (e.g., x-neighbor) [S1].
            *   $P_\delta$: (Specific variant mentioned in source, likely related to palette or delta encoding) [S1].
    *   **Palette Management:** If an operation requires a new label (e.g., $P_0$), append that label to the palette and record its index in the operation sequence [S1].

3.  **Output:** Return the final palette and the sequence of operations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_brick_encoding(brick: List[List[List[int]]], b: int) -> tuple:
    """
    Encodes a brick of size bxbxb using multi-resolution brick encoding.
    brick: 3D list of integers (labels).
    b: Side length of the brick (power of 2).
    Returns: (palette: List[int], operations: List[Operation])
    """
    N = int(math.log2(b))
    
    # Step 1: Build Resolution Pyramid
    # pyramid[l] is a 3D list of labels at level l
    # Level 0 is the original brick
    pyramid = [None] * (N + 1)
    pyramid[0] = copy.deepcopy(brick)
    
    for l in range(1, N + 1):
        coarse_size = b // (2 ** l)
        fine_size = b // (2 ** (l - 1))
        pyramid[l] = [[ [0] * coarse_size for _ in range(coarse_size) ] for _ in range(coarse_size)]
        
        for z in range(coarse_size):
            for y in range(coarse_size):
                for x in range(coarse_size):
                    # Find the 8 voxels in the previous level
                    # The voxel at (x,y,z) in level l covers
                    # x*2 to x*2+1, y*2 to y*2+1, z*2 to z*2+1 in level l-1
                    base_x = x * 2
                    base_y = y * 2
                    base_z = z * 2
                    
                    # Collect labels from the 8 neighbors in l-1
                    labels = []
                    for dz in range(2):
                        for dy in range(2):
                            for dx in range(2):
                                labels.append(pyramid[l-1][base_z + dz][base_y + dy][base_x + dx])
                    
                    # Assign most frequent label
                    pyramid[l][z][y][x] = get_most_frequent_label(labels)
    
    # Step 2: Traverse and Generate Operations
    palette = []
    operations = []
    
    # Helper to get Morton index for a node at level l, position (z,y,x)
    # Morton index interleaves bits of z, y, x
    def get_morton_index(l, z, y, x):
        # Implementation of Morton code for 3D coordinates
        # Since we process coarse-to-fine, we can just generate indices
        # and sort them, or generate them in order.
        # For simplicity, we assume a function that returns the Morton index
        # for the given level and coordinates.
        pass 

    # We need to traverse all nodes from L_N down to L_0
    # Within each level, sort by Morton Z-curve
    
    all_nodes = []
    for l in range(N, -1, -1):
        level_size = b // (2 ** l)
        for z in range(level_size):
            for y in range(level_size):
                for x in range(level_size):
                    # Calculate Morton index for sorting
                    # Note: Morton index usually for the finest level, 
                    # but we can map (l, z, y, x) to a unique Morton-like key
                    # or simply generate the Morton index for the finest level 
                    # corresponding to the top-left corner of the coarse node.
                    # The source says "along the Morton Z-curve within a level".
                    # This implies sorting nodes at level l by their spatial position
                    # mapped to the Z-curve of that level's grid.
                    index = get_morton_index(l, z, y, x)
                    all_nodes.append((l, z, y, x, index))
    
    # Sort by Morton index
    all_nodes.sort(key=lambda node: node[4])
    
    # Process nodes
    for l, z, y, x, _ in all_nodes:
        label = pyramid[l][z][y][x]
        
        # Determine operation
        # The specific logic for choosing between P0, Rp, R_neighbor depends on
        # the implementation details not fully specified in the source text
        # beyond "reading from palette or reusing parent/neighbors".
        # We assume a greedy approach or a predefined heuristic.
        
        op = None
        if l == N:
            # Root node
            if not palette or palette[0] != label:
                palette.append(label)
                op = Operation.ReadPalette(index=0)
            else:
                op = Operation.ReadPalette(index=0)
        else:
            # Check if label matches parent
            parent_l = l + 1
            parent_z = z // 2
            parent_y = y // 2
            parent_x = x // 2
            parent_label = pyramid[parent_l][parent_z][parent_y][parent_x]
            
            if label == parent_label:
                op = Operation.ReuseParent()
            else:
                # Check neighbors? (Source mentions R{x,y,z})
                # For simplicity, if not parent, read from palette
                # In a real implementation, neighbor checks would happen here
                if label in palette:
                    op = Operation.ReadPalette(index=palette.index(label))
                else:
                    palette.append(label)
                    op = Operation.ReadPalette(index=len(palette) - 1)
        
        operations.append(op)
        
    return palette, operations

def get_most_frequent_label(labels: List[int]) -> int:
    counts = {}
    for l in labels:
        counts[l] = counts.get(l, 0) + 1
    return max(counts, key=counts.get)
```

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** Generating the pyramid explicitly requires about 14% additional temporary memory [S1].
*   **Padding:** Input data must be padded to multiples of $b$ in each dimension [S1].
*   **Decoding Memory:** Decoding does not require more memory than necessary to store the $(b/2^l)^3$ labels of the desired level-of-detail [S1].

## Subtopics

### Compressed Segmentation Volume (CSV)
The parent method that uses this encoding as its core representation, applying rANS entropy coding to the operation sequence [4985-compressed-segmentation-volume.md].

### rANS Entropy Coding
A secondary compression step applied to the operation sequence generated by this method, leveraging global frequency tables for parallel efficiency [S1].

## Operating characteristics
*   **Time Complexity:** Encoding involves building the pyramid ($O(b^3)$) and traversing it ($O(b^3)$). The source reports high throughput, up to gigabytes per second for both compression and decompression [S1].
*   **Memory Complexity:** Encoding requires $O(b^3)$ for the input brick and $O(b^3)$ for the explicit pyramid (with ~14% overhead). Decoding memory is proportional to the output resolution level [S1].
*   **Compression Ratio:** Achieves strong compression ratios of about 1% to 3% of the original data set size [S1].
*   **Strengths:** Lossless, high throughput, suitable for GPU-based on-the-fly rendering, lightweight [S1].
*   **Limitations:** Requires input padding to power-of-2 brick sizes. Explicit pyramid generation adds temporary memory overhead [S1].

## Background
Multi-resolution Brick Encoding was proposed as part of a lossless compression technique for segmented volumes in a paper published in IEEE Transactions on Visualization and Computer Graphics in 2023 [S1]. It addresses the challenges of compressing segmentation volumes for scientific visualization, particularly for GPU-based rendering [S1]. The method exploits the presence of homogeneous label neighborhoods and effectively encodes boundaries between them [S1].

## Sources
[S1] Fast Compressed Segmentation Volumes for Scientific Visualization., https://doi.org/10.1109/tvcg.2023.3326573, fetched 2026-10-26
