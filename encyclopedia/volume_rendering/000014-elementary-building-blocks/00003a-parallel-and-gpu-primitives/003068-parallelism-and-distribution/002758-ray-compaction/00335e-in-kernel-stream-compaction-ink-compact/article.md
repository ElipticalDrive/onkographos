# In-Kernel Stream Compaction (InK-Compact)
Also called: In-Kernel compaction, InK-Compact

## Overview
InK-Compact is a parallel stream compaction method that computes the output indices for valid elements entirely within a single GPU kernel, eliminating the need to launch separate prefix-sum and scatter kernels [S1, unofficial]. Unlike conventional methods such as Thrust or Chag::PP which require leaving the kernel context, InK-Compact maintains global input-output data ordering by computing thread, warp, and block offsets using atomic operations and bit-decomposition scans, thereby reducing synchronization overhead and kernel launch latency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An input stream $X[0 \dots N-1]$ with a validity predicate $P[0 \dots N-1]$ (where $P[i]=1$ if valid, $0$ otherwise).
- Output: A compacted stream $Y[0 \dots M-1]$ containing only elements where $P[i]=1$, preserving the relative order of $X$.
- Intermediate: An intermediate buffer $\tilde{Y}$ for local block compaction.

**Data Structures:**
- **Section Data Structure ($S_i$):** Each section $i$ has three fields:
  - $S_i(f)$: Boolean flag indicating section completion.
  - $S_i(u)$: Offset value for the start of section $i$ in the output buffer.
  - $S_i(c)$: Atomic counter for the number of blocks in section $i$.
- **Shared Memory:**
  - $ww(c)(i)$: Stores the total write count for warp $i$.
  - $s(u)$: Stores the warp offset for the current warp.
  - $k(u)$: Stores the kernel offset (for InK-Collate variant).
- **Global Memory:**
  - $B(c)$: Buffer to store block counts for the section controller.
  - Global counter: A single integer in global device memory (for InK-Collate).

**Parameters:**
- $w(s)$: Warp size, typically 32 threads (CUDA 4.0) [S1, unofficial].
- $w(n)$: Number of warps per block.
- $S(N)$: Section size, number of blocks per section. Chosen as 32 to fit the final bit-decomposition scan into a single warp [S1, unofficial].

**Algorithm Steps:**
The method consists of three conceptual stages: computing thread offset $t(u)$, warp offset $w(u)$, and block offset $b(u)$.

1. **Thread Offset Computation ($t(u)$):**
   - Compute warp index: $w(i) = \lfloor t(i) / w(s) \rfloor$.
   - Compute warp lane: $w(l) = t(i) \pmod{w(s)}$.
   - Compute thread mask: $t(m) = \text{MAXINT} \gg (w(s) - w(l))$. This mask has bits $0$ to $w(l)-1$ set to 1.
   - Compute ballot: $b = \text{ballot}(P[t]) \ \& \ t(m)$.
   - Compute thread offset: $t(u) = \text{popc}(b)$.
   - Compute warp write count: The last thread in the warp ($w(l) = w(s)-1$) calculates $w(c) = t(u) + P[t]$.
   - Store $w(c)$ into shared memory $ww(c)(i)$ by the last thread only.
   - Perform memory fence and synchronization.

2. **Warp Offset Computation ($w(u)$):**
   - Synchronize using `_syncthreads_count()` with each thread's predicate to gather total writes in the block.
   - Compute warp offset $w_i(u)$ using bit-decomposition scan over $\log_2(w(s))$ bits.
   - For each bit $j$ from $0$ to $\log_2(w(s))-1$:
     - $b(j) = \text{ballot}(w_i(c) \ \& \ 2^j)$.
     - Accumulate: $w_i(u) += \text{popc}(b(j) \ \& \ t(m)) \gg i$.
   - Store $w_i(u)$ in shared memory $s(u)$.

