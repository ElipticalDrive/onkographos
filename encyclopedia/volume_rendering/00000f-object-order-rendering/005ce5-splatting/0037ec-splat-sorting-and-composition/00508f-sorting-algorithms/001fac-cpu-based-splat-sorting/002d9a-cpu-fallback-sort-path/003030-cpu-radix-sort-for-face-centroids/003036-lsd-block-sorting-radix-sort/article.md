# LSD Block Sorting (Radix Sort)
Also called: LSD counting sort

## Overview
LSD Block Sorting is a CPU-based radix sort that orders 32-bit integers by processing them in 8-bit blocks from least to most significant [S1, unofficial]. It solves the depth-ordering requirement for volume rendering splatting by achieving linear time complexity through sequential counting sort passes, differing from distance-based bucketing methods by relying on exact integer comparison [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ 32-bit unsigned integers representing face centroids or keys [G].
- Output: The array sorted in ascending order [G].
- Auxiliary: An auxiliary array of size $N$ used to store intermediate results [G].

**Data Structures:**
- `input`: Array of $N$ 32-bit integers [G].
- `output`: Array of $N$ 32-bit integers [G].
- `counts`: Array of 256 integers to store frequency for each 8-bit block value (0–255) [G].

**Steps:**
1. **Count Pass:** Traverse the input array once to count occurrences of all four 8-bit blocks (bits 0–7, 8–15, 16–23, 24–31) for every element [S1, unofficial].
2. **Sort Pass 1 (Bits 0–7):** Perform a stable counting sort on bits 0–7 of the input array, writing the results to the auxiliary array [S1, unofficial].
3. **Sort Pass 2 (Bits 8–15):** Perform a stable counting sort on bits 8–15 of the auxiliary array (input), writing the results back to the original input array [S1, unofficial].
4. **Sort Pass 3 (Bits 16–23):** Perform a stable counting sort on bits 16–23 of the input array, writing the results to the auxiliary array [S1, unofficial].
5. **Sort Pass 4 (Bits 24–31):** Perform a stable counting sort on bits 24–31 of the auxiliary array (input), writing the final sorted results back to the original input array [S1, unofficial].

**Counting Sort Logic (per pass):**
For each pass, extract the relevant 8-bit block from each number using bitwise operations: `key = (value >> shift) & 0xFF` [G]. Use these values as indices to accumulate frequencies in `counts` [G]. Compute cumulative sums over `counts` to determine final positions in the target array [G]. Place elements into the target array in reverse order to maintain stability [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lsd_block_sort(input_array, n):
    output_array = [0] * n
    counts = [0] * 256

    # Pass 1: Count occurrences for all 4 blocks
    for i in range(n):
        val = input_array[i]
        counts[(val >> 0) & 0xFF] += 1
        counts[(val >> 8) & 0xFF] += 1
        counts[(val >> 16) & 0xFF] += 1
        counts[(val >> 24) & 0xFF] += 1

    # Pass 2: Sort bits 0-7, write to output_array
    for i in range(256): counts[i] = 0
    for i in range(n):
        key = (input_array[i] >> 0) & 0xFF
        counts[key] += 1
    for i in range(1, 256):
        counts[i] += counts[i-1]
    for i in range(n-1, -1, -1):
        key = (input_array[i] >> 0) & 0xFF
        output_array[counts[key]-1] = input_array[i]
        counts[key] -= 1

    # Pass 3: Sort bits 8-15, write to input_array
    for i in range(256): counts[i] = 0
    for i in range(n):
        key = (output_array[i] >> 8) & 0xFF
        counts[key] += 1
    for i in range(1, 256):
        counts[i] += counts[i-1]
    for i in range(n-1, -1, -1):
        key = (output_array[i] >> 8) & 0xFF
        input_array[counts[key]-1] = output_array[i]
        counts[key] -= 1

    # Pass 4: Sort bits 16-23, write to output_array
    for i in range(256): counts[i] = 0
    for i in range(n):
        key = (input_array[i] >> 16) & 0xFF
        counts[key] += 1
    for i in range(1, 256):
        counts[i] += counts[i-1]
    for i in range(n-1, -1, -1):
        key = (input_array[i] >> 16) & 0xFF
        output_array[counts[key]-1] = input_array[i]
        counts[key] -= 1

    # Pass 5: Sort bits 24-31, write to input_array
    for i in range(256): counts[i] = 0
    for i in range(n):
        key = (output_array[i] >> 24) & 0xFF
        counts[key] += 1
    for i in range(1, 256):
        counts[i] += counts[i-1]
    for i in range(n-1, -1, -1):
        key = (output_array[i] >> 24) & 0xFF
        input_array[counts[key]-1] = output_array[i]
        counts[key] -= 1

    return input_array
```

**Parameters:**
- Block size: 8 bits [G].
- Number of passes: 5 total (1 counting pass + 4 sorting passes) [S1, unofficial].
- Bit shifts: 0, 8, 16, 24 [G].

**Edge Cases:**
- Empty array ($N=0$): Return immediately [G].
- Precision: Uses 32-bit unsigned integers, avoiding floating-point precision issues [G].
- Degenerate input: Handles duplicate keys correctly due to stable counting sort implementation [G].

## Subtopics

### [Logarithmic Distance Bucket Sort](14468-logarithmic-distance-bucket-sort.md)
A CPU-based sorting algorithm for volume rendering voxels that orders them back-to-front relative to the camera position. It solves the depth-sorting requirement for alpha blending or depth peeling by mapping voxel distances into fixed logarithmic buckets. It differs from Morton-order-based splat sorting (e.g., [SVRaster](52-svraster.md)) by relying solely on radial distance rather than ray-direction-dependent spatial locality, making it robust for arbitrary camera orientations but potentially less cache-friendly for static scenes [G].

## Operating characteristics
- **Time Complexity:** Linear $O(N)$, established by the constant five passes through the array [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the auxiliary array [G].
- **Performance:** A C++ implementation sorts upwards of 15 million faces per second [S1, unofficial].
- **Strengths:** Linear time complexity, simple implementation without machine-level optimizations, deterministic ordering [S1, unofficial].
- **Limitations:** Requires auxiliary memory, not in-place, performance can be improved with hardware-specific optimizations [S1, unofficial].

## Background
The algorithm is described in Steve Callahan's MS thesis, "THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING" [S1, unofficial]. It serves as a CPU fallback sort path for ordering face centroids in volume rendering applications, specifically within the k-buffer algorithm framework [S1, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
