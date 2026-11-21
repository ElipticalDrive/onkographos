# CPU Radix Sort for Face Centroids

Also called: Floating Point Radix Sort, LSD Radix Sort

## Overview
An out-of-place sorting algorithm used on the CPU to sort face centroids for volume rendering splatting, achieving linear time complexity at the expense of auxiliary memory [S1, unofficial]. It differs from its sibling [Logarithmic Distance Bucket Sort](14468-logarithmic-distance-bucket-sort.md) by using exact integer comparison rather than distance-based bucketing, making it suitable for general-purpose sorting where precise ordering is required without depth-peeling assumptions.

## How it works
**Inputs:** Array of $N$ face centroids (floating-point values, typically distances).
**Outputs:** Array of $N$ sorted face centroids (ascending order).
**Auxiliary Data:** An auxiliary array of size $N$ and a count array of size 256.

**Data Structures:**
1.  `input`: Array of $N$ 32-bit IEEE 754 floating-point numbers.
2.  `output`: Array of $N$ 32-bit unsigned integers (sorted keys).
3.  `counts`: Array of 256 unsigned integers (histogram for counting sort).

**Steps:**
1.  **Transformation:** Convert each floating-point number in `input` to a 32-bit unsigned integer representation that preserves numerical order. This handles the sign bit inversion required for negative numbers in IEEE 754.
2.  **Pass 1 (LSD):** Perform a counting sort on bits 0–7 of the transformed integers.
    *   Initialize `counts` to zero.
    *   Iterate through `input`, extract the lower 8 bits of each transformed value, and increment the corresponding index in `counts`.
    *   Compute prefix sums (cumulative counts) in `counts` to determine output positions.
    *   Distribute elements from `input` to `output` based on the computed positions.
3.  **Pass 2:** Perform a counting sort on bits 8–15. Use `output` from Pass 1 as input and write to `input` (or a second auxiliary buffer, swapping roles).
4.  **Pass 3:** Perform a counting sort on bits 16–23.
5.  **Pass 4:** Perform a counting sort on bits 24–31 (MSD).
6.  **Final Output:** The result is in the array written by Pass 4.

**Formulas:**
*   **Float to Int Transformation:**
    $$ f_{int} = f \oplus ((-(f \gg 31)) \mid 0x80000000) $$
    Where $f$ is the bit-cast of the 32-bit float to an unsigned integer. This transformation ensures that negative floats sort before positive floats and maintain relative order [S1, unofficial].
*   **Bit Extraction:**
    For block $k$ (where $k \in \{0, 1, 2, 3\}$), the key for counting sort is:
    $$ key = (f_{int} \gg (8 \times k)) \ \& \ 0xFF $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def float_to_uint32(f_bits):
    """Convert IEEE 754 float bits to signed-magnitude-like uint32 for sorting."""
    # f_bits is the raw 32-bit integer representation of the float
    sign_mask = -(f_bits >> 31)  # 0xFFFFFFFF if negative, 0x00000000 if positive
    return f_bits ^ (sign_mask | 0x80000000)

def cpu_radix_sort_face_centroids(centroids_float):
    """
    Sorts an array of floating-point face centroids using LSD Radix Sort.
    centroids_float: List of floats.
    Returns: List of sorted floats.
    """
    n = len(centroids_float)
    if n == 0:
        return []

    # Step 1: Convert floats to sortable integers
    ints = [float_to_uint32(float_to_bits(c)) for c in centroids_float]
    
    # Auxiliary arrays
    output = [0] * n
    counts = [0] * 256
    
    # Current data source and destination
    src = ints
    dst = output

    # Step 2-5: Four passes for 8-bit blocks
    for k in range(4):
        # Reset counts
        for i in range(256):
            counts[i] = 0
            
        # Count occurrences
        for i in range(n):
            key = (src[i] >> (8 * k)) & 0xFF
            counts[key] += 1
            
        # Compute prefix sums (cumulative counts)
        # counts[i] will store the starting index for key i in the output
        total = 0
        for i in range(256):
            temp = counts[i]
            counts[i] = total
            total += temp
            
        # Distribute elements to dst
        for i in range(n):
            key = (src[i] >> (8 * k)) & 0xFF
            pos = counts[key]
            dst[pos] = src[i]
            counts[key] += 1
            
        # Swap src and dst for next pass
        src, dst = dst, src

    # Convert back to floats
    sorted_floats = [bits_to_float(d) for d in src]
    return sorted_floats

def float_to_bits(f):
    import struct
    return struct.unpack('I', struct.pack('f', f))[0]

def bits_to_float(i):
    import struct
    return struct.unpack('f', struct.pack('I', i))[0]
```

**Parameters:**
*   Block size: 8 bits (4 passes total) [S1, unofficial].
*   Count array size: 256 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **NaN Values:** The transformation assumes standard IEEE 754 numbers excluding NaNs. NaNs may not sort correctly or consistently [S1, unofficial].
*   **Precision:** The transformation relies on the specific bit layout of IEEE 754. Non-compliant floating-point representations will fail [S1, unofficial].
*   **Memory:** The algorithm is out-of-place, requiring $O(N)$ auxiliary memory [S1, unofficial].

## Subtopics

### [LSD Block Sorting (Radix Sort)](6217-lsd-block-sorting-radix-sort.md)
LSD Block Sorting is a CPU-based radix sort algorithm that sorts 32-bit integers by processing them in 8-bit blocks from least significant to most significant. It solves the problem of ordering face centroids for volume rendering splatting by achieving linear time complexity through five sequential counting sort passes. It differs from its sibling [Logarithmic Distance Bucket Sort](14468-logarithmic-distance-bucket-sort.md) by using exact integer comparison rather than distance-based bucketing, making it suitable for general-purpose sorting where precise ordering is required without depth-peeling assumptions.

## Operating characteristics
*   **Time Complexity:** Linear $O(N)$, where $N$ is the number of faces. The constant factor is small due to the fixed number of passes (4) and small radix (256) [S1, unofficial].
*   **Memory Complexity:** $O(N)$ auxiliary space for the output array and $O(1)$ for the count array (size 256) [S1, unofficial].
*   **Performance:** Sorts upwards of 15 million faces per second on the tested hardware [S1, unofficial].
*   **Comparison:** Significantly faster than shellsort (584 ms), heapsort (507 ms), and quicksort (281 ms) for sorting one million floating-point numbers (radixsort took 64 ms) [S1, unofficial].
*   **Strengths:** Predictable performance independent of data distribution; much faster than comparison-based sorts for large $N$ [S1, unofficial].
*   **Limitations:** Requires auxiliary memory; not in-place; assumes IEEE 754 format [S1, unofficial].
*   **When to use:** When sorting large numbers of face centroids on the CPU where GPU sorting is unavailable or disabled [S1, unofficial].
*   **When not to use:** When memory is extremely constrained (due to auxiliary array requirement) or when the data is already nearly sorted and a specialized adaptive sort might be preferred (though the source notes that re-sorting is generally more efficient than adaptive sorts for this use case due to viewing direction changes) [S1, unofficial].

## Background
Originated in the context of the K-Buffer and its applications to volume rendering [S1, unofficial]. The method was selected over other common sorting algorithms (shellsort, heapsort, quicksort, mergesort) because re-sorting face centroids with a fast sort is more efficient in practice than using an algorithm optimized for re-sorting previously sorted sequences, as the ordering can change significantly from frame to frame due to viewing direction changes [S1, unofficial]. The specific float-to-int transformation is based on established techniques for sorting IEEE 754 floats as integers [S1, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
