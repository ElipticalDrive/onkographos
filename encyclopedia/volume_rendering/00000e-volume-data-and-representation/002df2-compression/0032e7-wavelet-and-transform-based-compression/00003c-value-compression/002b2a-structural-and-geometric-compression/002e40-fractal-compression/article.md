# Fractal Compression

Also called: Block-based Fractal Compression, Iterative Function System Compression, 分形压缩

## Overview
Fractal compression is a lossy volume compression method that exploits self-similarity within the dataset by mapping each "range block" to a larger, dissimilar "domain block" from the same volume, storing only the affine transformation required to convert the domain block into the range block [S1]. Unlike vector quantization, which stores a reference to a codeword, fractal compression stores the geometric relationship between two different-sized blocks, allowing for high compression ratios at the cost of extreme encoding complexity and iterative reconstruction during decoding [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset $V$.
*   **Output:** A compressed bitstream containing, for each range block, the position (coordinates) of the matched domain block and the parameters of the affine transformation mapping the domain to the range.

**Data Structures**
The compressed data structure consists of a list of entries, one per range block. Each entry contains:
1.  **Domain Position:** 3D integer coordinates $(x_d, y_d, z_d)$ indicating the top-left (or equivalent anchor) of the domain block in the original volume grid.
2.  **Transformation Parameters:** A set of coefficients defining the affine transform. For a 3D volume, this typically includes scaling factors, rotation/rotation-like parameters, and a brightness/contrast offset (if intensity scaling is allowed). The exact bit-packing is not specified in the sources, but it must be sufficient to store the integer coordinates and floating-point transformation coefficients [S1].

**Algorithm Steps**
1.  **Block Partitioning:** Divide the volume into non-overlapping range blocks of size $N \times N \times N$.
2.  **Domain Search:** For each range block $R$:
    *   Define a search space of potential domain blocks $D$. Domain blocks are typically larger than range blocks (e.g., $2N \times 2N \times 2N$) to allow for scaling [S1].
    *   Search for the domain block $D^*$ that minimizes the distortion metric (e.g., Mean Squared Error) between the transformed $D^*$ and $R$. The transformation includes geometric affine operations and potentially intensity adjustments (contrast/brightness) [S1].
    *   If a match is found within a predefined threshold, store the position of $D^*$ and the transformation parameters.
3.  **Recursive Subdivision (Fractal Step):** If no suitable domain block is found in the current search space (i.e., the distortion exceeds the threshold):
    *   Subdivide the current range block into smaller sub-blocks (e.g., 8 sub-blocks for a $2 \times 2 \times 2$ subdivision).
    *   Recursively apply the search process (Step 2) to each sub-block.
    *   This recursion continues until a base block size is reached or a maximum depth is exceeded [S1].
4.  **Encoding:** Write the stored positions and transformation parameters to the output bitstream.

**Formulas**
Let $R$ be the intensity values of a range block and $D$ be the intensity values of a domain block. The goal is to find an affine transformation $T$ such that:
$$ T(D) \approx R $$
The distortion $E$ is minimized:
$$ E = \| R - T(D) \|^2 $$
Where $T$ typically involves scaling, rotation, and translation in the spatial domain, and potentially a linear intensity transform $I' = aI + b$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fractal_compress(volume, range_size, domain_size, threshold, max_depth):
    """
    Compress a volume using fractal compression.
    
    Args:
        volume: 3D numpy array or similar structure representing the volume.
        range_size: Integer, size of the range block (N).
        domain_size: Integer, size of the domain block (M > N).
        threshold: Float, maximum allowed distortion for a match.
        max_depth: Integer, maximum recursion depth for subdivision.
        
    Returns:
        compressed_data: List of tuples (domain_pos, transform_params) for each range block.
    """
    compressed_data = []
    volume_shape = volume.shape
    
    # Iterate over all range blocks
    for z in range(0, volume_shape[0] - range_size + 1, range_size):
        for y in range(0, volume_shape[1] - range_size + 1, range_size):
            for x in range(0, volume_shape[2] - range_size + 1, range_size):
                range_block = volume[z:z+range_size, y:y+range_size, x:x+range_size]
                
                # Attempt to find a match
                domain_pos, transform, distortion = find_best_match(
                    volume, range_block, range_size, domain_size, threshold
                )
                
                if distortion <= threshold:
                    compressed_data.append((domain_pos, transform))
                else:
                    # If no match, subdivide and recurse
                    # Note: This requires handling boundary conditions and 
                    # merging results from sub-blocks.
                    sub_blocks = divide_into_subblocks(range_block, range_size)
                    sub_results = []
                    for i, sub_block in enumerate(sub_blocks):
                        # Recursive call with reduced range_size and depth
                        # The domain search space might also be adjusted
                        sub_compressed = fractal_compress_recursive(
                            volume, sub_block, range_size // 2, domain_size, 
                            threshold, max_depth - 1
                        )
                        sub_results.extend(sub_compressed)
                    
                    # Store markers or flags indicating subdivision 
                    # instead of a single transform. 
                    # The sources do not specify the exact bitstream format 
                    # for the recursive case, so we assume a list of 
                    # sub-block results is appended.
                    compressed_data.extend(sub_results)
                    
    return compressed_data

def find_best_match(volume, range_block, range_size, domain_size, threshold):
    """
    Search for the best matching domain block in the volume.
    
    Returns:
        domain_pos: Tuple (z, y, x) of the best match.
        transform: Object or tuple representing the affine transform.
        distortion: Float, the MSE between transformed domain and range.
    """
    best_distortion = float('inf')
    best_pos = None
    best_transform = None
    
    # Define search space for domain blocks
    # Domain blocks are larger, so they can be positioned such that 
    # a scaled/transformed version fits the range block.
    # The exact search strategy (sliding window, etc.) is not detailed 
    # in the sources, but implies a exhaustive or heuristic search.
    
    for z in range(0, volume.shape[0] - domain_size + 1):
        for y in range(0, volume.shape[1] - domain_size + 1):
            for x in range(0, volume.shape[2] - domain_size + 1):
                domain_block = volume[z:z+domain_size, y:y+domain_size, x:x+domain_size]
                
                # Apply candidate transformations
                # This includes geometric affine transforms and intensity scaling
                for transform in get_candidate_transforms(range_size, domain_size):
                    transformed_domain = apply_transform(domain_block, transform)
                    
                    # Calculate distortion
                    # Interpolation may be needed if domain_size != range_size
                    # after transformation
                    distortion = calculate_mse(transformed_domain, range_block)
                    
                    if distortion < best_distortion:
                        best_distortion = distortion
                        best_pos = (z, y, x)
                        best_transform = transform
                        
    if best_distortion <= threshold:
        return best_pos, best_transform, best_distortion
    else:
        return None, None, best_distortion

def apply_transform(domain_block, transform):
    """
    Apply geometric and intensity transformations to a domain block.
    """
    # Geometric: Scale, Rotate, Translate
    # Intensity: Contrast/Brightness adjustment
    # Implementation details depend on the specific affine model used.
    # Sources mention "affine transformations" and "color and voxel permutations"
    # but do not provide the exact mathematical form.
    pass

def calculate_mse(block1, block2):
    return np.mean((block1 - block2) ** 2)

def get_candidate_transforms(range_size, domain_size):
    """
    Generate a set of candidate affine transformations.
    """
    # This is a placeholder for the specific transformation set.
    # Sources mention "affine transformations" and "permutations".
    pass

def divide_into_subblocks(block, size):
    """
    Divide a block into 8 sub-blocks (2x2x2).
    """
    half = size // 2
    subblocks = []
    for dz in [0, half]:
        for dy in [0, half]:
            for dx in [0, half]:
                subblocks.append(block[dz:dz+half, dy:dy+half, dx:dx+half])
    return subblocks

def fractal_compress_recursive(volume, range_block, range_size, domain_size, threshold, max_depth):
    """
    Recursive helper for subdivision.
    """
    if max_depth <= 0 or range_size == 1:
        # Base case: store raw data or minimal representation
        # Sources do not specify the base case encoding.
        return [(None, None, 0)] # Placeholder
        
    return fractal_compress(volume, range_size, domain_size, threshold, max_depth)
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions:** When domain blocks are near the volume boundary, transformations may map pixels outside the volume. These must be handled (e.g., clamping, reflection, or zero-padding) [S1].
*   **Precision:** The iterative reconstruction process is sensitive to precision errors in the affine transformations. Small errors can accumulate, leading to visible artifacts [S1].
*   **Degenerate Input:** If the volume lacks self-similarity (e.g., random noise), the search will fail to find matches, leading to deep recursion and poor compression ratios [S1].

## Operating characteristics
*   **Time Complexity:** Encoding is extremely computationally intensive due to the exhaustive search for similar blocks and the recursive subdivision process [S1]. Decompression is straightforward but requires iterative application of transformations, which can be slow for large volumes [S1].
*   **Memory Complexity:** The compressed data size is small, as only positions and transformation parameters are stored [S1]. However, the search process requires significant memory to hold the volume and intermediate blocks.
*   **Performance:** High compression ratios are achievable, but the encoding time is prohibitive for large volumes [S1].
*   **Strengths:** High compression ratio; effective for volumes with self-similar structures.
*   **Limitations:** Extremely high encoding complexity; slow decompression; problematic for real-time rendering due to iterative reconstruction; not suitable for very large volumes [S1].
*   **When to Use:** Off-line volume compression where storage space is critical and encoding time is not a constraint [S1].
*   **When Not to Use:** Real-time rendering, large dynamic volumes, or volumes with low self-similarity [S1].

## Background
Fractal compression was proposed for volume data by Chentanez et al. [CHF96] [S1]. It is based on the mathematical theory of fractals and Iterative Function Systems (IFS), where complex shapes are generated by simple affine transformations [S1]. The method shares common problems with vector quantization, such as the computational intensity of the search process [S1]. Restricted forms of this method have been proposed for real-time rendering in combination with dictionary methods, as suggested for image coding [HS00] [S1].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
