# 4-way Parallel Radix Sort

Also called: 2-bit-radix sort, 4-way parallel radix sorting, 4-way radix sort, Fast 4-way parallel radix sorting on GPUs, Four-way radix sorting

## Overview
A hardware-optimized parallel implementation of the radix sort algorithm for GPUs that processes two bits per iteration (radix 4) to sort elements, lifting limitations on element count and precision compared to previous GPU sorters [S1, unofficial]. It reduces the number of passes by half compared to binary-radix sort, eliminating restrictions on key space and supporting arbitrary data types [S1, unofficial]. Unlike [8-bit Radix Sort](2599-8-bit-radix-sort.md) which processes one bit at a time, this method clusters numbers with the same radix combination bits inside a block to enable coalesced memory writing, addressing the non-coalesced scattering bottleneck of element-wise mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of $N$ keys (32-bit integers, floats, or mapped floats) and an optional array of values/indices to be permuted in parallel.
- Output: The keys and associated data sorted in ascending order.

**Data Structures:**
- `blockSize`: Integer, typically 256 (128 threads $\times$ 2 elements per thread).
- `blockSumArray`: Array of size $\lceil N / blockSize \rceil \times 4$. Stores the total count of elements for each of the 4 digits (0-3) per block.
- `localPrefixSumArray`: Array of size $N$. Stores the local prefix sum for the specific digit of each element.
- `sharedMemory`: Size sufficient for `blockSize` elements.

**Parameters:**
- `radixWidth`: 2 bits.
- `keyBits`: 32 bits (loop runs for bits 0 to 30).
- `blockSize`: 256.
- `threadsPerBlock`: 128.

**Algorithm Steps:**

1.  **Preprocessing (Floating Point):** If keys are floating-point numbers, map them to integers using the method by Terdiman [Ter00] to preserve order for negative and positive numbers.
2.  **Initialization:**
    - Set `blockSize = 256`.
    - Allocate `blockSumArray` on GPU memory and initialize to zero.
    - Allocate `localPrefixSumArray` on GPU memory.
