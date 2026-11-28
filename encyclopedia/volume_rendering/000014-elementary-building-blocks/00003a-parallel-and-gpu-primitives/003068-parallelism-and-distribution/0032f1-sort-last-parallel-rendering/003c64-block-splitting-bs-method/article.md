# Block-Splitting (BS) method
Also called: BS method, Binary Splitting (BS) Method
## Overview
The Block-Splitting (BS) method is a parallel compositing technique for sort-last parallel volume rendering that reduces the number of communication steps to $\log_2 P$ by doubling the block size at each stage [S1, unofficial]. It serves as a baseline for comparison against hierarchical variants like the DSH method [S2]. Unlike the Parallel Pipeline (PP) method, BS is optimal when the processor count $P$ is a power of two, offering fewer communication steps for square processor grids [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** $P$ processors, each holding a partial image of size $A/P$ pixels. The total image size is $A$ [S1, unofficial].
*   **Output:** One complete composite image held by a designated root processor [G].

**Data Structures:**
*   **Partial Image:** A 2D array of pixels [G].
*   **Block:** A rectangular sub-region of the partial image. Initially, each processor holds one block of size $A/P$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Each processor $P_i$ ($0 \le i < P$) holds a partial image block $B_i$ of size $A/P$ [S1, unofficial].
2.  **Communication Loop:** Perform $k = 1$ to $\log_2 P$ steps:
    a.  **Pairing:** In step $k$, processors are paired based on their indices. Specifically, processor $i$ communicates with processor $i \oplus 2^{k-1}$ (where $\oplus$ is bitwise XOR) [G].
    b.  **Block Size:** In step $k$, the block size being transmitted is $A_k = A / 2^k$ [S1, unofficial].
    c.  **Transmission:** The sending processor transmits its current block to the receiving processor [G].
    d.  **Compositing:** The receiving processor composites the received block with its own local block using the "over" operator [S1, unofficial].
3.  **Termination:** After $\log_2 P$ steps, the root processor (or the designated final accumulator) holds the fully composite image [G].

**Formulas:**
*   **Block Size at Step $k$:** $A_k = \frac{A}{2^k}$ [S1, unofficial].
*   **Communication Time per Step:** $T_s$ (latency) + $A_k \times T_p$ (pixel transfer time) [S1, unofficial].
*   **Compositing Time per Step:** $A_k \times T_o$ (pixel operation time) [S1, unofficial].
*   **Total Time:**
    $$ T_{total}(BS) = \sum_{k=1}^{\log P} \left( T_s + \frac{A}{2^k} T_p \right) + \sum_{k=1}^{\log P} \frac{A}{2^k} T_o $$
    Simplifying the geometric series $\sum_{k=1}^{\log P} \frac{1}{2^k} = 1 - \frac{1}{2^{\log P}} = 1 - \frac{1}{P}$:
    $$ T_{total}(BS) = (\log P) T_s + \left(1 - \frac{1}{P}\right) (T_p + T_o) A $$
    [S1, unofficial]

**Parameters:**
*   $P$: Number of processors. Must be a power of two for the standard binary splitting topology [S1, unofficial].
*   $A$: Total number of pixels in the final image.
*   $T_s$: Communication latency per message.
*   $T_p$: Pixel transfer time per unit area.
*   $T_o$: Compositing operation time per unit area.

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Processors:** The standard BS method assumes $P$ is a power of two. If $P$ is not a power of two, the topology must be adapted (e.g., using the Rotate-Tiling method which generalizes BS) [S1, unofficial].
*   **Precision:** Standard floating-point compositing is assumed. No specific precision handling is detailed in the source [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_splitting_composite(partial_image, processor_id, total_processors, A, Ts, Tp, To):
    """
    Performs Block-Splitting compositing.
    partial_image: The initial partial image block held by this processor.
    processor_id: Unique ID of this processor (0 to P-1).
    total_processors: P, must be a power of 2.
    A: Total pixels in the final image.
    Ts, Tp, To: Timing parameters.
    Returns: The final composite image.
    """
    import math
    
    current_block = partial_image
    num_steps = int(math.log2(total_processors))
    
    for k in range(1, num_steps + 1):
        mask = 1 << (k - 1)
        partner_id = processor_id ^ mask
        block_size = A // (1 << k)
        
        if (processor_id & mask) == 0:
            # [G] Standard network send to partner_id
            send(current_block, partner_id)
        else:
            # [G] Standard network receive from partner_id
            received_block = receive(partner_id)
            current_block = composite_over(received_block, current_block)
            
    return current_block

def composite_over(src, dst):
    # [G] Standard Porter-Duff 'over' operator applied pixel-wise
    # Assumes RGBA tuples. Returns new list of tuples.
    return [(s[0] + d[0] * (1 - s[3]), s[1] + d[1] * (1 - s[3]), 
             s[2] + d[2] * (1 - s[3]), s[3] + d[3] * (1 - s[3])) 
            for s, d in zip(src, dst)]
```
## Subtopics

### [DSH (Divided-Screenwise Hierarchical) method](9240-dsh-divided-screenwise-hierarchical-method.md)
Eliminates redundant communication by dividing the screen into non-overlapping tiles and dynamically assigning processors. Choose when $P$ is large and data coverage is sparse to avoid transmitting blank regions.

### [Rotate-Tiling (RT) Method](16314-rotate-tiling-rt-method.md)
Generalizes BS to arbitrary $P$ and $N$ (number of blocks) using ring rotations. Choose when $P$ is not a power of two or when performance tuning via $N$ is required.

## Operating characteristics
**Time and Memory Complexity:**
*   **Communication Steps:** $\log_2 P$ [S1, unofficial].
*   **Total Time:** $T_{total}(BS) = (\log P) T_s + (1 - 1/P) (T_p + T_o) A$ [S1, unofficial].
*   **Memory:** Each processor holds one block of size $A/P$ at any time. No significant additional memory for buffering is required beyond the block being transmitted [S1, unofficial].

**Measured Performance:**
*   BS is faster than the Parallel Pipeline (PP) method when $P$ is a power of two, because $T_{total}(BS) \le T_{total}(PP)$ in this case [S1, unofficial].
*   BS transmits more pixels than the DSH method as the number of regions $n$ increases, because DSH avoids transmitting redundant blank regions [S2].

**Strengths and Limitations:**
*   **Strengths:** Simple topology (butterfly), minimal communication steps ($\log P$), optimal for power-of-two processor counts [S1, unofficial].
*   **Limitations:** Inefficient for non-power-of-two processor counts without modification. Transmits all data regardless of sparsity (unlike DSH), leading to higher bandwidth usage on sparse volumes [S2].

**When to Use:**
*   Use when $P$ is a power of two and the dataset is dense enough that the simplicity of the butterfly network outweighs the benefit of hierarchical tiling [G].
*   Use as a baseline for comparison [S2].

**When Not to Use:**
*   Use when $P$ is not a power of two (use RT instead) [S1, unofficial].
## Background
The Block-Splitting method is a foundational sort-last parallel compositing algorithm [S2]. It is cited in the context of comparing various parallel volume rendering compositing strategies, specifically against the Parallel Pipeline (PP) and Rotate-Tiling (RT) methods [S1, unofficial]. It is also used as a baseline for the Divided-Screenwise Hierarchical (DSH) method [S2]. The specific origin paper is not explicitly named in the provided notes, but the analysis is attributed to [S1, unofficial] and [S2].
## Sources
[S1] A Rotate-Tiling Image Compositing Method for Sort-Last Parallel Volume ..., https://ychung1103.github.io/Journal/2004-JISE.pdf, fetched 2026-10-29 (unofficial)
[S2] A Divided-Screenwise Hierarchical Compositing for Sort-Last Parallel ..., http://www-ppl.ist.osaka-u.ac.jp/research/papers/200304_ino_ipdps.pdf, fetched 2026-10-29