3. **Block Offset and Compaction (Order-Preserving):**
   - **Stage 1 (Local Compaction):** Each block computes local offsets. Valid outputs are stored in intermediate buffer $\tilde{Y}$ at index: $\text{block\_id} \times \text{threads\_per\_block} + t(u) + w(u)$. The block computes its total write count $k(c)$.
   - **Stage 2 (Section Controller):** A single thread atomically increments the section counter $S_i(c)$. All blocks except the section controller (the last block in the section, where $S_i(c)$ reaches $S(N)$) exit. The section controller loads block counts into $B(c)$ and performs a bit-wise prefix sum scan to find block offsets.
   - **Stage 3 (Section Synchronization):** The last block in section $S_i$ waits for the flag $S_{i-1}(f)$ from the previous section to become true. Section $S_0$ proceeds immediately.
   - **Stage 4 (Final Write):** Once the total valid elements $t(f)$ for the section are known, the section controller sets $S_i(u) = S_{i-1}(u) + t(f)$. It sets the memory fence and then sets $S_i(f) = \text{true}$. The last block in $S_i$ moves data from $\tilde{Y}$ to the final output array $Y$ using the computed offsets.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ink_compact_kernel(input_stream, predicate, output_stream, intermediate_buffer):
    # Constants
    WARP_SIZE = 32
    SECTION_SIZE = 32 # Number of blocks per section
    
    # Shared memory variables
    shared_ww_c = [0] * WARP_SIZE # Warp counts
    shared_s_u = 0 # Warp offset
    shared_k_u = 0 # Kernel offset (for collate)
    
    # Global section data (assumed pre-allocated and zeroed)
    # section_flags, section_offsets, section_counters
    
    tid = get_thread_id()
    bid = get_block_id()
    warp_id = tid // WARP_SIZE
    lane_id = tid % WARP_SIZE
    
    # --- Step 1: Thread Offset ---
    # Compute mask for threads before current thread in warp
    thread_mask = MAXINT >> (WARP_SIZE - lane_id)
    
    # Ballot for valid predicates in warp
    ballot_result = ballot(predicate[tid])
    masked_ballot = ballot_result & thread_mask
    
    # Thread offset is population count of masked ballot
    t_u = popc(masked_ballot)
    
    # Warp write count (computed by last thread in warp)
    w_c = 0
    if lane_id == WARP_SIZE - 1:
        w_c = t_u + predicate[tid]
        shared_ww_c[warp_id] = w_c
        memory_fence()
        syncthreads()
        
    # --- Step 2: Warp Offset ---
    # Bit-decomposition scan for warp offset
    w_i_u = 0
    for j in range(log2(WARP_SIZE)):
        # Ballot warp counts for the j-th bit
        b_j = ballot(shared_ww_c[warp_id] & (1 << j))
        # Accumulate popcount of bits before current warp
        w_i_u += popc(b_j & thread_mask) >> warp_id
        
    shared_s_u = w_i_u
    syncthreads()
    
    # --- Step 3: Block Offset and Local Compaction ---
    # Calculate local output index in intermediate buffer
    local_offset = bid * (WARP_SIZE * get_warps_per_block()) + t_u + shared_s_u
    if predicate[tid]:
        intermediate_buffer[local_offset] = input_stream[tid]
        
    # Compute block write count k_c
    # This is typically the sum of all warp counts in the block
    # Using the syncthreads_count result from Step 2 or re-summing
    k_c = syncthreads_count(predicate[tid])
    
    # --- Step 4: Section Management ---
    # Atomic increment of section counter
    section_id = bid // SECTION_SIZE
    section_counter = atomic_add(section_counters[section_id], 1)
    
    # Determine if this block is the section controller (last block in section)
    is_controller = (section_counter == SECTION_SIZE - 1)
    
    if is_controller:
        # Load block counts into shared memory B(c)
        # Note: In practice, blocks must have stored their k_c somewhere accessible
        # or we rely on the fact that k_c was computed. 
        # Assuming k_c is available or stored in a global array B(c) indexed by block
        load_block_counts_into_shared()
        
        # Perform prefix sum on block counts to find section offsets
        # This fits in one warp if SECTION_SIZE <= 32
        section_offset = 0
        if section_id > 0:
            section_offset = section_offsets[section_id - 1]
            
        # Calculate total valid elements in this section
        total_valid = 0
        for b in range(SECTION_SIZE):
            total_valid += block_counts[b]
            
        section_offsets[section_id] = section_offset + total_valid
        memory_fence()
        
        # Wait for previous section to finish
        if section_id > 0:
            while not section_flags[section_id - 1]:
                pass # Spin wait
            
        # Set current section flag
        section_flags[section_id] = True
        
        # Move data from intermediate to final output
        # The last block in the section performs the final copy
        # Or all blocks copy based on their local offset + section_offset
        final_offset = local_offset + section_offsets[section_id]
        if predicate[tid]:
            output_stream[final_offset] = intermediate_buffer[local_offset]
    else:
        # Non-controller blocks:
        # Wait for section offset to be ready? 
        # In the described algorithm, non-controllers exit after Stage 2/3 
        # but must wait for the section offset to write to final output.
        # However, the text says "All blocks within the section, except for the section controller 
        # are allowed to exit...". This implies the controller handles the global offset logic.
        # But to write to Y, they need the offset.
        # The text says: "The last block in section Si moves the already block-compacted data... to Y".
        # This implies only the last block copies? Or all blocks copy?
        # "moves the already block-compacted data from the intermediate output array Si to the final output array Y"
        # Usually, all threads write their own data.
        # Let's assume all threads write, but need the section offset.
        
        # Wait for section offset to be computed by controller
        while section_offsets[section_id] == 0 and section_id > 0:
             pass # Spin wait for offset
            
        final_offset = local_offset + section_offsets[section_id]
        if predicate[tid]:
            output_stream[final_offset] = intermediate_buffer[local_offset]
