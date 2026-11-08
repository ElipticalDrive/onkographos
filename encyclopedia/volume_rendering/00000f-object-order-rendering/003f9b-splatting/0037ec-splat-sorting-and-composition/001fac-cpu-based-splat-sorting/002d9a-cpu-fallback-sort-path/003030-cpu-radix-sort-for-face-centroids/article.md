# CPU Radix Sort for Face Centroids

Also called: Floating Point Radix Sort, LSD Radix Sort

## Overview
This method provides a CPU-side fallback for sorting face centroids in volume splatting when GPU compute sorting is unavailable, building on [CPU Fallback Sort Path](5586-cpu-fallback-sort-path.md). It achieves linear time complexity by converting IEEE 754 floating-point numbers to signed integers and performing a Least Significant Digit (LSD) radix sort using counting sort on 8-bit blocks. It is chosen over general-purpose sorts (quicksort, heapsort) or re-sorting-optimized algorithms (mergesort) because viewing direction changes significantly alter centroid ordering, making the overhead of adaptive sorts inefficient compared to the raw speed of radix sort [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ face centroids, represented as IEEE 754 single-precision (32-bit) floating-point numbers.
- Output: The same array, sorted in ascending order. An auxiliary array of size $N$ is used for intermediate storage.

**Data Structures:**
- `float* input`: Pointer to the array of floating-point centroids.
- `float* output`: Pointer to the auxiliary array of the same size.
- `unsigned int* counts`: A lookup table of size 256 (for 8-bit counting sort) to store frequency of each byte value.

**Transformation:**
Floating-point numbers must be converted to unsigned integers for integer-based radix sorting. The transformation preserves the ordering of IEEE 754 floats (excluding NaNs) by treating the sign bit appropriately [S1, unofficial].
The conversion function is:
$$ \text{fint}(f) = f \oplus ((-(f \gg 31)) \mid 0x80000000) $$
Where $f$ is the bit-cast of the float to an unsigned 32-bit integer. This maps negative floats to large unsigned integers and positive floats to small unsigned integers, maintaining correct numerical order [S1, unofficial].

**Algorithm Steps:**
The sort operates on four 8-bit blocks (bytes) of the 32-bit integer representation. It performs five passes total: one initial pass to prepare counts, followed by four sorting passes.

1.  **Initialize Counts:** For the current block (starting with bits 0-7), iterate through the input array. For each element, extract the relevant 8-bit byte. Increment the corresponding index in the `counts` array.
2.  **Compute Prefix Sums:** Transform the `counts` array into prefix sums. This determines the starting position for each value in the output array.
3.  **Distribute:** Iterate through the input array again. For each element, extract the byte, look up its position in the prefix sum array, place the element in the `output` array at that position, and increment the position counter.
4.  **Swap Buffers:** If this is not the final pass (bits 24-31), swap the roles of `input` and `output`. The `output` from this pass becomes the `input` for the next pass.
5.  **Repeat:** Repeat steps 1-4 for the next byte (bits 8-15, then 16-23, then 24-31).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_radix_sort_face_centroids(input_array, n):
    """
    Sorts an array of floats representing face centroids using LSD radix sort.
    input_array: list of floats
    n: number of elements
    Returns: sorted list of floats
    """
    # Auxiliary array for sorting
    output_array = [0.0] * n
    
    # Temporary array for integer conversion if needed, 
    # but we can convert on the fly or pre-convert. 
    # For clarity, we convert once.
    int_array = [float_to_uint32(x) for x in input_array]
    
    # We need a buffer for the final result if the number of passes is odd/even
    # The algorithm alternates between input and output buffers.
    # Pass 1 (bits 0-7): input -> output
    # Pass 2 (bits 8-15): output -> input
    # Pass 3 (bits 16-23): input -> output
    # Pass 4 (bits 24-31): output -> input
    
    # Current source buffer
    src = int_array
    dst = output_array
    
    # Perform 4 passes for 4 bytes
    for byte_idx in range(4):
        # 1. Count occurrences
        counts = [0] * 256
        for i in range(n):
            # Extract byte at byte_idx
            val = src[i]
            byte_val = (val >> (byte_idx * 8)) & 0xFF
            counts[byte_val] += 1
            
        # 2. Compute prefix sums
        # counts[i] will store the starting index for value i
        total = 0
        for i in range(256):
            temp = counts[i]
            counts[i] = total
            total += temp
            
        # 3. Distribute elements
        for i in range(n):
            val = src[i]
            byte_val = (val >> (byte_idx * 8)) & 0xFF
            pos = counts[byte_val]
            dst[pos] = val
            counts[byte_val] += 1
            
        # 4. Swap buffers for next pass
        # If this is the last pass, we don't need to swap back if dst is the final target
        # However, to keep logic uniform, we swap references.
        # If byte_idx is 3 (last pass), dst holds the final sorted data.
        if byte_idx < 3:
            src, dst = dst, src
            
    # If the number of passes resulted in the final data being in 'src' 
    # (because we swapped after the last logical step or didn't swap),
    # we need to ensure we return the correct buffer.
    # With 4 passes:
    # Pass 1: src->dst
    # Pass 2: dst->src
    # Pass 3: src->dst
    # Pass 4: dst->src
    # So final result is in 'src' if we swapped after each pass.
    # In the loop above, we swap *after* distribution.
    # Pass 1 ends: data in dst. Swap: src=dst, dst=old_src.
    # Pass 2 ends: data in dst (which is old_src). Swap: src=dst, dst=old_dst.
    # Pass 3 ends: data in dst. Swap: src=dst, dst=old_src.
    # Pass 4 ends: data in dst. No swap (loop ends).
    # So final result is in dst.
    
    # Convert back to floats
    result = [uint32_to_float(x) for x in dst]
    return result

def float_to_uint32(f):
    # Bit cast float to unsigned int
    # In Python, struct module is used for this, but for pseudocode:
    return struct.unpack('I', struct.pack('f', f))[0]

def uint32_to_float(u):
    return struct.unpack('f', struct.pack('I', u))[0]
```

**Parameters:**
- Block size: 8 bits. This allows the use of a small 256-entry count array, fitting in CPU cache [S1, unofficial].
- Passes: 4 passes for the 4 bytes of a 32-bit integer [S1, unofficial].

**Edge Cases and Pitfalls:**
- **NaN Values:** The transformation does not correctly order NaN values. The source notes that IEEE 754 floats excluding NaNs are properly ordered [S1, unofficial]. NaNs should be handled separately or filtered out before sorting.
- **Precision:** The method relies on the specific bit layout of IEEE 754. It is not portable to non-IEEE floating-point formats without modification.
- **Memory:** The algorithm is out-of-place, requiring auxiliary memory equal to the size of the input array [S1, unofficial].

## Subtopics

### LSD Block Sorting (Radix Sort)
This is the specific implementation variant described in the source. It sorts bits 0-7, 8-15, 16-23, and 24-31 sequentially using counting sort. It requires five passes through the array (one to count, four to sort) and establishes linear complexity. It is chosen for its ability to sort upwards of 15 million faces per second on the tested hardware [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear $O(N)$, where $N$ is the number of faces. The constant factor is small due to the fixed number of passes (4) and small bucket size (256) [S1, unofficial].
- **Memory Complexity:** $O(N)$ auxiliary space for the output array and $O(1)$ for the count array (size 256) [S1, unofficial].
- **Performance:** On the test hardware, this method sorts one million floating-point numbers in 64 ms. This is significantly faster than shellsort (584 ms), heapsort (507 ms), and quicksort (281 ms) [S1, unofficial]. The implementation can sort upwards of 15 million faces per second [S1, unofficial].
- **Strengths:** High throughput for large datasets; predictable performance; no recursion depth issues.
- **Limitations:** Requires auxiliary memory; not suitable for in-place constraints; performance may degrade if cache misses are high for large arrays; not adaptive to partially sorted data (unlike mergesort, which was found less efficient due to frequent full re-sorting needs) [S1, unofficial].
- **When to use:** When GPU sorting is unavailable and high throughput is required for dynamic scenes where viewing direction changes frequently [S1, unofficial].
- **When not to use:** When memory is extremely constrained; when the dataset is very small (overhead of setup may outweigh benefits); when NaN handling is critical without pre-filtering.

## Background
This method originates from the "K-Buffer and Its Applications to Volume Rendering" thesis by Steve Callahan (2006) [S1, unofficial]. It was developed as a CPU fallback for volume splatting rendering, where face centroids need to be sorted for correct depth ordering. The choice of LSD radix sort over other algorithms was based on extensive testing showing its superior performance for the specific workload of sorting face centroids in interactive volume rendering [S1, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
