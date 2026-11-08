# Burrows-Wheeler Transform

## Overview
The Burrows-Wheeler Transform (BWT) is a reversible permutation scheme that re-sorts a block of data according to its local context, organizing it into a compression-friendly format suitable for subsequent entropy coding [S1, S2]. It distinguishes itself from linear decorrelation transforms like the Karhunen-Loeve Transform (KLT) or Discrete Cosine Transform (DCT) by relying on sorting-based context modeling rather than basis function projection [S1, S2].

## How it works
**Inputs and Outputs:**
- Input: A block of $N$ symbols (e.g., volume data values) from a finite alphabet $\Sigma$.
- Output: A transformed block of $N$ symbols and an index $I$ indicating the original position of the first symbol in the sorted rotation matrix.

**Data Structures:**
- **Rotations:** A conceptual $N \times N$ matrix where each row $i$ is the cyclic right-rotation of the input string by $i$ positions.
- **Sorted Matrix:** The rows of the rotation matrix sorted lexicographically.
- **Index ($I$):** An integer $0 \le I < N$ storing the row number in the sorted matrix that corresponds to the original unrotated input string.

**Steps:**
1. **Generate Rotations:** Create all $N$ cyclic rotations of the input string $S$. Let $S^{(i)}$ denote the rotation starting at index $i$.
2. **Sort:** Sort all $N$ rotations lexicographically to form the sorted matrix $M$.
3. **Extract Transform:** The BWT output $L$ is the last column of $M$.
4. **Record Index:** The index $I$ is the row number in $M$ where the original string $S$ appears (i.e., where the rotation is identical to $S$).

**Inverse Transform:**
1. **Count:** Count the occurrences of each symbol in $L$ to determine the starting row in $M$ for each symbol type.
2. **First Column:** Construct the first column $F$ of $M$ by sorting $L$.
3. **Map:** Use the "Last-to-First" mapping property. For each position $i$ in $L$, find the corresponding position in $F$ that represents the same instance of the symbol. This allows reconstruction of the original string by traversing the mapping from index $I$.

**Parameters:**
- Block size $N$: Not given in the sources.

**Edge Cases and Pitfalls:**
- The transform is reversible only if the index $I$ is stored.
- Sorting complexity can be high for large blocks; efficient implementations use suffix arrays or similar structures [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bwt_transform(input_string):
    n = len(input_string)
    # Generate all cyclic rotations
    rotations = []
    for i in range(n):
        rotated = input_string[i:] + input_string[:i]
        rotations.append((rotated, i))
    
    # Sort rotations lexicographically
    rotations.sort(key=lambda x: x[0])
    
    # Extract the last column
    last_column = [rot[0][-1] for rot in rotations]
    
    # Find the index of the original string
    original_index = -1
    for i, (rot, orig_idx) in enumerate(rotations):
        if orig_idx == 0:
            original_index = i
            break
            
    return last_column, original_index

def bwt_inverse(last_column, original_index):
    n = len(last_column)
    # Create pairs of (symbol, index)
    indexed_chars = list(enumerate(last_column))
    
    # Sort by symbol to reconstruct the first column order
    # This effectively builds the mapping from last to first
    sorted_chars = sorted(indexed_chars, key=lambda x: x[1])
    
    # Reconstruct the string
    result = []
    current_index = original_index
    for _ in range(n):
        symbol = last_column[current_index]
        result.append(symbol)
        # Move to the next character in the original string
        # Find the position of the current symbol in the sorted list
        # that corresponds to the current position in the last column
        # This requires a stable mapping or rank calculation
        # For simplicity, we use the property that the k-th occurrence 
        # of a char in L maps to the k-th occurrence in F
        # We need to track ranks. A full implementation requires 
        # counting occurrences to build the LF mapping.
        pass # Placeholder for complex LF mapping logic not fully detailed in sources
    return "".join(result)
```

## Subtopics

### [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md)
Uses triadic wavelets for multi-resolution analysis.

### [Rational Bézier Deblocking Filter](832-rational-b-zier-deblocking-filter.md)
Filters artifacts using rational Bézier curves.

### [Vector Quantization](907-vector-quantization.md)
Compresses data by mapping vectors to a finite set of codewords.

### [Differential Data Stream (DDS) File Format](1754-differential-data-stream-dds-file-format.md)
Stores differences between data points.

### [Zerobit Encoding Scheme](1918-zerobit-encoding-scheme.md)
Encodes zero-valued regions efficiently.

### [Laplacian Pyramid](2250-laplacian-pyramid.md)
Multi-resolution representation using pyramid decomposition.

### [Random Access of Compressed Data (Westermann's Method)](2269-random-access-of-compressed-data-westermann-s-method.md)
Enables selective access to compressed volume data.

### [Predictive and Differential Coding](4393-predictive-and-differential-coding.md)
Predicts values based on neighbors to reduce redundancy.

### [Standard Image Compression Formats](4394-standard-image-compression-formats.md)
Covers JPEG, PNG, etc.

### [Structural and Geometric Compression](4959-structural-and-geometric-compression.md)
Compresses based on geometric primitives.

### [Splat parameter encoding and codec selection](5019-splat-parameter-encoding-and-codec-selection.md)
Encodes splat-based rendering parameters.

### [Texture Format Compression](5623-texture-format-compression.md)
Compresses texture data for GPU use.

### [Time-Varying Volume Compression Strategies](5714-time-varying-volume-compression-strategies.md)
Compresses dynamic volume data over time.

### [Quantization](5935-quantization.md)
Reduces precision of data values.

### [Transform-based Block Coding](6882-transform-based-block-coding.md)
Applies transforms to blocks of data.

### [Wavelet Kernel Selection](7035-wavelet-kernel-selection.md)
Chooses appropriate wavelet filters.

## Operating characteristics
- **Time Complexity:** Sorting $N$ rotations takes $O(N \log N)$ or $O(N^2)$ depending on implementation. Inverse transform is $O(N)$ with proper data structures [G].
- **Memory Complexity:** Requires storing the sorted matrix or suffix array, $O(N)$ space [G].
- **Measured Performance:** Komma et al. [KFDB07] compared BWT compressibility against other transforms for volume data, but specific numerical results are not provided in the sources.
- **Strengths:** Excellent at grouping identical symbols together, which benefits entropy coders like Huffman or Arithmetic coding.
- **Limitations:** High computational cost for large blocks due to sorting. Not inherently multi-resolution like wavelets.
- **When to use:** When data has strong local context redundancy and block sizes are manageable.
- **When not to use:** For very large blocks where sorting is prohibitive, or when random access is required without additional indexing structures.

## Background
The Burrows-Wheeler Transform was introduced by Burrows and Wheeler as a basis for a family of compression algorithms [S1, S2]. It has been applied in the context of compressed GPU-based Direct Volume Rendering (DVR) [S1, S2]. Komma et al. [KFDB07] specifically evaluated its compressibility for volume data compared to other transforms.

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
