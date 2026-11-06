# Variable-Block Color Compression
## Overview
Variable-Block Color Compression is a lossy compression method for sparse 3D color data that groups voxels into variable-length blocks to optimize the compression ratio [S1, unofficial]. It sets itself apart from fixed-size block methods by allowing block boundaries to adapt to local color similarity, thereby reducing the number of headers required for large homogeneous regions while maintaining a strict error threshold [S1, unofficial].

## How it works
**Inputs:** A sequence of voxel colors; an error threshold $\epsilon$ (Mean Square Error) [S1, unofficial].
**Outputs:** A block header array and a weight array [S1, unofficial].

**Data Structures:**
1. **Block Header Array:** A 32-bit integer array where each header occupies two consecutive 32-bit words (64 bits total) [S1, unofficial]. Each header contains Base Color $B_1$ (16 bits), Base Color $B_2$ (16 bits), and Start Voxel Index (32 bits) [S1, unofficial].
2. **Weight Array:** An array of weights, each stored with a fixed bit width (3 bits) [S1, unofficial].

**Parameters:**
* $B_1, B_2$: Base colors, 16 bits each [S1, unofficial].
* $w$: Weight, 3 bits [S1, unofficial].
* $\epsilon$: Error threshold (MSE) [S1, unofficial].

**Algorithm:**

**Encoding:**
1. Initialize each voxel color as a separate block [S1, unofficial].
2. Sequentially iterate through blocks and attempt to merge each block with its left or right neighbor, whichever yields the lowest Mean Square Error (MSE) after merging [S1, unofficial].
3. If both potential merges violate the error threshold $\epsilon$, the block is considered done [S1, unofficial]. This involves finding new base colors and weights for the union of the merged blocks [S1, unofficial].
4. Repeat the iteration with newly merged blocks until all blocks are considered done [S1, unofficial].
5. Store the resulting blocks' base colors and start voxel indices in the Header Array [S1, unofficial].
6. Store the corresponding weights in the Weight Array [S1, unofficial].

**Decoding:**
1. Find the voxel index (using leaf counters) [S1, unofficial].
2. Perform a binary search on the headers' first voxel index to find the block spanning the color [S1, unofficial].
3. Perform a direct lookup in the weight array using the same index [S1, unofficial].
4. Reconstruct the color using the base colors (from header) and the weight [S1, unofficial].

