# DCT Transform Method
Also called: DCT compression
## Overview
The DCT Transform Method is a value compression technique for TSDF volumes that partitions data into blocks and subblocks, transforms them using the 3D Discrete Cosine Transform (DCT), and entropy codes the resulting coefficients [S1][S2]. It distinguishes itself from the DFT approach by using the DCT, which processes real-valued data directly without complex arithmetic, and follows a compression pipeline analogous to 2D JPEG lossy compression [S1][S2].
## How it works
**Inputs:** A TSDF volume decomposed into fixed-size blocks by the parent method (TSDF Block Transform Coding) [G].
**Outputs:** A compressed bitstream of the volume data [G].

**Data Structures:**
* **Block:** A fixed-size 3D cube of voxels. Dimensions not given in the sources.
* **Subblock:** A smaller 3D cube within a block. Dimensions not given in the sources.
* **Coefficient Array:** 3D array of DCT coefficients for each subblock [G].

**Steps:**
1. Partition the volume data into blocks [S1][S2].
2. Subdivide each block into smaller subblocks [S1][S2].
3. Transform each subblock using the 3D Discrete Cosine Transform (DCT) [S1][S2].
4. Group the DC coefficients (zero-frequency components) from all subblocks, quantize them, and apply delta encoding [S1][S2].
5. Compress the AC coefficients (non-zero frequency components) using zigzag ordering, run-length encoding, and Huffman encoding, following the same pipeline as the DFT approach [S1][S2].

**Formulas:**
The 1D DCT-II is computed as $X_k = \sum_{n=0}^{N-1} x_n \cos\left(\frac{\pi k (2n+1)}{2N}\right)$ for $k=0,\dots,N-1$ [G]. The 3D transform is applied separably along each axis [G].

**Parameters:**
* **Block Size:** Not given in the sources.
* **Subblock Size:** Not given in the sources.
* **Quantization Step:** Not given in the sources.

**Edge Cases:**
* **Boundary Conditions:** Proper handling of block edges is required to avoid artifacts [G].
* **Transformed-Domain Rendering Limitation:** DCT does not support volume rendering based on transformed domain data [S3, unofficial].
* **Degenerate Input:** If a subblock is constant, all AC coefficients will be zero, leading to efficient compression [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dct_transform_method(volume_data, block_size, subblock_size, quantization_step):
    compressed_stream = []
    dc_values = []
    ac_values_list = []

    # Step 1 & 2: Partition volume into blocks, then subdivide into subblocks
    for i in range(0, len(volume_data), block_size):
        for j in range(0, len(volume_data[0]), block_size):
            for k in range(0, len(volume_data[0][0]), block_size):
                subblock = get_subblock(volume_data, i, j, k, subblock_size)
                # Step 3: Transform subblock using 3D DCT
                dct_coeffs = apply_3d_dct(subblock)
                dc_values.append(dct_coeffs[0][0][0])
                ac_vals = []
                for a in range(len(dct_coeffs)):
                    for b in range(len(dct_coeffs[0])):
                        for c in range(len(dct_coeffs[0][0])):
                            if a == 0 and b == 0 and c == 0: continue
                            ac_vals.append(dct_coeffs[a][b][c])
                ac_values_list.append(ac_vals)

    # Step 4: Group DC coefficients, quantize, delta encode
    quantized_dc = [int(v / quantization_step) for v in dc_values]
    delta_dc = [quantized_dc[0]]
    for i in range(1, len(quantized_dc)):
        delta_dc.append(quantized_dc[i] - quantized_dc[i-1])
    compressed_stream.append(huffman_encode(delta_dc))

    # Step 5: Compress AC coefficients with zigzag, run-length, Huffman
    for ac_vals in ac_values_list:
        zigzag_ac = zigzag_scan(ac_vals)
        rle_ac = run_length_encode(zigzag_ac)
        compressed_stream.append(huffman_encode(rle_ac))

    return compressed_stream

def get_subblock(V, i, j, k, size):
    return [[[V[x][y][z] for z in range(k, k+size)] for y in range(j, j+size)] for x in range(i, i+size)]

def apply_3d_dct(data):
    # Separable 3D DCT: apply 1D DCT along each axis sequentially [G]
    temp = apply_1d_dct(data, axis=0)
    temp = apply_1d_dct(temp, axis=1)
    return apply_1d_dct(temp, axis=2)

def apply_1d_dct(data, axis):
    # Standard DCT-II computation along the specified axis [G]
    if axis == 0:
        return [[apply_1d_dct_1d([data[x][y][z] for x in range(len(data))]) for y in range(len(data[0]))] for z in range(len(data[0][0]))]
    elif axis == 1:
        return [[apply_1d_dct_1d([data[x][y][z] for y in range(len(data[0]))]) for y in range(len(data[0]))] for x in range(len(data))]
    else:
        return [[[apply_1d_dct_1d([data[x][y][z] for z in range(len(data[0][0]))]) for z in range(len(data[0][0]))] for y in range(len(data[0]))] for x in range(len(data))]

def apply_1d_dct_1d(signal):
    N = len(signal)
    return [sum(signal[n] * cos(pi * k * (2 * n + 1) / (2 * N)) for n in range(N)) for k in range(N)]

def zigzag_scan(coeffs):
    # Reorders 1D coefficient array into a sequence that groups low-frequency components first [G]
    # Follows standard 3D zigzag traversal order
    pass # Standard 3D zigzag scan implementation [G]

def run_length_encode(sequence):
    if not sequence: return []
    encoded = []
    current_val = sequence[0]
    count = 1
    for val in sequence[1:]:
        if val == current_val:
            count += 1
        else:
            encoded.append((current_val, count))
            current_val = val
            count = 1
    encoded.append((current_val, count))
    return encoded

def huffman_encode(data):
    # Assigns variable-length binary codes based on symbol frequency to compress data [G]
    pass # Standard Huffman coding algorithm [G]
```
## Subtopics

### Discrete Cosine Transform (DCT)
Partitions data into blocks and decomposes each block using DCT. A key limitation is that it does not support volume rendering based on transformed domain data [S3, unofficial].

## Operating characteristics
* **Time Complexity:** Compression and decompression are faster than the DFT approach [S3, unofficial].
* **Memory Complexity:** Not given in the sources.
* **Strengths:** Avoids complex arithmetic of DFT. Good energy compaction for smooth data.
* **Limitations:** Block-based nature can introduce blocking artifacts if quantization is too aggressive. Does not support volume rendering based on transformed domain data [S3, unofficial].
* **When to use:** Not given in the sources.
* **When not to use:** Not given in the sources.
## Background
The DCT Transform Method was proposed by Yau and Lippert [YL95] [S1][S2]. It partitions data into blocks and subblocks, transforms them using the 3D DCT, and entropy codes the coefficients using a pipeline analogous to 2D JPEG lossy compression [S1][S2].
## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
