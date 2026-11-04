# S3Dc GPU Decompression and Rendering

Also called: Fast Decompression, S3Dc, Scalar 3Dc

## Overview
S3Dc (Scalar 3Dc) is a lossy volume compression algorithm for scalar data that enables direct GPU rendering by storing compressed data in two 3D textures: a MinMax Volume (MMV) and an Index Volume (IV) [S1, unofficial]. Unlike standard texture compression formats like DXT or VTC which are designed for RGB(A) color data and cause block artifacts on non-smooth scalar fields, S3Dc is tailored for scalar volumetric datasets, allowing the GPU to reconstruct voxel values on-the-fly during rendering without storing decompressed data in memory [S1, unofficial][S2]. This approach reduces CPU-GPU transfer bottlenecks and allows interactive frame rates for large volumes that would otherwise exceed GPU memory limits [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Original Volume (OV) with dimensions $v_x \times v_y \times v_z$ and sample precision $s_{bits}$.
*   **Parameters:** Block dimensions $b_x, b_y, b_z$ (divisors of volume dimensions) and index bit depth $i_{bits} \in \{1, 2, 4, 8\}$ [S1, unofficial].
*   **Output:** Two 3D textures: MinMax Volume (MMV) and Index Volume (IV) [S1, unofficial].

**Data Structures**
1.  **MinMax Volume (MMV):** A 3D texture with internal format `Luminance16Alpha16` (assuming 16-bit original samples) [S1, unofficial].
    *   Each texel stores a pair of values: Luminance channel holds the minimum sample value ($\min$) of the corresponding block; Alpha channel holds the maximum sample value ($\max$) [S1, unofficial].
    *   Dimensions: $\frac{v_x}{b_x} \times \frac{v_y}{b_y} \times \frac{v_z}{b_z}$ [S1, unofficial].
2.  **Index Volume (IV):** A 3D texture with internal format `Alpha16` [S1, unofficial].
    *   Stores compressed indices for each original voxel.
    *   To minimize texture fetches, correlative index values in the x-direction are merged into a single 16-bit texture element [S1, unofficial].
    *   Dimensions: $v_x \times v_y \times v_z$ (logical), but physically packed based on $i_{bits}$ [S1, unofficial].

**Compression Steps (CPU)**
1.  **Block Subdivision:** Divide the Original Volume (OV) into virtual blocks of size $b_x \times b_y \times b_z$ [S1, unofficial].
2.  **MinMax Calculation:** For each block, compute the minimum ($\min$) and maximum ($\max$) sample values from all voxels within that block [S1, unofficial].
3.  **MMV Storage:** Store $\min$ and $\max$ for each block into the MMV texture [S1, unofficial].
4.  **Index Computation:** For each voxel in OV with value $value$:
    *   Identify the block it belongs to.
    *   Retrieve $\min$ and $\max$ for that block from MMV.
    *   Compute the index $idx$ using:
        $$idx = \left\lfloor \frac{value - \min}{\max - \min} (2^{i_{bits}} - 1) + \frac{1}{2} \right\rfloor \quad (1)$$
        where $idx \in \{0, \dots, 2^{i_{bits}} - 1\}$ [S1, unofficial].
5.  **IV Storage:** Store $idx$ into the IV texture. If $i_{bits} < 16$, pack multiple indices into one 16-bit texel by merging x-direction neighbors [S1, unofficial].

**Decompression and Rendering (GPU)**
Rendering is performed directly on the GPU using fragment shaders (for texture slicing) or vertex shaders (for splatting) [S1, unofficial][S3]. The following steps describe the fragment shader approach for texture slicing:

1.  **Coordinate Transformation:** Compute texture coordinates for MMV and IV based on the current ray/quad position. Linear transformations map the original volume coordinates to the compressed texture coordinates [S1, unofficial].
2.  **Texture Fetch:**
    *   Fetch $\min$ and $\max$ from MMV.
    *   Fetch the packed index data from IV.
3.  **Index Extraction:** If indices are packed, extract the specific index $idx$ corresponding to the current voxel using bit manipulation or a lookup table [S1, unofficial][S3].
4.  **Value Reconstruction:** Compute the decompressed value $dv$ using:
    $$dv = \min + (\max - \min) \cdot \frac{idx}{2^{i_{bits}} - 1} \quad (2)$$
5.  **Interpolation (Optional):**
    *   **Nearest Neighbor:** Uses the decompressed value of the closest voxel. Requires 2 texture fetches (MMV + IV) [S1, unofficial].
    *   **Linear Interpolation:** Computes an 8-neighborhood interpolated value. Requires 16 texture fetches (or 6 more for gradient calculation) [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compression (CPU)
def compress_volume(OV, bx, by, bz, ibits, sbits):
    # OV is a 3D array of size vx x vy x vz
    # sbits is assumed to be 16 for MMV storage format
    vx, vy, vz = OV.shape
    num_blocks_x = vx // bx
    num_blocks_y = vy // by
    num_blocks_z = vz // bz
    
    # Initialize MMV: stores (min, max) per block
    MMV = np.zeros((num_blocks_x, num_blocks_y, num_blocks_z, 2), dtype=np.uint16)
    
    # Initialize IV: stores indices per voxel
    # Logical size is vx x vy x vz, but packed into uint16
    IV_packed_size = (vx * vy * vz) // (16 // ibits)
    IV = np.zeros(IV_packed_size, dtype=np.uint16)
    
    max_idx = (1 << ibits) - 1
    
    for z in range(num_blocks_z):
        for y in range(num_blocks_y):
            for x in range(num_blocks_x):
                # Extract block from OV
                x_start = x * bx
                y_start = y * by
                z_start = z * bz
                block = OV[x_start:x_start+bx, y_start:y_start+by, z_start:z_start+bz]
                
                # Compute min and max
                block_min = np.min(block)
                block_max = np.max(block)
                
                # Store in MMV
                MMV[x, y, z] = [block_min, block_max]
                
                # Compute indices for all voxels in this block
                for vz_idx in range(bz):
                    for vy_idx in range(by):
                        for vx_idx in range(bx):
                            val = block[vz_idx, vy_idx, vx_idx]
                            
                            # Avoid division by zero if max == min
                            if block_max == block_min:
                                idx = 0
                            else:
                                # Formula (1)
                                idx = int(np.floor((val - block_min) / (block_max - block_min) * max_idx + 0.5))
                            
                            # Store in IV
                            # Map 3D voxel coordinates to linear index
                            voxel_linear_idx = (z_start + vz_idx) * (vx * vy) + (y_start + vy_idx) * vx + (x_start + vx_idx)
                            packed_idx = voxel_linear_idx // (16 // ibits)
                            bit_offset = (voxel_linear_idx % (16 // ibits)) * ibits
                            
                            # Clear bits and set new index
                            IV[packed_idx] = (IV[packed_idx] & ~(max_idx << bit_offset)) | (idx << bit_offset)
                            
    return MMV, IV

# Decompression (GPU Fragment Shader)
def decompress_value(mmv_tex, iv_tex, tex_coords, ibits):
    # tex_coords are normalized coordinates for the original volume
    # Compute block coordinates
    bx, by, bz = 16, 16, 16 # Example block size, must match compression params
    vx, vy, vz = 512, 512, 512 # Example volume size
    
    # Map to MMV coordinates (block indices)
    mmv_coords = (tex_coords * [vx/bx, vy/by, vz/bz])
    mmv_min_max = mmv_tex.sample(mmv_coords) # Returns (min, max)
    
    min_val = mmv_min_max[0]
    max_val = mmv_min_max[1]
    
    # Map to IV coordinates (voxel indices)
    iv_coords = tex_coords
    packed_index = iv_tex.sample(iv_coords) # Returns packed uint16
    
    # Extract specific index from packed value
    # Assuming x-direction merging and ibits bits per index
    # Calculate which index in the packed word corresponds to current voxel
    # This requires knowing the exact packing layout (x-merging)
    # Simplified extraction logic:
    voxel_idx = int(tex_coords[0] * vx) # Assuming unit step in x for simplicity in this snippet
    packed_word_idx = voxel_idx // (16 // ibits)
    bit_offset = (voxel_idx % (16 // ibits)) * ibits
    max_idx = (1 << ibits) - 1
    
    # Extract bits
    idx = (packed_index >> bit_offset) & max_idx
    
    # Formula (2)
    if max_val == min_val:
        return min_val
    else:
        dv = min_val + (max_val - min_val) * (idx / max_idx)
        return dv
```

**Edge Cases and Pitfalls**
*   **Block Dimensions:** $b_x, b_y, b_z$ must be divisors of $v_x, v_y, v_z$ to ensure uniform block subdivision [S1, unofficial].
*   **GPU Compatibility:** Older GPUs do not support non-power-of-two textures. The algorithm must pad 3D textures to powers of two, which reduces the effective compression ratio [S1, unofficial].
*   **Artifacts:** Large block sizes with low $i_{bits}$ (e.g., $i_{bits}=2$) cause artifacts in regions with high range variation because the limited indices cannot represent the gradient [S1, unofficial].
*   **Zero Division:** If $\max = \min$ in a block, the index is defined as 0 to avoid division by zero [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity**
*   **Compression Ratio (CR):**
    $$CR = \left( \frac{i_{bits}}{s_{bits}} + \frac{2}{b_x \cdot b_y \cdot b_z} \right)^{-1} \quad (5)$$
    The dominant term is the ratio of index bits to sample bits [S1, unofficial].
*   **Memory:** Requires storage for two 3D textures (MMV and IV). MMV size is $\frac{2}{b_x b_y b_z}$ of the original volume size. IV size is $\frac{i_{bits}}{s_{bits}}$ of the original volume size [S1, unofficial].
*   **Rendering Speed:**
    *   Nearest neighbor access requires 2 texture fetches and is faster [S1, unofficial].
    *   Linear interpolation requires 16 texture fetches (or 6 more for gradients) but yields better quality [S1, unofficial].

**Measured Performance**
*   **Hardware:** Tested on a 2.6GHz Quad Core PC with 8GB RAM and GeForce 8800 GTX [S1, unofficial].
*   **Configuration:** For a $512 \times 512 \times 486$ volume with 16-bit samples, $i_{bits}=4$ and block size $16^3$ provided the best balance of frame rate, quality, and compression [S1, unofficial].
*   **Metrics:** For $i_{bits}=2$ and block size $16^3$:
    *   Compression Ratio: 7.968
    *   Nearest Frame Rate: 16.30 fps
    *   Linear Frame Rate: 5.05 fps
    *   PSNR: 28.30
    *   SSIM: 95.04 [S1, unofficial]

**Strengths and Limitations**
*   **Strengths:** Direct GPU rendering without decompression overhead; supports scalar data better than RGB-based compression formats; random access capability [S1, unofficial].
*   **Limitations:** Lossy compression; artifacts in high-variation regions with low index bits; dependency on GPU texture format limitations (e.g., power-of-two constraints) [S1, unofficial].

## Background
S3Dc was introduced as a lossy volume compression algorithm suitable for scalar values, inspired by the 3Dc normal map compression technique [S1, unofficial]. It was developed to address the limitations of hardware-accelerated texture compression formats like DXT and VTC, which support only RGB(A) color formats and cause block artifacts for non-smooth scalar data [S1, unofficial]. The method allows for CPU-side compression with ratios up to 4:1 or 8:1 while maintaining good visual quality and enabling direct rendering on the GPU [S1, unofficial].

## Sources
[S1] S3Dc: A 3Dc-based Volume Compression Algorithm, https://doi.org/10.2312/localchapterevents/ceig/ceig08/095-104, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
