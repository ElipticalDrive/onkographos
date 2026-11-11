# Burrows-Wheeler Transform


## Overview
The Burrows-Wheeler Transform (BWT) is a reversible permutation scheme that re-sorts a block of data according to its local context, organizing it into a form that is highly compressible for subsequent entropy coding [S1, S2]. Unlike the linear decorrelation transforms in its family (such as Wavelets or KLT), BWT does not change the data values but rearranges their positions to group identical or similar symbols together, thereby increasing redundancy [S1, S2]. It is distinguished by its ability to create long runs of identical characters, which are efficiently handled by Run-Length Encoding (RLE) or Move-to-Front (MTF) transforms before entropy coding [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A block of $N$ data values (typically treated as a string of bytes or symbols) from the volume data.
*   **Output:** A permuted block of $N$ values and an index $I$ indicating the position of the original block within the sorted rotations.

**Data Structures:**
*   **Input Block:** $S = s_0, s_1, \dots, s_{N-1}$.
*   **Rotation Matrix:** A conceptual $N \times N$ matrix where each row is a cyclic rotation of $S$.
*   **Sorted Indices:** An array $L$ of size $N$ containing the last character of each sorted rotation.
*   **Index:** An integer $I \in [0, N-1]$ pointing to the row in the sorted matrix that corresponds to the original input string.

**Steps:**
1.  **Generate Rotations:** Create all $N$ cyclic rotations of the input string $S$. The $i$-th rotation is $S[i:] + S[:i]$.
2.  **Sort Rotations:** Sort the $N$ rotations lexicographically. Let the sorted rotations be $R_0, R_1, \dots, R_{N-1}$.
3.  **Extract Last Column:** Construct the output string $L$ by taking the last character of each sorted rotation. $L[i]$ is the last character of $R_i$.
4.  **Identify Original Index:** Find the index $I$ such that $R_I$ is identical to the original input string $S$.
5.  **Output:** Return $L$ and $I$.

**Inverse Transform Steps:**
1.  **Count Sort:** Create a frequency count of each character in $L$.
2.  **Construct First Column:** Generate the first column $F$ of the sorted rotation matrix by sorting the characters in $L$.
3.  **Map Occurrences:** For each character, map its $k$-th occurrence in $L$ to its $k$-th occurrence in $F$. This creates a permutation vector $T$ (often called the "Next" array) where $T[i]$ is the index in $L$ of the character that precedes $L[i]$ in the original cyclic string.
4.  **Reconstruct:** Start at index $I$. Output $L[I]$, then update $I = T[I]$. Repeat $N$ times to reconstruct the original string $S$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bwt_transform(data: list) -> tuple:
    """
    Performs the Burrows-Wheeler Transform on a list of comparable items.
    Returns (transformed_list, original_index).
    """
    n = len(data)
    if n == 0:
        return [], 0

    # Step 1: Generate all cyclic rotations
    # We store indices to avoid copying large data blocks repeatedly
    rotations = list(range(n))
    
    # Step 2: Sort rotations lexicographically
    # Comparison function for rotation starting at index i
    def rotation_key(i):
        # Returns a tuple representing the rotation starting at i
        # This is a conceptual representation; efficient implementations
        # use suffix arrays or similar structures.
        return tuple(data[(i + j) % n] for j in range(n))

    rotations.sort(key=rotation_key)

    # Step 3: Extract last column and find original index
    transformed = []
    original_index = -1
    
    for idx in rotations:
        # The last character of the rotation starting at idx
        last_char = data[(idx - 1) % n]
        transformed.append(last_char)
        
        # Check if this rotation is the original string
        # The original string starts at index 0.
        # A rotation starting at idx is the original string if idx == 0.
        if idx == 0:
            original_index = len(transformed) - 1

    return transformed, original_index

def bwt_inverse(transformed: list, original_index: int) -> list:
    """
    Reverses the Burrows-Wheeler Transform.
    """
    n = len(transformed)
    if n == 0:
        return []

    # Step 1: Count frequencies
    # Assuming alphabet is known or derived from unique elements
    unique_chars = sorted(set(transformed))
    char_to_idx = {c: i for i, c in enumerate(unique_chars)}
    num_chars = len(unique_chars)
    
    count = [0] * num_chars
    for c in transformed:
        count[char_to_idx[c]] += 1
        
    # Step 2: Construct cumulative counts to determine start positions in F
    # F is the sorted version of transformed
    # We need to map occurrences in L to occurrences in F
    # T[i] gives the index in L of the character preceding L[i] in the original string
    
    # Calculate starting positions for each character in the sorted array F
    start_pos = [0] * num_chars
    current_pos = 0
    for i in range(num_chars):
        start_pos[i] = current_pos
        current_pos += count[i]
        
    # Build the T array (Next array)
    # We need to track how many times we've seen each character so far
    occ = [0] * num_chars
    T = [0] * n
    
    for i in range(n):
        c_idx = char_to_idx[transformed[i]]
        # The position in F for this specific occurrence
        pos_in_f = start_pos[c_idx] + occ[c_idx]
        T[pos_in_f] = i
        occ[c_idx] += 1
        
    # Step 3: Reconstruct the string
    result = []
    curr = original_index
    for _ in range(n):
        result.append(transformed[curr])
        curr = T[curr]
        
    return result
```

**Parameters:**
*   **Block Size:** The size of the data block $N$ processed by BWT. Larger blocks generally yield better compression due to longer context, but increase memory usage and transform time. Not given in the sources.
*   **Alphabet Size:** The number of unique symbols in the data. Affects the complexity of the sorting and mapping steps.

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Storing all rotations explicitly requires $O(N^2)$ space. Efficient implementations use suffix arrays or suffix trees to achieve $O(N)$ space and $O(N \log N)$ or $O(N)$ time complexity. [G]
*   **Sorting Stability:** The sort must be stable to ensure deterministic behavior, although the inverse transform relies on the "first occurrence" mapping which handles stability implicitly if implemented correctly via the T-array construction.
*   **Small Blocks:** For very small blocks, the overhead of the transform may outweigh the compression gains.

## Subtopics

### Move-to-Front (MTF) Transform
Often applied after BWT to further improve compressibility by converting frequent symbols into small integers. [Link to MTF if available, otherwise note as common companion].

### Run-Length Encoding (RLE)
Applied after MTF to compress runs of identical values. [Link to RLE if available].

### Entropy Coding
Huffman or Arithmetic coding applied to the final output. [Link to Entropy Coding if available].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ or $O(N)$ depending on the sorting algorithm and data structure used (e.g., Suffix Array construction). [G]
*   **Memory Complexity:** $O(N)$ for efficient implementations using Suffix Arrays. Naive implementations are $O(N^2)$. [G]
*   **Strengths:** Excellent at creating runs of identical characters, which are highly compressible with RLE and MTF. Reversible without loss. [S1, S2]
*   **Limitations:** Does not decorrelate data values like linear transforms (Wavelets, KLT). Performance depends heavily on the subsequent entropy coding stage. [S1, S2]
*   **When to Use:** When the data has strong local context redundancy and is suitable for block-based compression. [S1, S2]
*   **When Not to Use:** When multi-resolution access is required (BWT is not inherently multi-resolution like Wavelets). [S1, S2]

## Background
The Burrows-Wheeler Transform was introduced by Michael Burrows and David J. Wheeler. It represents the basis for a whole family of compression algorithms, most notably the bzip2 compressor. In the context of volume rendering, Komma et al. [KFDB07] compared the compressibility of BWT against other transforms for volume data compression. [S1, S2]

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