3.  **Main Loop:** Iterate `bit` from 0 to 30 (inclusive), incrementing by 2.
    1.  **Order Check:** Perform a parallel reduction to check if the current input array is already sorted. If sorted, terminate the loop immediately [S1, unofficial].
    2.  **Block Division:** Divide the input data into equal blocks of size `blockSize`.
    3.  **Parallel Block Processing (for all blocks in parallel):**
        1.  **Load Data:** Load `blockSize` elements into shared memory (`s_data`).
        2.  **Bit Extraction:** Each thread extracts the 2-bit radix pair from its assigned elements at the current `bit` position.
        3.  **Counting:** Initialize four counting arrays `cnt[0..3]` in shared memory. For each element, set the corresponding counter bit to 1 if the extracted radix matches the digit index.
        4.  **Local Prefix Sum:** Perform a parallel optimized prefix sum (scan) on the four counting combinations simultaneously in shared memory. This computes the local frequency and position of each element within its block relative to its digit [S1, unofficial].
        5.  **Shuffle:** Reorder elements in shared memory so that elements with the same radix digit are clustered together in sorted order. This ensures that subsequent global writes are coalesced [S1, unofficial].
        6.  **Store Local Prefix:** Write the local prefix sum value for each element to `localPrefixSumArray` at the element's original global index.
        7.  **Store Block Sum:** Write the total count of elements for each of the 4 digits to the corresponding entry in `blockSumArray` for this block.
    4.  **Global Prefix Sum:** Perform a parallel prefix sum (scan) on the `blockSumArray` to compute global starting positions for each block and digit combination.
    5.  **Mapping:**
        - For each element, calculate its final sorted position using the formula:
          $$ \text{Sorted Position} = P_d[n] + m $$
          Where:
          - $d$ is the 2-bit radix digit of the element.
          - $n$ is the block index containing the element.
          - $P_d[n]$ is the global prefix block sum for digit $d$ at block $n$ (from the global scan of `blockSumArray`).
          - $m$ is the local prefix sum value for the element (from `localPrefixSumArray`).
        - Permute the elements from the shared memory shuffle buffer to their calculated global positions in the output array. This mapping is cluster-wise, ensuring coalesced global memory writes [S1, unofficial].
    6.  **Swap Buffers:** If not the last pass, swap input and output arrays.
    7.  Increment `bit` by 2.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def four_way_radix_sort(keys, values, n):
    blockSize = 256
    threadsPerBlock = 128
    numBlocks = (n + blockSize - 1) // blockSize
    
    # Allocate memory
    blockSumArray = allocate_gpu_array(numBlocks * 4, dtype=int)
    localPrefixSumArray = allocate_gpu_array(n, dtype=int)
    outputKeys = allocate_gpu_array(n, dtype=int)
    outputValues = allocate_gpu_array(n, dtype=int)
    
    # Initialize blockSumArray to 0
    clear_gpu_array(blockSumArray)
    
    # Current and next buffers
    currKeys = keys
    currValues = values
    nextKeys = outputKeys
    nextValues = outputValues
    
    for bit in range(0, 31, 2):
        # 1. Order Check
        if is_sorted(currKeys, n):
            break
            
        # 2. Parallel Block Processing
        for block in range(numBlocks):
            startIdx = block * blockSize
            endIdx = min(startIdx + blockSize, n)
            localSize = endIdx - startIdx
            
            # Launch kernel for block processing
            # Kernel arguments: currKeys, currValues, localPrefixSumArray, 
            #                   blockSumArray, block, bit, localSize
            
            # Pseudo-kernel logic for a single block:
            # shared s_data[256]
            # shared cnt[4][128] # 4 counters, 128 threads
            # shared localPos[128]
            
            # Load data to shared memory
            for i in range(localSize):
                tid = get_thread_id()
                if tid < localSize:
                    s_data[tid] = currKeys[startIdx + tid]
            
            sync_threads()
            
            # Extract bits and count
            for tid in range(threadsPerBlock):
                if tid < localSize:
                    val = s_data[tid]
                    radix = (val >> bit) & 3 # Extract 2 bits
                    
                    # Initialize counters
                    for b in range(4):
                        cnt[b][tid] = 1 if b == radix else 0
            
            # Parallel Prefix Sum on counters (shared memory scan)
            # This computes the rank of each element within its digit group
            parallel_scan_4way(cnt, localSize)
            
            # Store local prefix sum for this element
            if tid < localSize:
                localPos[tid] = cnt[radix][tid]
                localPrefixSumArray[startIdx + tid] = localPos[tid]
                
            # Shuffle data in shared memory based on radix
            # Elements with same radix are grouped together
            shuffle_shared_memory(s_data, cnt, radix, localSize)
            
            # Store block sums
            # Sum up the last element of each counter array
            for b in range(4):
                blockSumArray[block * 4 + b] = cnt[b][localSize - 1]
                
            sync_threads()
            
            # Write shuffled data to global memory temporarily or keep in shared for final mapping
            # The algorithm description implies shuffling happens before global write
            # to enable coalesced writing.
            for tid in range(localSize):
                nextKeys[startIdx + tid] = s_data[tid] # Placeholder for shuffled data
                # Note: Actual implementation might write to a temporary buffer or 
                # directly to final positions if global offsets are known.
                # However, global offsets depend on the global scan of blockSumArray.
                
        # 3. Global Prefix Sum on blockSumArray
        # Compute prefix sum over blockSumArray to get global offsets
        global_scan(blockSumArray, numBlocks * 4)
        
        # 4. Final Mapping
        # Calculate final positions and write to nextKeys/nextValues
        for i in range(n):
            val = currKeys[i]
            radix = (val >> bit) & 3
            block = i // blockSize
            offset_in_block = i % blockSize
            
            # P_d[n] is the global prefix sum for digit d at block n
            # Note: blockSumArray stores counts. The global scan result 
            # gives the starting index for each (block, digit) pair.
            # Let globalOffsets be the result of the scan.
            # globalOffsets[block * 4 + radix] is the start position for this digit in this block.
            
            # m is the local prefix sum (rank within the digit group in the block)
            # We need to retrieve this. In the block processing step, we stored it in localPrefixSumArray.
            m = localPrefixSumArray[i]
            
            # The global scan of blockSumArray provides the base index.
            # However, the blockSumArray contains counts. The prefix sum of counts 
            # gives the starting index.
            # We need the prefix sum result. Let's assume a function get_global_offset(block, digit)
            # which returns the value from the scanned blockSumArray.
            
            base = get_global_offset(block, radix)
            finalPos = base + m
            
            nextKeys[finalPos] = val
            nextValues[finalPos] = currValues[i] # Assuming values are permuted similarly
            
        # Swap buffers
        swap(currKeys, nextKeys)
        swap(currValues, nextValues)

