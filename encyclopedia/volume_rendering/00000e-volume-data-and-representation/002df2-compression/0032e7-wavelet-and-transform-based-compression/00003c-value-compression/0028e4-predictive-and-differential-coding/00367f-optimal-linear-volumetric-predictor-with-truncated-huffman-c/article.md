# Optimal Linear Volumetric Predictor with Truncated Huffman Coding

Also called: DPCM with Huffman Coding, DPCM-Huffman, Predictive Huffman Coding for Volumes, Volume DPCM with Huffman Coding, compvox algorithm, volumetric DPCM with optimal coefficients, 差值编码与霍夫曼编码结合

## Overview
This method is a lossless compression algorithm for 3D volume data that exploits spatial coherence by predicting voxel values from previously encoded neighbors and encoding the difference (residual) using a truncated Huffman code [S1, unofficial]. It achieves greater compression than generic algorithms (e.g., gzip, zip) by removing spatial redundancy in addition to statistical redundancy [S1, unofficial]. The method is distinguished from its siblings by its use of a globally optimal linear predictor derived from the Yule-Walker equations, assuming wide-sense stationarity, and a specific truncated Huffman coding scheme to handle the large alphabet of 16-bit residuals [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input:** A 3D volume array $v(x, y, z)$ of size $X \times Y \times Z$. The source notes specify 16-bit voxel values [S1, unofficial].
- **Output:** A compressed bitstream containing the encoded residuals, preceded by a header storing the mean, predictor coefficients, and Huffman table [S1, unofficial].

### Data Structures
1.  **Volume Data:** $v(x, y, z)$, integer values.
2.  **Covariance Matrix $A$:** A $3 \times 3$ symmetric matrix used to solve for predictor coefficients.
3.  **Covariance Vector $b$:** A $3 \times 1$ vector.
4.  **Predictor Coefficients:** $a = [a_1, a_2, a_3]^T$.
5.  **Huffman Table:** A mapping from difference values $s_i$ to bit codes.
6.  **Header:** Stores $\bar{v}$, $a_1, a_2, a_3$, and the Huffman table [S1, unofficial].

### Parameters
-   **Scan Order:** Raster-scan order (lexicographic order of indices $x, y, z$) to ensure causality [S1, unofficial].
-   **Predictor Order:** 3 coefficients, using the 6-neighborhood (face-sharing neighbors: $x-1, y-1, z-1$) [S1, unofficial].
-   **Truncated Huffman Parameter $L_1$:** The number of most frequent symbols Huffman coded. Default $L_1 = 256$ [S1, unofficial].
-   **Alphabet Size $L$:** For 16-bit voxels, difference values range from $-66535$ to $65536$, resulting in $L = 131,071$ possible symbols [S1, unofficial].

### Formulas
1.  **Mean:**
    $$ \bar{v} = \frac{1}{XYZ} \sum_{x=0}^{X-1} \sum_{y=0}^{Y-1} \sum_{z=0}^{Z-1} v(x, y, z) $$
2.  **Covariance:**
    $$ r(x, y, z) = \frac{1}{(X-2)(Y-2)(Z-2)} \sum_{x'=1}^{X-2} \sum_{y'=1}^{Y-2} \sum_{z'=1}^{Z-2} [v(x', y', z') - \bar{v}] [v(x'+x, y'+y, z'+z) - \bar{v}] $$
    for lags $x, y, z \in \{-1, 0, 1\}$ [S1, unofficial].
3.  **Predictor Coefficients (Yule-Walker):**
    Solve $A \mathbf{a} = \mathbf{b}$ for $\mathbf{a} = [a_1, a_2, a_3]^T$, where:
    $$
    A = \begin{bmatrix}
    r(0,0,0) & r(1,-1,0) & r(1,0,-1) \\
    r(-1,1,0) & r(0,0,0) & r(0,1,-1) \\
    r(-1,0,1) & r(0,-1,1) & r(0,0,0)
    \end{bmatrix}, \quad
    \mathbf{b} = \begin{bmatrix}
    r(1,0,0) \\
    r(0,1,0) \\
    r(0,0,1)
    \end{bmatrix}
    $$
    [S1, unofficial].
4.  **Prediction:**
    $$ \hat{v}(x, y, z) = a_1 v_1(x, y, z) + a_2 v_2(x, y, z) + a_3 v_3(x, y, z) $$
    where neighbors are mean-subtracted and zeroed at boundaries:
    $$
    v_1(x, y, z) = \begin{cases} 0 & \text{if } x=0 \\ v(x-1, y, z) - \bar{v} & \text{otherwise} \end{cases}
    $$
    $$
    v_2(x, y, z) = \begin{cases} 0 & \text{if } y=0 \\ v(x, y-1, z) - \bar{v} & \text{otherwise} \end{cases}
    $$
    $$
    v_3(x, y, z) = \begin{cases} 0 & \text{if } z=0 \\ v(x, y, z-1) - \bar{v} & \text{otherwise} \end{cases}
    $$
    [S1, unofficial].
5.  **Residual:**
    $$ d(x, y, z) = v(x, y, z) - \hat{v}(x, y, z) $$
    [S1, unofficial].
6.  **Entropy:**
    $$ H = - \sum_{i=1}^{k} p_i \log_2 p_i $$
    where $k = |\Sigma|$ and $p_i$ is the probability of symbol $i$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_volume(volume: Array3D, L1: int = 256) -> bytes:
    """
    Compress a 3D volume using Optimal Linear Predictor and Truncated Huffman Coding.
    volume: 3D array of shape (X, Y, Z) with 16-bit integer values.
    L1: Number of most frequent symbols to Huffman code.
    Returns: Compressed bytes.
    """
    X, Y, Z = volume.shape
    
    # Step 1: Estimate Mean
    mean_v = sum(volume[x, y, z] for x in range(X) for y in range(Y) for z in range(Z)) / (X * Y * Z)
    
    # Step 2: Estimate Covariance
    # Covariance r(x, y, z) for lags in {-1, 0, 1}
    # Note: The source uses a specific normalization factor and summation range.
    # We compute the required covariance values for the Yule-Walker system.
    # The source defines r(x,y,z) using dummy variables x', y', z' from 1 to X-2, etc.
    
    cov = {}
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                sum_val = 0
                count = 0
                for x_prime in range(1, X - 1):
                    for y_prime in range(1, Y - 1):
                        for z_prime in range(1, Z - 1):
                            val1 = volume[x_prime, y_prime, z_prime] - mean_v
                            val2 = volume[x_prime + dx, y_prime + dy, z_prime + dz] - mean_v
                            sum_val += val1 * val2
                            count += 1
                cov[(dx, dy, dz)] = sum_val / count if count > 0 else 0.0

    # Step 3: Calculate Predictor Coefficients
    # Solve A * a = b
    # A matrix elements based on source Eq (8)
    A = [
        [cov[(0,0,0)], cov[(1,-1,0)], cov[(1,0,-1)]],
        [cov[(-1,1,0)], cov[(0,0,0)], cov[(0,1,-1)]],
        [cov[(-1,0,1)], cov[(0,-1,1)], cov[(0,0,0)]]
    ]
    b = [
        cov[(1,0,0)],
        cov[(0,1,0)],
        cov[(0,0,1)]
    ]
    
    # Solve linear system (e.g., using Gaussian elimination or numpy.linalg.solve)
    # [G] Standard linear algebra solver for 3x3 system
    a1, a2, a3 = solve_linear_system(A, b)
    
    # Step 4 & 5: Calculate Differences and Count Frequencies
    # We scan to calculate differences and count frequencies.
    # To save memory, we do not store all differences, but we need to count them.
    # Since the alphabet is large (131,071), we use a frequency array or hash map.
    
    freq = {} # Map from difference value to count
    
    # Pre-calculate mean-subtracted neighbors for efficiency if needed, 
    # but direct access is fine for correctness.
    
    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                # Calculate neighbors with mean subtraction and boundary handling
                v1 = 0 if x == 0 else volume[x-1, y, z] - mean_v
                v2 = 0 if y == 0 else volume[x, y-1, z] - mean_v
                v3 = 0 if z == 0 else volume[x, y, z-1] - mean_v
                
                pred = a1 * v1 + a2 * v2 + a3 * v3
                diff = volume[x, y, z] - pred
                
                freq[diff] = freq.get(diff, 0) + 1
                
    # Step 6: Build Huffman Table
    # Sort symbols by frequency
    sorted_symbols = sorted(freq.keys(), key=lambda s: freq[s], reverse=True)
    
    # Truncated Huffman: First L1 symbols get Huffman codes, rest get fixed-length
    huffman_table = {}
    
    # Get probabilities for the top L1 symbols
    total_count = sum(freq.values())
    
    # Huffman coding for top L1 symbols
    # [G] Standard Huffman tree construction and code assignment
    huffman_codes = build_huffman_codes(sorted_symbols[:L1], [freq[s] for s in sorted_symbols[:L1]])
    
    for i, symbol in enumerate(sorted_symbols[:L1]):
        huffman_table[symbol] = huffman_codes[i]
        
    # Fixed-length codes for remaining symbols
    remaining_symbols = sorted_symbols[L1:]
    if remaining_symbols:
        # Determine bit length for fixed codes
        # [G] The source does not specify the exact bit length for the fixed codes,
        # only that they are fixed-length. A common approach is ceil(log2(remaining_count)).
        # However, to be strictly implementable from text, we note this is unspecified.
        # We will assume a standard fixed-length encoding for the remaining alphabet.
        # Since the max difference is 131,071, and we handled 256, 
        # the remaining are encoded with a fixed number of bits sufficient to index them or their values.
        # The source says "assigned fixed-length codes".
        # Let's assume the codes are simply the binary representation of the symbol index 
        # or the symbol value itself if it fits, but typically it's a prefix-free code.
        # Without explicit detail, we mark this as a gap.
        pass 
        
    # Step 7: Encode
    output_bits = []
    
    # Encode Header
    # [G] Serialize mean_v, a1, a2, a3, and huffman_table structure into bits.
    # The exact serialization format is not specified, but these must be stored.
    
    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                v1 = 0 if x == 0 else volume[x-1, y, z] - mean_v
                v2 = 0 if y == 0 else volume[x, y-1, z] - mean_v
                v3 = 0 if z == 0 else volume[x, y, z-1] - mean_v
                
                pred = a1 * v1 + a2 * v2 + a3 * v3
                diff = volume[x, y, z] - pred
                
                if diff in huffman_table:
                    output_bits.append(huffman_table[diff])
                else:
                    # [G] Encode using the fixed-length code for this symbol.
                    # Implementation detail required.
                    pass
                    
    return pack_bits(output_bits)

def decompress_volume(compressed_data: bytes, X: int, Y: int, Z: int, mean_v: float, a1: float, a2: float, a3: float, huffman_table: dict) -> Array3D:
    """
    Decompress a 3D volume.
    """
    volume = zeros((X, Y, Z))
    bits = unpack_bits(compressed_data)
    bit_iterator = iter(bits)
    
    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                # Calculate prediction
                v1 = 0 if x == 0 else volume[x-1, y, z] - mean_v
                v2 = 0 if y == 0 else volume[x, y-1, z] - mean_v
                v3 = 0 if z == 0 else volume[x, y, z-1] - mean_v
                
                pred = a1 * v1 + a2 * v2 + a3 * v3
                
                # Decode residual
                # [G] Look up bits in Huffman table to find residual.
                # For fixed-length codes, read fixed number of bits.
                residual = decode_residual(bit_iterator, huffman_table)
                
                volume[x, y, z] = pred + residual
                
    return volume
```

### Edge Cases and Pitfalls
-   **Causality:** The predictor must only use previously processed voxels ($x-1, y, z$), ($x, y-1, z$), and ($x, y, z-1$) so the decoder can generate identical predictions [S1, unofficial].
-   **Non-stationarity:** The optimal predictor assumes wide-sense stationarity. Real volume data is not stationary, making global statistics suboptimal [S1, unofficial].
-   **Memory vs Speed:** The algorithm scans the volume four times (two for statistics, one for differences, one for encoding) to avoid storing all difference values, which would be memory-intensive [S1, unofficial].
-   **Boundary Handling:** Neighbors at the boundary ($x=0, y=0, z=0$) are treated as 0 in the mean-subtracted space [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The algorithm requires 4 full scans of the volume. The covariance calculation is $O(XYZ)$ with a constant factor related to the number of lag combinations (27 lags). Solving the $3 \times 3$ linear system is $O(1)$. Huffman coding is $O(N \log N)$ or $O(N)$ depending on implementation, where $N$ is the number of voxels.
-   **Memory Complexity:** $O(1)$ auxiliary memory if differences are not stored (requiring 4 scans). If differences are stored, $O(XYZ)$. The Huffman table size is proportional to the alphabet size $L$ (131,071 for 16-bit) [S1, unofficial].
-   **Performance:** Achieves approximately 50% lossless compression on test datasets [S1, unofficial]. On the `3dhead.vox` dataset (256x256x109), it achieved 48.5% compression compared to 36.1% for gzip and 33.8% for zip [S1, unofficial].
-   **Strengths:** Removes spatial redundancy effectively; optimal coefficients compensate for non-isotropic sampling; simple implementation [S1, unofficial].
-   **Limitations:** Assumes stationarity; slow due to multiple scans; fixed-length codes for rare symbols may be inefficient if not optimized [S1, unofficial].

## Background
-   **Origin:** Introduced by James Edwin Fowler and Roni Yagel in January 1995 at the Proceedings of the 1994 symposium on Volume Visualization [S1, unofficial].
-   **History:** The method was developed to address the high storage requirements of volume data in biomedical and scientific applications where lossless compression is critical. It combines DPCM and Huffman coding to outperform generic compression tools [S1, unofficial].
-   **Usage:** Used for archiving medical imaging data (MRI, CT) and other scientific volume datasets [S1, unofficial].

## Sources
[S1] (PDF) Lossless Compression of Volume Data - ResearchGate, https://www.researchgate.net/publication/221252510_Lossless_Compression_of_Volume_Data, fetched 2026-11-03 (unofficial)
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
