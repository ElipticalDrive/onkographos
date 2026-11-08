# Compressed Segmentation Volume

Also called: Brick-based Iterative Refinement, CSVD, CSVE, CSVol, Fast Compressed Segmentation Volumes, Iterative Refinement Compression for Segmentation Volumes

## Overview
Compressed Segmentation Volume (CSV) is a lossless compression technique for voxel-based segmentation volumes that encodes individual bricks using a multi-resolution label pyramid and entropy-coded operation sequences [S1]. It solves the storage and transfer bottlenecks of large segmentation datasets by achieving compression ratios of 1–3% of the original data size while supporting fast, parallel decompression and GPU-based level-of-detail (LoD) rendering [S1]. Unlike general-purpose compressors (e.g., HDF5, PNG) or specialized segmentation tools (e.g., Compresso), CSV exploits the local homogeneity of label regions through parent/neighbor reuse and palette back-references, enabling efficient on-the-fly decompression without full volume reconstruction [S1].

## How it works
**Inputs and Outputs:**
- Input: A segmentation volume (array of `uint32_t` labels) partitioned into cubic bricks of size $b \times b \times b$ (where $b$ is a power of two, typically 16, 32, or 64) [S1][S2, unofficial].
- Output: A compressed representation consisting of a global palette of labels and an encoding array (`m_encoding`) containing rANS-encoded operation streams for each brick, along with a start index array (`m_brick_starts`) [S2, unofficial].

**Data Structures:**
- `MultiGridNode`: Contains `uint32_t label` and `bool constant_subregion` [S2, unofficial].
- `CompressedSegmentationVolume` class stores:
  - `m_encoding`: `std::vector<uint32_t>` containing compressed entries for all LoDs of all bricks [S2, unofficial].
  - `m_brick_starts`: `std::vector<uint32_t>` storing the start index of each brick in `m_encoding` [S2, unofficial].
  - `m_volume_dim`: `glm::uvec3` defining the volume dimensions [S2, unofficial].
  - `m_brick_size`: `uint32_t` defining the brick side length [S2, unofficial].
  - `m_use_rANS`: `bool` flag for rANS usage [S2, unofficial].
  - `m_use_detail_freq`: `bool` flag for separate frequency tables for leaf nodes [S2, unofficial].
- Neighbor lookup: `static constexpr const glm::ivec3 neighbor[][3]` with 8 outer indices, each containing 3 `glm::ivec3` vectors for neighbor directions in $[-1, 1]^3$ [S2, unofficial].

**Algorithm Steps:**

1. **Multi-Resolution Pyramid Construction (Per Brick):**
   - Build a resolution pyramid for the brick. Level $L_l$ stores $(b/2^l)^3$ labels. Each voxel in $L_l$ overlaps $2^3$ voxels in $L_{l-1}$ [S1].
   - The label of a voxel in $L_l$ is the most frequent label among its 8 children in $L_{l-1}$. If all valid children have the same label and are constant, flag the node as `constant_subregion = true` [S2, unofficial].
   - Voxels outside the volume boundary are flagged with label `0xFFFFFFFF` and skipped during frequency calculation [S2, unofficial].
   - Ambiguity in most frequent label resolution: choose the label occurring first in the child node array [S1].
   - Temporary memory overhead is ~14% of the brick size [S1].

2. **Operation Stream Generation (Per Brick):**
   - Initialize palette with the root node's label (index $i_p = 0$) [S1].
   - Iterate levels $l$ from $N$ (coarsest) down to 1. For each node $i$ on level $l$ (spacing $2^l$) in Morton Z-curve order:
     - If `pyramid[i].constantChildren` is true, skip [S1].
     - Set `parent = pyramid[i].label` [S1].
     - Iterate child nodes $j$ (spacing $2^{l-1}$) in Z-order:
       - Get label $L = \text{pyramid}[j].\text{label}$ and stop bit $stop = \text{pyramid}[j].\text{constantChildren}$ [S1].
       - Determine operation `op` using `bestOperation(parent, pyramid, palette, L)` [S1]. The fixed selection order is: Parent Reuse ($R_p$), Neighbor Reuse ($R_x, R_y, R_z$), Palette Back Reference ($P^*0$), Palette Back Reference ($P^*\delta$), Palette Advance ($P_a$) [S1].
       - If `op` is $P_a$, push $L$ to palette. If `op` is $P^*\delta$, output $\delta$ (4-bit value) [S1].
       - Output tuple `(op, stop)` [S1]. Each operation occupies at least 4 bits [S1].

3. **Entropy Coding (rANS):**
   - Perform a prepass over a subset of bricks (e.g., every 512th or 4096th) to compute global frequency tables [S1].
   - Two frequency tables are created: one for interior nodes (levels $L_N \dots L_1$) and one for leaf nodes ($L_0$, where stop bits are always 0) [S1].
   - Normalize frequencies: set any zero frequency to 1 to prevent undefined rANS behavior [S2, unofficial].
   - Encode the operation stream using rANS [S1].

