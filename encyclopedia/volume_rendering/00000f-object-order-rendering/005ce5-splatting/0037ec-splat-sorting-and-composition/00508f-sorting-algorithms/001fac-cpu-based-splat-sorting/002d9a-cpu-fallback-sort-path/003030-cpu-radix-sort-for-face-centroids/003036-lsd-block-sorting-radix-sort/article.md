# LSD Block Sorting (Radix Sort)
Also called: LSD counting sort

## Overview
LSD Block Sorting is a CPU-based radix sort algorithm that sorts 32-bit integers by processing them in 8-bit blocks from least significant to most significant. It solves the problem of ordering face centroids for volume rendering splatting by achieving linear time complexity through five sequential counting sort passes. It differs from its sibling [Logarithmic Distance Bucket Sort](14468-logarithmic-distance-bucket-sort.md) by using exact integer comparison rather than distance-based bucketing, making it suitable for general-purpose sorting where precise ordering is required without depth-peeling assumptions.

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ 32-bit unsigned integers (face centroids or keys).
- Output: The same array sorted in ascending order.
- Auxiliary: An auxiliary array of size $N$ used to store intermediate results.

**Data Structures:**
- `input`: Array of $N$ 32-bit integers.
- `output`: Array of $N$ 32-bit integers.
- `counts`: Array of 256 integers (for each 8-bit block, values 0-255).

**Steps:**
1. **Pass 1 (Bits 0-7):** Perform a counting sort on the least significant 8 bits (bits 0-7) of all elements in `input`. Store the sorted results in `output`.
2. **Pass 2 (Bits 8-15):** Perform a counting sort on bits 8-15 of the elements in `output` (which are sorted by bits 0-7). Store the results back into `input`.
3. **Pass 3 (Bits 16-23):** Perform a counting sort on bits 16-23 of the elements in `input`. Store the results in `output`.
4. **Pass 4 (Bits 24-31):** Perform a counting sort on bits 24-31 of the elements in `output`. Store the results back into `input`.

**Counting Sort Logic (per pass):**
For each pass, extract the relevant 8-bit block from each number. Use these values as indices to count occurrences. Compute cumulative counts to determine positions. Place elements into the target array based on these positions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lsd_block_sort(input_array, n):
    output_array = [0] * n
    counts = [0] * 256
    
    # Pass 1: Bits 0-7
    # Count occurrences of bits 0-7
    for i in range(n):
        key = (input_array[i] >> 0) & 0xFF
        counts[key] += 1
    
    # Cumulative count
    for i in range(1, 256):
        counts[i] += counts[i-1]
    
    # Place elements in output
    for i in range(n-1, -1, -1):
        key = (input_array[i] >> 0) & 0xFF
        output_array[counts[key]-1] = input_array[i]
        counts[key] -= 1
    
    # Copy output to input for next pass
    input_array = output_array[:]
    
    # Pass 2: Bits 8-15
    counts = [0] * 256
    for i in range(n):
        key = (input_array[i] >> 8) & 0xFF
        counts[key] += 1
    
    for i in range(1, 256):
        counts[i] += counts[i-1]
    
    for i in range(n-1, -1, -1):
        key = (input_array[i] >> 8) & 0xFF
        output_array[counts[key]-1] = input_array[i]
        counts[key] -= 1
    
    input_array = output_array[:]
    
    # Pass 3: Bits 16-23
    counts = [0] * 256
    for i in range(n):
        key = (input_array[i] >> 16) & 0xFF
        counts[key] += 1
    
    for i in range(1, 256):
        counts[i] += counts[i-1]
    
    for i in range(n-1, -1, -1):
        key = (input_array[i] >> 16) & 0xFF
        output_array[counts[key]-1] = input_array[i]
        counts[key] -= 1
    
    input_array = output_array[:]
    
    # Pass 4: Bits 24-31
    counts = [0] * 256
    for i in range(n):
        key = (input_array[i] >> 24) & 0xFF
        counts[key] += 1
    
    for i in range(1, 256):
        counts[i] += counts[i-1]
    
    for i in range(n-1, -1, -1):
        key = (input_array[i] >> 24) & 0xFF
        output_array[counts[key]-1] = input_array[i]
        counts[key] -= 1
    
    return output_array
```

**Parameters:**
- Block size: 8 bits.
- Number of passes: 4 (plus one initial count pass if implemented as 5 total traversals as described in sources, though the pseudocode above combines count and place in one pass per block for efficiency, the source describes 5 passes total: 1 for counting all blocks, then 4 for sorting each block). The source states "five passes through the array are necessary". This implies one pass to count all 4 blocks simultaneously, then 4 passes to sort. The pseudocode above is a standard stable radix sort implementation. To match the source's "five passes" description exactly:
    1. Pass 1: Count occurrences for bits 0-7, 8-15, 16-23, 24-31 for all elements.
    2. Pass 2: Sort bits 0-7 using counts, write to auxiliary.
    3. Pass 3: Sort bits 8-15 using counts on auxiliary, write to original.
    4. Pass 4: Sort bits 16-23 using counts, write to auxiliary.
    5. Pass 5: Sort bits 24-31 using counts, write to original.

**Edge Cases:**
- Empty array: Return immediately.
- Precision: Uses 32-bit integers, so no floating-point precision issues.

## Subtopics

### [Logarithmic Distance Bucket Sort](14468-logarithmic-distance-bucket-sort.md)
A sorting algorithm for voxels based on their distance from the camera, using logarithmic distance mapping into fixed buckets to achieve back-to-front ordering for depth peeling or alpha blending.

## Operating characteristics
- **Time Complexity:** Linear $O(N)$, where $N$ is the number of elements. The algorithm performs a constant number of passes (5) through the array [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the auxiliary array.
- **Performance:** A C++ implementation can sort upwards of 15 million faces per second [S1, unofficial].
- **Strengths:** Linear time complexity, simple implementation, no machine-level optimizations required for baseline performance.
- **Limitations:** Requires auxiliary memory, not in-place. Performance can be improved with machine-level optimizations [S1, unofficial].

## Background
The LSD Block Sorting method is described in Steve Callahan's MS thesis, "THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING" [S1, unofficial]. It is used as a CPU fallback sort path for sorting face centroids in volume rendering applications, specifically within the context of the k-buffer algorithm.

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
