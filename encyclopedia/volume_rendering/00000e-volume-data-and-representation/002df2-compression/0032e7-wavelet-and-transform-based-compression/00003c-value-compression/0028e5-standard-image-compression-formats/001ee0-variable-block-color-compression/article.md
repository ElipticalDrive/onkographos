# Variable-Block Color Compression

Also called: VBC Compression

## Overview
Variable-Block Color Compression is a lossy compression method for sparse 3D color data that partitions a sequence of voxels into variable-length blocks, encoding each block with two base colors and a single interpolation weight [S1, unofficial]. It solves the problem of high memory overhead in sparse voxel representations by reducing the number of headers required through adaptive block merging, while maintaining a fixed-rate header structure for efficient decoding [S1, unofficial]. It sets itself apart from its sibling [ASTC](5729-astc.md) by using variable-length spatial blocks rather than fixed-size 2D/3D texture blocks, and from its parent [Standard Image Compression Formats](4394-standard-image-compression-formats.md) by optimizing for luminance error in sparse 3D data rather than fixed-rate 2D image compression [S1, unofficial].

## How it works
**Inputs:** An array of 3D voxel colors $C_0, C_1, \dots, C_{N-1}$.
**Outputs:** A list of block headers and a list of weights.

**Data Structures:**
1.  **Block Header:** Stored in a 32-bit array. Contains:
    *   Base Color 1 ($B_1$): 16 bits per channel (RGB), total 48 bits.
    *   Base Color 2 ($B_2$): 16 bits per channel (RGB), total 48 bits.
    *   Start Voxel Index: 32 bits.
    *   *Note:* The sources describe the header as containing $B_1, B_2$ and the Start Voxel Index, totaling $2 \times 16 + 32$ bits per channel? No, the text says "2x16 + 32 bits" for the header in a 32-bit array context, but later specifies 16-bit base colors. The layout description in [S1, unofficial] states: "The base colors and the first voxel index is stored together (2 × 16 + 32 bits) in which we call the header". This implies 16-bit precision for base colors.
2.  **Weights Array:** A separate array storing the weight $w$ for each block. The weight has a fixed bit width (e.g., 3 bits or 4 bits depending on implementation, [S1, unofficial] mentions 3 bits in text but 4-bit indices in [S2, unofficial] context for BC6H, but for VBC specifically, [S1, unofficial] says "per-color weight (e.g., 3 bits)").