4. **Decoding (Per Brick, Target LoD $t$):**
   - Initialize palette read index $i_p = 0$. Initialize output array `out[0...2^{3(N-t)}]` to empty. Set `out[0] = palette[i_p]` [S1].
   - Iterate levels $l$ from $N$ down to $t+1$:
     - Iterate all already decoded nodes on level $l$ (spacing $2^{l-t}$) in Morton Z-curve order [S1].
     - If `out[lastChildOf(i)]` is not empty (constant area), continue [S1].
     - Set `parent = out[i]` [S1].
     - Iterate child nodes on level $l$ (spacing $2^{l-t-1}$) in Z-order:
       - Get child index $j$ [S1].
       - Read next operation and stop bit: `(op, stop) ← readNextOperationAndStopBit()` [S1].
       - Apply operation:
         - $R_p$: `out[j] = parent` [S1].
         - $R_{x,y,z}$: `out[j] = neighbor value`. If neighbor has later Z-index (not decoded), reinterpret as reference to neighbor's parent [S1]. If neighbor does not exist at current LOD, round index down to parent element [S2, unofficial].
         - $P_a$: `out[j] = palette[++i_p]` [S1].
         - $P^*0$: `out[j] = palette[i_p]` [S1].
         - $P^*\delta$: `out[j] = palette[i_p - \delta]` [S1].
       - If `stop` is true, fill $j$'s entire sub-block of $2^{3(l-t-1)}$ labels in `out` with the label assigned to $j$ [S1].
   - In-place decoding: Coarser LODs are overwritten until target LOD $t$ is reached [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_brick(brick_voxels, brick_size):
    # 1. Build Multi-Resolution Pyramid
    # Levels 0 to N where N = log2(brick_size)
    # L0 is the original brick. Ll stores (brick_size / 2^l)^3 labels.
    pyramid = []
    current_level = brick_voxels # L0
    for l in range(0, int(math.log2(brick_size)) + 1):
        pyramid.append(current_level)
        if l == int(math.log2(brick_size)):
            break
        next_level = []
        step = 2 ** l
        for z in range(0, brick_size, step):
            for y in range(0, brick_size, step):
                for x in range(0, brick_size, step):
                    # Gather 8 children
                    children = []
                    for dz in range(0, 2):
                        for dy in range(0, 2):
                            for dx in range(0, 2):
                                val = current_level[z + dz][y + dy][x + dx]
                                if val != 0xFFFFFFFF: # Skip out-of-bounds
                                    children.append(val)
                    
                    # Find most frequent label
                    if not children:
                        label = 0xFFFFFFFF
                        constant = False
                    else:
                        counts = {}
                        for c in children:
                            counts[c] = counts.get(c, 0) + 1
                        max_count = max(counts.values())
                        # Ambiguity resolution: first occurring
                        label = None
                        for c in children:
                            if counts[c] == max_count:
                                label = c
                                break
                        constant = (len(set(children)) == 1)
                    
                    next_level.append({'label': label, 'constant': constant})
        current_level = next_level

    # 2. Generate Operation Stream
    palette = [pyramid[-1][0]['label']] # Root label
    operations = []
    
    N = int(math.log2(brick_size))
    for l in range(N, 0, -1):
        level_nodes = pyramid[l]
        child_step = 2 ** (l - 1)
        parent_step = 2 ** l
        
        # Iterate in Morton Z-order (simplified here as linear for logic)
        for i in range(len(level_nodes)):
            node = level_nodes[i]
            if node['constant']:
                continue
            
            parent_label = node['label']
            
            # Iterate children
            for j in range(8):
                child_idx = i * 8 + j
                if child_idx >= len(pyramid[l-1]):
                    break
                child_node = pyramid[l-1][child_idx]
                child_label = child_node['label']
                stop = child_node['constant']
                
                # Determine best operation
                op = determine_best_operation(parent_label, child_label, palette, operations)
                
                if op == 'Pa':
                    palette.append(child_label)
                elif op == 'P*_delta':
                    # delta is encoded separately
                    pass 
                
                operations.append((op, stop))
                
    return palette, operations

def determine_best_operation(parent, child, palette, history):
    # Fixed order: Rp, Rx, Ry, Rz, P*0, P*delta, Pa
    if child == parent:
        return 'Rp'
    
    # Neighbor reuse logic would check spatial neighbors in the decoded grid
    # For simplicity, assume neighbor check returns None if not applicable
    neighbor_op = check_neighbor_reuse(parent, child, history)
    if neighbor_op:
        return neighbor_op
        
    if child == palette[-1]:
        return 'P*0'
    
    # Check P*delta (last k entries)
    for k in range(1, 16):
        if palette[-k-1] == child:
            return ('P*_delta', k)
            
    return 'Pa'

def decompress_brick(palette, operations, target_lod, brick_size):
    N = int(math.log2(brick_size))
    t = target_lod
    output_size = 2 ** (3 * (N - t))
    out = [None] * output_size
    
    ip = 0
    out[0] = palette[ip]
    
    # In-place decoding
    for l in range(N, t, -1):
        # Spacing logic for in-place array
        # This part requires precise index mapping based on Morton order
        # and spacing 2^(l-t)
        pass 
        # Detailed index mapping omitted for brevity, follows [S1] Algorithm 1
```

**Parameters:**
- `brick_size` ($b$): Power of two, typically 16, 32, or 64. Larger bricks yield better compression but reduce parallelization efficiency [S1].
- `use_rANS`: Boolean, default `true`. Enables rANS entropy coding [S2, unofficial].
- `use_detail_freq`: Boolean, default `true`. Uses separate frequency table for leaf nodes [S2, unofficial].
- `subsampling_factor`: Integer > 1, default 8. Subsamples bricks during frequency table computation [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Neighbor Reuse Ambiguity:** If a neighbor reference points to a node with a later Z-index (not yet decoded), it is reinterpreted as a reference to the neighbor's parent [S1].
- **Zero Frequencies:** If subsampling misses operations, frequencies are clamped to 1 to prevent rANS division by zero [S2, unofficial].
- **Boundary Handling:** Voxels outside the volume are labeled `0xFFFFFFFF` and skipped during coarsening [S2, unofficial].
- **Parallelization:** Naive parallelization (T threads for T bricks) leads to suboptimal utilization for large bricks due to divergent processing times [S1].

## Subtopics

### [Multi-resolution Brick Encoding](5775-multi-resolution-brick-encoding.md)
Multi-resolution Brick Encoding is a lossless compression technique for segmentation volumes that exploits homogeneous label neighborhoods to compactly encode label data and boundaries. It builds a per-brick resolution pyramid to generate a sequence of operations and a label palette, forming the foundational representation for Compressed Segmentation Volume (CSV) compression. Unlike flat storage, it separates the label values from the structural assignment logic, enabling efficient entropy coding.

### Segmented Dataset Quantization
A method for segmented datasets where small blocks are quantized to have only one or two different components inside. Encoded data consists of the block type and the splitting plane which subdivides the block [S4].

## Operating characteristics
**Time and Memory Complexity:**
- Compression and decompression throughput is high, up to gigabytes per second [S1][S3, unofficial].
- Compression speed is 1.5 to 3 GB/s, with $b=32$ achieving highest throughput despite lower CPU core utilization [S1].
- Temporary memory overhead for pyramid generation is ~14% of the brick size [S1].
- In-place decoding overwrites coarser LODs, increasing memory consumption during decoding but not strictly required for correctness [S3, unofficial].

**Measured Performance:**
- Compression Ratio (CR) for CELLS (4.0GB, 1M labels, 1000³ voxels): CSV ($b=64$, no rANS) is 6.428%, CSV ($b=64$, with rANS) is 2.805% [S1].
- Compression Ratio (CR) for FIBER (5.7GB, 32K labels, 1579×1092×1651 voxels): CSV ($b=64$, no rANS) is 2.502%, CSV ($b=64$, with rANS) is 0.899% [S1].
- Compression Ratio (CR) for CORTEX (927.7GB, 15M labels, 6144×9216×4096 voxels): Representative 1024³ subvolume: CSV ($b=64$, no rANS) is 3.564%, CSV ($b=64$, with rANS) is 1.590% [S1].
- Worst-case CRs ($b=64$): CELLS 3.794%, FIBER 2.360%, CORTEX 4.436% [S1].

**Scaling and Trade-offs:**
- rANS improves CR by ~40% compared to 4-bit nibble storage, with only ~2.5% impact on throughput [S1].
- Larger bricks ($b=64$) yield better compression but suffer from less optimal parallelization utilization [S1].
- Hilbert curves yield similar CR to Morton Z-curves but take 2–3 times longer to compress [S1].
- CSV is 8–30x faster than Compresso with LZMA while achieving similar CRs (and significantly better CR for FIBER) [S1].

**When to Use/Not Use:**
- Use for large segmentation volumes requiring GPU-based visualization with on-the-fly decompression and LoD [S1][S3, unofficial].
- Not suitable for quantitative scalar data where wavelet or run-length encoding is more effective [S3, unofficial].

## Background
Originated in the paper "Fast Compressed Segmentation Volumes for Scientific Visualization" (2023) [S1]. It addresses the challenges of storing and visualizing voxel-based segmentation volumes with large numbers of labels and voxels [S1]. The method builds on the principles of brick-based hierarchical storage, utilizing multi-resolution pyramids and entropy coding [S1]. It is used in GPU-based volume visualization, supporting level-of-detail rendering and optional on-demand streaming of detail coefficients [S1].

## Sources
[S1] Fast Compressed Segmentation Volumes for Scientific Visualization., https://doi.org/10.1109/tvcg.2023.3326573, fetched 2026-10-26
[S2] max-pio/compressed-segmentation-volumes, https://github.com/max-pio/compressed-segmentation-volumes, fetched 2026-10-26 (unofficial)
[S3] (PDF) Lossless Compression of Volume Data - ResearchGate, https://www.researchgate.net/publication/221252510_Lossless_Compression_of_Volume_Data, fetched 2026-11-03 (unofficial)
[S4] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