```

*Note: The pseudocode above synthesizes the steps. The source specifies that non-controller blocks "exit" after incrementing the counter, but they still need to write their data. The source states "The last block in section Si moves the already block-compacted data... to Y". This suggests a potential optimization where only the last block performs the final scatter, or that the "exit" refers to exiting the *compaction logic* (atomic ops), not the kernel. Given the register burden note, it is likely all threads write, but the synchronization logic is complex. The provided pseudocode assumes all threads write to `output_stream` using the `section_offsets` computed by the controller.*

## Subtopics

### In-Kernel Collation (InK-Collate)
A compromise compaction method that preserves local ordering within a block but not global ordering of kernel blocks, avoiding inter-block dependencies [S1, unofficial]. It computes a kernel offset $k(u)$ using `atomicAdd` on a global counter by thread 0 in the block, then adds the local warp/thread offsets [S1, unofficial]. It is non-order preserving globally but shows little performance difference from InK-Compact in ray-tracing applications [S1, unofficial].

### Out-of-Kernel Compaction (OfK-Compact)
A variant where InK-Compact is wrapped in a global kernel and called when data needs to be compacted, mimicking the usage pattern of Thrust and Chag::PP [S1, unofficial]. Kernels output ray indices to an output array (non-valid elements set to zero/null), and then a host-side method (like Thrust's `copy-if`) is called to compact the data [S1, unofficial]. When used this way, it runs on par with Chag::PP [S1, unofficial].

### Chag::PP Compaction
A comparison baseline where kernels output ray indices to an output array (non-valid elements set to zero/null), followed by a call to `Chag::PP.compact` to compact the items [S1, unofficial]. It is slower than InK-Compact and requires kernel synchronization before calling the host-side compaction [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method performs a constant number of passes per warp and block, with $O(\log w(s))$ complexity for the warp offset scan. The block offset computation is $O(S(N))$ for the controller.
- **Memory Complexity:** Requires an intermediate buffer $\tilde{Y}$ of size $N$ (or $M$) and shared memory for warp counts. The intermediate buffer allows blocks to exit early, freeing resources [S1, unofficial].
- **Performance:** On average, InK-Compact is 99% faster than Thrust and 28% faster than Chag::PP in ray-tracing pipeline tests on an NVIDIA 480 GTX [S1, unofficial]. When used out-of-kernel, it runs on par with Chag::PP [S1, unofficial].
- **Strengths:** Eliminates kernel launch overhead and synchronization between separate kernels. Reduces bottlenecks by keeping compaction within the operating kernel [S1, unofficial].
- **Limitations:** Causes a register burden because it runs as a device function within the operation kernel. However, on architectures like Fermi, registers used in the main work are reallocated for compaction once the main work is complete, mitigating this cost [S1, unofficial]. Requires an intermediate buffer, which increases memory usage if the data type is large [S1, unofficial].

## Background
InK-Compact was introduced by D. M. Hughes et al. in 2013 [S1, unofficial]. It was developed to address the inefficiencies of multi-kernel pipelines in GPGPU computing, particularly for ray-tracing-based visualization of volumetric data. The method was motivated by the need to avoid the overhead of leaving the kernel to run prefix sum and scatter kernels, as done by libraries like Thrust [S1, unofficial]. It is used in a multi-kernel isosurface ray-tracing pipeline for stages: Generation-Traversal, Traversal-Intersection, Intersection-Traversal, and Intersection-Shade [S1, unofficial].

## Sources
[S1] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