def is_sorted(arr, n):
    # Optimized reduction to check if arr[i] <= arr[i+1] for all i
    # Returns True if sorted, False otherwise
    pass

def parallel_scan_4way(cnt, localSize):
    # Perform parallel prefix sum on the 4 counting arrays in shared memory
    # Standard parallel scan algorithm (e.g., Blelloch or Hillis-Steele)
    # applied to each of the 4 arrays independently
    pass

def shuffle_shared_memory(s_data, cnt, radix, localSize):
    # Reorder s_data such that elements with the same radix are contiguous
    # and sorted by their local rank (cnt[radix][tid])
    # This requires a scatter operation in shared memory
    pass

def get_global_offset(block, digit):
    # Retrieve the value from the globally scanned blockSumArray
    # blockSumArray[block * 4 + digit] after scan
    pass
```

**Edge Cases and Pitfalls:**
- **Non-coalesced Scattering:** Standard radix sort scatters elements to global memory based on individual indices, causing uncoalesced writes. This method uses in-block shuffling to cluster elements with the same radix, enabling coalesced global writes [S1, unofficial].
- **Floating Point Mapping:** Floating point numbers must be mapped to integers to preserve order. The Terdiman method is used [S1, unofficial].
- **Block Size:** The block size (256) is chosen to be a multiple of the warp size (32) to ensure coalesced memory access and to hide global memory latency while fitting in shared memory/cache [S1, unofficial].
- **Order Checking:** The order check is performed at the start of each pass. If the array is already sorted, the loop terminates early, saving passes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot \frac{B}{R})$, where $N$ is the number of elements, $B$ is the number of bits in the key (32), and $R$ is the radix width (2). This results in 16 passes for 32-bit keys.
- **Memory Complexity:** $O(N)$ for the input/output arrays, plus $O(N/R)$ for the block sum array and $O(N)$ for the local prefix sum array.
- **Performance:** Outperforms all known GPU-based sorting systems by about a factor of two at the time of publication [S1, unofficial]. It is almost twice as fast as the fastest known approaches (Sintorn/Assarsson, Sengupta, Le Grand) [S1, unofficial].
- **Strengths:**
    - Eliminates restrictions on key space and supports arbitrary data types (floats, negatives, pointers) [S1, unofficial].
    - Reduces the number of passes by half compared to binary-radix sort [S1, unofficial].
    - Early termination if the array is already sorted [S1, unofficial].
    - Coalesced global memory writes via in-block shuffling [S1, unofficial].
- **Limitations:**
    - Requires shared memory for the in-block shuffle, limiting the maximum block size [S1, unofficial].
    - The order check adds a small overhead (approx. 0.1% of sorting time) but is negligible [S1, unofficial].
- **When to Use:** Ideal for GPU-based sorting of large datasets where performance is critical, especially in physics simulations, collision detection, and scenes with thousands of objects or millions of particles [S1, unofficial].

## Background
The algorithm was presented by Linh Ha, Jens Krüger, and Cláudio T. Silva from the University of Utah in a paper submitted to COMPUTER GRAPHICS Forum in February 2009 [S1, unofficial]. It addresses the challenge of efficient parallel sorting on GPUs, overcoming limitations of previous approaches like CUDPP (binary-radix sort) and Sintorn/Assarsson (atomic counting) [S1, unofficial]. The work was motivated by the need for fast sorting in physics-based simulations for computer games and virtual environments, such as collision detection for falling chairs, particle systems, and n-body galaxy problems [S1, unofficial].

## Sources
[S1] Fast 4-way parallel radix sorting on GPUs, https://www.sci.utah.edu/~csilva/papers/cgf.pdf, fetched 2026-10-29 (unofficial)
