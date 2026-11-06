# Entropy Encoding

Also called: Arithmetic Coding, Entropy Coding, Huffman Coding, Lossless Compression, 熵编码

## Overview
Entropy encoding is a lossless data compression technique that reduces the size of volume data by exploiting the statistical distribution of voxel values or coefficients [S1][S2][S3]. It assigns shorter codewords to frequent values and longer codewords to infrequent ones, achieving high compression ratios [S1][S2][S3]. Unlike fixed-length encodings, it is variable-length, which hinders quick random access reconstruction, making it less suitable for direct massively parallel GPU rendering without pre-decoding [S2][S3]. It is frequently used in compressed volume rendering, often in hybrid schemes combined with Run-Length Encoding (RLE) to mitigate decoding overhead [S2][S3].

## How it works
**Inputs:** A stream of voxel values or transformed coefficients (e.g., wavelet coefficients) [S1][S2].
**Outputs:** A binary bitstream representing the compressed data [S1].

**Data Structures:**
1. **Frequency Table:** A map or array storing the count or probability of each unique value in the input stream [S1][S2].
2. **Codeword Table:** A map from each unique value to its assigned variable-length binary codeword [S1][S2].

**Steps:**
1. **Analyze Distribution:** Scan the input data to count the frequency of each distinct value [S1][S2].
2. **Assign Codewords:** Generate variable-length codewords based on frequency. Frequent values receive shorter codewords; infrequent values receive longer codewords [S1][S2]. Common methods include Huffman coding or arithmetic coding [S1][S2][S4, unofficial].
3. **Encode Stream:** Replace each value in the input stream with its corresponding codeword and write the resulting bits to the output bitstream [S1].

**Formulas:**
The Shannon entropy lower bound is $H = - \sum_x p_x(x) \log p_x(x)$, where $p_x(x)$ is the underlying distribution of $x$, providing the minimum expected number of bits per element [S1]. The bit rate for a dataset $X = (x_1, \dots, x_N)$ is $R(X) = -\frac{1}{N} \sum_{i=1}^N \log p_x(x_i)$ [S1]. Each $x_i$ is encoded to a bit string of length $-\log p_x(x_i)$ bits, achievable arbitrarily closely on average by an arithmetic coder [S1].

**Parameters:**
*   **Codeword Assignment Strategy:** Huffman coding or arithmetic coding [S1][S2][S4, unofficial].
*   **Combination with RLE:** Fixed-length Huffman coders are sometimes combined with RLE to accelerate decoding [S2][S3].

**Edge Cases and Pitfalls:**
*   **Random Access:** Variable-length encoding makes it difficult to quickly reconstruct an arbitrary sequence of data, which is critical in massively parallel GPU settings [S2][S3].
*   **Decoding Complexity:** Entropy encoding implies significantly more computational work during decoding compared to fixed-length schemes [S2][S3].
*   **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def entropy_encode(input_values):
    # Step 1: Analyze distribution
    N = len(input_values)
    freq = {}
    for x in input_values:
        freq[x] = freq.get(x, 0) + 1
        
    # Step 2: Assign codewords based on frequency
    # [G] Standard Huffman tree construction or arithmetic coding interval mapping
    codeword_table = {}
    for x, count in freq.items():
        p = count / N
        # [G] Variable-length codeword assigned; length approximates -log2(p)
        codeword_table[x] = [0] # [G] Standard codeword generation omitted; uses Huffman or arithmetic coding [G]
        
    # Step 3: Encode stream
    output_bits = []
    for x in input_values:
        output_bits.extend(codeword_table[x])
    return output_bits
```

## Subtopics

### Golomb-Rice Encoding
An entropy encoder favored over arithmetic and Huffman coding in some volume rendering contexts because it offers better random access capabilities and lower computational complexity [S2][S3]. It is used when fast random access for reconstruction is required [S2].

## Operating characteristics
*   **Time Complexity:** Decoding requires significantly more computational work than fixed-length encoding [S2][S3].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** High compression ratio due to exploitation of data distribution [S1][S2][S3].
*   **Limitations:** Variable-length nature hinders quick random access reconstruction, which is critical in massively parallel GPU settings [S2][S3]. Implies more work during decoding [S2][S3].
*   **When to use:** When high compression ratios are needed for on-disk storage and decoding can be performed before rendering time, or when combined with RLE to accelerate decoding [S2][S3].
*   **When not to use:** When fast random access is required directly on the GPU during rendering, in which case Golomb-Rice or fixed-length schemes may be preferred [S2][S3].

## Background
Entropy encoding has been widely used in compressed volume rendering since early applications [S2][S3]. Huffman encoding and arithmetic encoding are frequently used, often combined with RLE [S2][S3]. Research has focused on finding encoding schemes with fast random access for reconstruction, such as Golomb-Rice encoding, to address the limitations of traditional entropy encoders in parallel settings [S2][S3]. A survey by Komma et al. benchmarks and analyzes lossless compression encodings for compressed volume data, including entropy encodings [S2][S3].

## Sources
[S1] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S4] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