**Formula:**
$$ C = wB_1 + (1 - w)B_2 $$
where $C$ is the reconstructed color, $B_1$ and $B_2$ are base colors, and $w$ is the weight [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_colors(colors, epsilon):
    """
    Encodes a list of colors into variable-block compression format.
    colors: List of color objects (e.g., tuples or vectors).
    epsilon: Maximum allowed Mean Square Error for a block.
    Returns: headers (list of dicts), weights (list of floats)
    """
    # Step 1: Each color is initially considered a separate block.
    blocks = []
    for i, c in enumerate(colors):
        blocks.append({
            'start': i,
            'end': i,
            'colors': [c],
            'base_colors': None,
            'weights': None
        })

    # Step 4: Repeat iteration until all blocks are done.
    changed = True
    while changed:
        changed = False
        i = 0
        while i < len(blocks) - 1:
            current = blocks[i]
            neighbor = blocks[i + 1]
            
            # Try merging current with neighbor
            merged_colors = current['colors'] + neighbor['colors']
            
            # [G] Standard optimization for 2-base color compression.
            # Iteratively refines B1, B2, and per-voxel weights to minimize MSE.
            merged_base, merged_weights = _optimize_block(merged_colors)
            
            # Calculate MSE of the merged block
            mse = _compute_mse(merged_colors, merged_base, merged_weights)
            
            if mse <= epsilon:
                # Merge successful
                current['end'] = neighbor['end']
                current['colors'] = merged_colors
                current['base_colors'] = merged_base
                current['weights'] = merged_weights
                
                # Remove the neighbor block
                blocks.pop(i + 1)
                changed = True
                # Do not increment i, as the current block might merge further
            else:
                # Merge failed, move to next block
                i += 1
        
        # Handle the last block if it wasn't merged
        if len(blocks) > 0:
            last = blocks[-1]
            if last['base_colors'] is None:
                last['base_colors'], last['weights'] = _optimize_block(last['colors'])

    # Step 5 & 6: Store headers and weights
    headers = []
    weights = []
    
    for block in blocks:
        headers.append({
            'B1': block['base_colors'][0],
            'B2': block['base_colors'][1],
            'StartVoxel': block['start']
        })
        # Weights are stored per-voxel in the weight array
        for j in range(block['start'], block['end'] + 1):
            weights.append(block['weights'][j - block['start']])
            
    return headers, weights

def decode_color(header, weight):
    """
    Reconstructs a color from a header and a weight.
    """
    B1 = header['B1']
    B2 = header['B2']
    w = weight
    # Formula: C = w*B1 + (1-w)*B2
    return w * B1 + (1 - w) * B2

def find_block_for_voxel(headers, voxel_index):
    """
    Binary search for the block containing the voxel index.
    """
    low = 0
    high = len(headers) - 1
    
    while low <= high:
        mid = (low + high) // 2
        start = headers[mid]['StartVoxel']
        
        # Determine block end from next header's start
        if mid == len(headers) - 1:
            end = float('inf')
        else:
            end = headers[mid + 1]['StartVoxel'] - 1
            
        if start <= voxel_index <= end:
            return headers[mid]
        elif voxel_index < start:
            high = mid - 1
        else:
            low = mid + 1
            
    return None

# [G] Standard iterative least-squares optimization for 2-base color compression.
def _optimize_block(colors):
    # Initialize B1, B2 as first two colors
    B1 = colors[0]
    B2 = colors[1]
    weights = [0.5] * len(colors)
    
    # Iterate to minimize MSE
    for _ in range(10):
        # Update weights for fixed B1, B2
        for i, c in enumerate(colors):
            # Solve for w in C = w*B1 + (1-w)*B2 using least squares projection
            diff = c - B2
            dot_B1_B2 = sum((B1[k] - B2[k]) ** 2 for k in range(len(B1)))
            if dot_B1_B2 > 1e-9:
                w = sum(diff[k] * (B1[k] - B2[k]) for k in range(len(B1))) / dot_B1_B2
                weights[i] = max(0.0, min(1.0, w))
        
        # Update B1, B2 for fixed weights
        sum_w = sum(weights)
        sum_wB1 = sum(w * c for w, c in zip(weights, colors))
        sum_wB2 = sum((1 - w) * c for w, c in zip(weights, colors))
        B1 = [sum_wB1[k] / sum_w for k in range(len(B1))]
        B2 = [sum_wB2[k] / (len(colors) - sum_w) for k in range(len(B2))]
        
    return B1, B2, weights

def _compute_mse(colors, base_colors, weights):
    B1, B2 = base_colors
    mse_sum = 0.0
    for c, w in zip(colors, weights):
        recon = [w * B1[k] + (1 - w) * B2[k] for k in range(len(B1))]
        mse_sum += sum((c[k] - recon[k]) ** 2 for k in range(len(B1)))
    return mse_sum / len(colors)
```

## Subtopics

### [Variable Bit Rate Color Compression with Macro Blocks](1886-variable-bit-rate-color-compression-with-macro-blocks.md)
Introduces variable bits per weight and macro blocks to reduce header overhead. It sets itself apart from its sibling [Variable-Block Color Compression](1878-variable-block-color-compression.md) by localizing header offsets, thereby reducing the bit-width required for pointers and enabling efficient variable bit-rate encoding without prohibitive header sizes.

## Operating characteristics
* **Time Complexity:** Encoding is iterative and depends on the number of merges, processing all blocks per pass [S1, unofficial]. Decoding requires a binary search on headers ($O(\log N)$) and a direct lookup in weights ($O(1)$) [S1, unofficial].
* **Memory Complexity:** Storage consists of fixed-size headers (64 bits per block) and a weight array (3 bits per voxel) [S1, unofficial]. The number of blocks is variable, depending on the error threshold [S1, unofficial].
* **Strengths:** Outperforms previous attempts in compression/quality for sparse 3D data [S1, unofficial]. Memory overhead is practically negligible compared to previous methods that almost doubled the size of the DAG [S1, unofficial].
* **Limitations:** Lookup times are a bit more expensive than fixed-block methods due to the binary search [S1, unofficial].
* **When to use:** When sparse 3D color data requires high compression ratios with controlled error, and memory overhead must be minimized [S1, unofficial].

## Background
Originated in the context of sparse voxel DAGs and memory efficient compression of surface representations [S1, unofficial]. The method was introduced as an alternative to mapping attributes to a DAG, aiming to reduce memory overhead while maintaining compression quality [S1, unofficial].

## Sources
[S1] PDF On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