**Parameters:**
*   **Error Threshold:** The maximum allowed Mean Square Error (MSE) for a block. If merging two blocks exceeds this threshold, the merge is rejected [S1, unofficial].
*   **Base Colors ($B_1, B_2$):** 16-bit floating point or fixed-point per channel [S1, unofficial].
*   **Weight ($w$):** Scalar value used for interpolation [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Treat each voxel color as an individual block.
2.  **Merging Phase:**
    *   Iterate through the blocks sequentially.
    *   For each block, attempt to merge it with its left neighbor and its right neighbor.
    *   Calculate the MSE for the union of the block and the neighbor.
    *   Select the merge (left or right) that results in the lowest MSE.
    *   If the lowest MSE is within the Error Threshold, perform the merge. This involves computing new base colors and weights for the combined set of voxels.
    *   If both merges violate the error threshold, the block is marked as "done" and is not merged further in this pass.
3.  **Iteration:**
    *   Repeat the merging phase with the newly formed blocks.
    *   Continue until no more merges can be performed that satisfy the error threshold.
4.  **Endpoint Refinement (Optional):**
    *   If `INSET_COLOR_BBOX` is defined, refine the bounding box of the block in log2 RGB space to tighten the endpoints [S2, unofficial].
    *   If `OPTIMIZE_ENDPOINTS` is defined, optimize endpoints via least squares [S2, unofficial].
5.  **Quantization:**
    *   Quantize the refined base colors to 10-bit precision (or 16-bit as per [S1, unofficial] header definition, see discrepancy below) [S2, unofficial].
    *   Quantize the weight to the target bit width (e.g., 3 bits) [S1, unofficial].
6.  **Encoding:**
    *   Pack the header (Base Colors + Start Voxel Index) into the header array.
    *   Pack the weight into the weights array.

**Decoding:**
1.  **Find Block:** Given a voxel index, perform a binary search on the headers' Start Voxel Index to find the block spanning that index [S1, unofficial].
2.  **Lookup Weight:** Use the voxel index (relative to the block start) to look up the weight in the weights array [S1, unofficial].
3.  **Reconstruct Color:**
    *   Retrieve Base Colors $B_1$ and $B_2$ from the header.
    *   Calculate the reconstructed color $C$ using linear interpolation:
        $$ C = w B_1 + (1 - w) B_2 $$
        where $w$ is the weight [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_mse(colors, base1, base2, weight):
    """Calculate Mean Square Error for a list of colors given base colors and weight."""
    if not colors:
        return 0.0
    mse_sum = 0.0
    for c in colors:
        # Reconstruct color
        reconstructed = weight * base1 + (1.0 - weight) * base2
        # MSE calculation (Euclidean distance squared)
        diff = [c[i] - reconstructed[i] for i in range(3)]
        mse_sum += sum(d * d for d in diff)
    return mse_sum / len(colors)

def find_optimal_base_colors_and_weight(colors):
    """Find optimal two base colors and weight for a set of colors."""
    if not colors:
        return None, None, None
    
    # Simple k-means with k=2 to find base colors
    # Initialize with min and max
    c1 = min(colors, key=lambda x: x[0]) # Simplified init
    c2 = max(colors, key=lambda x: x[0])
    
    for _ in range(10): # Standard iteration count [G]
        group1 = [c for c in colors if sum((c[i] - c1[i])**2 for i in range(3)) < sum((c[i] - c2[i])**2 for i in range(3))]
        group2 = [c for c in colors if sum((c[i] - c1[i])**2 for i in range(3)) >= sum((c[i] - c2[i])**2 for i in range(3))]
        
        if not group1 or not group2:
            break
            
        c1 = [sum(c[i] for c in group1) / len(group1) for i in range(3)]
        c2 = [sum(c[i] for c in group2) / len(group2) for i in range(3)]
        
    # Find optimal weight
    # Minimize sum((c - (w*b1 + (1-w)*b2))^2)
    # Derivative w.r.t w: sum(2*(c - (w*b1 + (1-w)*b2)) * (b1 - b2)) = 0
    # Let diff = b1 - b2
    # sum((c - b2 - w*diff) * diff) = 0
    # sum((c - b2)*diff) - w * sum(diff*diff) = 0
    # w = sum((c - b2)*diff) / sum(diff*diff)
    
    diff = [c1[i] - c2[i] for i in range(3)]
    diff_sq_sum = sum(d * d for d in diff)
    
    if diff_sq_sum == 0:
        return c1, c2, 0.5
        
    numerator = sum(sum((colors[k][i] - c2[i]) * diff[i] for i in range(3)) for k in range(len(colors)))
    weight = numerator / diff_sq_sum
    
    # Clamp weight to [0, 1]
    weight = max(0.0, min(1.0, weight))
    
    return c1, c2, weight

def merge_blocks(block1, block2, error_threshold):
    """Try to merge two blocks. Returns merged block if MSE <= threshold, else None."""
    all_colors = block1['colors'] + block2['colors']
    b1, b2, w = find_optimal_base_colors_and_weight(all_colors)
    
    mse = calculate_mse(all_colors, b1, b2, w)
    
    if mse <= error_threshold:
        return {
            'colors': all_colors,
            'base1': b1,
            'base2': b2,
            'weight': w,
            'start_index': block1['start_index']
        }
    else:
        return None

def compress_voxel_colors(colors, error_threshold):
    """
    Compress a list of voxel colors using Variable-Block Color Compression.
    
    Args:
        colors: List of [r, g, b] tuples.
        error_threshold: Maximum allowed MSE per block.
        
    Returns:
        headers: List of dicts with 'base1', 'base2', 'start_index'.
        weights: List of weights corresponding to headers.
    """
    if not colors:
        return [], []
    
    # Step 1: Each color is a block
    blocks = []
    for i, c in enumerate(colors):
        blocks.append({
            'colors': [c],
            'base1': c,
            'base2': c,
            'weight': 0.5,
            'start_index': i,
            'done': False
        })
    
    # Step 2 & 3: Iterative merging
    changed = True
    while changed:
        changed = False
        i = 0
        while i < len(blocks) - 1:
            if blocks[i]['done'] and blocks[i+1]['done']:
                i += 1
                continue
                
            # Try merge with right neighbor
            merged = merge_blocks(blocks[i], blocks[i+1], error_threshold)
            
            if merged:
                # Replace the two blocks with the merged one
                blocks[i] = merged
                blocks.pop(i+1)
                changed = True
                # Do not increment i, check if this new block can merge further left?
                # The algorithm says "Sequentially try to merge...". 
                # To be safe, we might need to check left, but the description implies
                # a single pass of left/right selection. 
                # Let's assume we move to next block to avoid infinite loops or complex backtracking
                # unless specified. The source says "Sequentially try to merge a block with its left or right neighbor".
                # This implies checking both and picking the best.
                # If we merged right, the current block is now larger. 
                # We should probably check if it can merge left now? 
                # The source says "Repeat the iteration with newly merged blocks".
                # So we can just continue to next block in this pass, and the next pass will handle left merges.
                i += 1
            else:
                # Try merge with left neighbor?
                # The source says "whichever has the lowest MSE".
                # We need to check left if i > 0 and left is not done?
                # But if we are iterating sequentially, we might have already processed i-1.
                # Let's stick to the simple "merge with right if better than left" logic or just "merge with right if valid".
                # The source: "Sequentially try to merge a block with its left or right neighbor, whichever has the lowest MSE after merging."
                # This implies looking at both neighbors.
                
                # Check left
                if i > 0:
                    merged_left = merge_blocks(blocks[i-1], blocks[i], error_threshold)
                    # Check right
                    merged_right = merge_blocks(blocks[i], blocks[i+1], error_threshold) if i < len(blocks) - 1 else None
                    
                    mse_left = calculate_mse(blocks[i-1]['colors'] + blocks[i]['colors'], 
                                             blocks[i-1]['base1'], blocks[i-1]['base2'], blocks[i-1]['weight']) # Approx
                    # Actually, we need the MSE of the *merged* result.
                    
                    # Re-evaluate properly
                    best_merge = None
                    best_mse = float('inf')
                    
                    if i > 0:
                        m_left = merge_blocks(blocks[i-1], blocks[i], float('inf')) # Get MSE regardless of threshold for comparison
                        if m_left:
                            # Calculate MSE for left merge
                            # We need a function that returns MSE
                            pass 
                    
                    # Simplified: Just merge right if valid, else mark done?
                    # The source is ambiguous on the "left or right" selection in a sequential pass.
                    # "Sequentially try to merge... whichever has the lowest MSE".
                    # This suggests looking at both options.
                    
                    # Let's implement the "look at both" logic
                    options = []
                    if i > 0:
                        # Calculate MSE for merging i-1 and i
                        all_c = blocks[i-1]['colors'] + blocks[i]['colors']
                        b1, b2, w = find_optimal_base_colors_and_weight(all_c)
                        mse_l = calculate_mse(all_c, b1, b2, w)
                        options.append(('left', i-1, i, mse_l))
                        
                    if i < len(blocks) - 1:
                        all_c = blocks[i]['colors'] + blocks[i+1]['colors']
                        b1, b2, w = find_optimal_base_colors_and_weight(all_c)
                        mse_r = calculate_mse(all_c, b1, b2, w)
                        options.append(('right', i, i+1, mse_r))
                        
                    if options:
                        # Sort by MSE
                        options.sort(key=lambda x: x[3])
                        best_type, idx1, idx2, mse_val = options[0]
                        
                        if mse_val <= error_threshold:
                            # Perform merge
                            merged = merge_blocks(blocks[idx1], blocks[idx2], float('inf')) # Get the merged object
                            # Replace
                            # If left: replace i-1 and i with merged
                            # If right: replace i and i+1 with merged
                            
                            if best_type == 'left':
                                blocks[idx1] = merged
                                blocks.pop(idx2)
                                i = idx1 # Stay at this index to check further merges? Or move on?
                                # If we merge left, the block at i-1 is now larger.
                                # We should probably check if it can merge further left.
                                # But to keep it simple and sequential, let's just decrement i to re-check left?
                                # Or just continue. The source says "Repeat the iteration".
                                i = max(0, i-1) 
                            else:
                                blocks[idx1] = merged
                                blocks.pop(idx2)
                                # i stays same, but the block at i is now larger.
                                # Check if it can merge right again?
                                # Let's just continue to next iteration of while loop?
                                # No, we are in a for/while loop over blocks.
                                # If we remove an element, indices shift.
                                # Let's just continue to next i.
                                pass
                            
                            changed = True
                        else:
                            # No valid merge
                            blocks[i]['done'] = True
                            i += 1
                    else:
                        blocks[i]['done'] = True
                        i += 1
                else:
                    # No left neighbor, check right
                    if i < len(blocks) - 1:
                        merged_right = merge_blocks(blocks[i], blocks[i+1], error_threshold)
                        if merged_right:
                            blocks[i] = merged_right
                            blocks.pop(i+1)
                            changed = True
                            i += 1
                        else:
                            blocks[i]['done'] = True
                            i += 1
                    else:
                        blocks[i]['done'] = True
                        i += 1
        # End of pass
        
    # Extract headers and weights
    headers = []
    weights = []
    for block in blocks:
        headers.append({
            'base1': block['base1'],
            'base2': block['base2'],
            'start_index': block['start_index']
        })
        weights.append(block['weight'])
        
    return headers, weights
```

## Subtopics

### [Variable Bit Rate Color Compression with Macro Blocks](1886-variable-bit-rate-color-compression-with-macro-blocks.md)
Variable Bit Rate Color Compression with Macro Blocks is a compression format for sparse 3D color data that uses variable bits per weight (bpw) to optimize compression ratio, addressing the limitation of fixed bpw methods where theoretical optimal compression is constrained by header overhead. It sets itself apart from its sibling [Variable-Block Color Compression](1878-variable-block-color-compression.md) by introducing macro blocks to localize header offsets, thereby reducing the bit-width required for pointers and enabling efficient variable bit-rate encoding without prohibitive header sizes.

### [GPURealTimeBC6H](13886-gpurealtimebc6h.md)
GPURealTimeBC6H is a GPU-based real-time compressor for the BC6H texture format. It solves the problem of high-quality HDR texture compression by pruning the BC6H block mode search space to specific modes tailored for speed or quality presets. It sets itself apart from offline compressors by performing all arithmetic in log2 space to optimize for perceptual error after tone mapping, and by restricting the search to modes 11 (fast) or 2 and 6 (quality).

## Operating characteristics
*   **Time Complexity:** Encoding is iterative and depends on the number of merges. In the worst case, it is $O(N^2)$ or $O(N \log N)$ depending on implementation, as each pass may merge $O(N)$ blocks, and there may be $O(N)$ passes. Decoding is $O(\log B)$ where $B$ is the number of blocks, due to binary search on headers.
*   **Memory Complexity:** The compressed size is proportional to the number of blocks $B$. Each block adds a fixed-size header (approx. 128 bits + 32 bits = 160 bits) and a weight (3-4 bits). The overhead is low compared to storing all voxel colors.
*   **Performance:** The method is designed for sparse 3D data where spatial coherence allows large blocks to be formed, reducing the number of headers. It outperforms previous attempts in compression/quality [S1, unofficial].
*   **Strengths:** Adaptive block size allows for high compression ratios in smooth regions. Fixed-size headers allow for efficient binary search decoding.
*   **Limitations:** Encoding is computationally expensive due to iterative merging and MSE calculations. Not suitable for real-time encoding without optimization.

## Background
The method was introduced in the context of sparse voxel DAGs and memory efficient compression of surface data [S1, unofficial]. It provides an alternative to mapping attributes to a DAG, offering negligible memory overhead compared to previous methods which doubled the size of the DAG [S1, unofficial].

## Sources
[S1] PDF On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
[S2] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
