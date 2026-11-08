# DSH (Divided-Screenwise Hierarchical) method

Also called: Divided-Screenwise Hierarchical (DSH) Compositing

## Overview
The DSH method is a sort-last parallel volume rendering compositing technique designed for high performance on distributed memory multiprocessors with a large number of processors [S1]. It solves the problem of redundant communication and inefficient load balancing in hierarchical compositing by dividing the screen into non-overlapping tile-shaped regions, dynamically assigning processors to these regions based on their data coverage, and compositing each region concurrently in a hierarchical order [S1]. Unlike the Block-Splitting (BS) method, DSH eliminates redundant communication between processors that lack common non-blank regions and allows up to $D$ processors to merge the final image simultaneously, avoiding the bottleneck where only one processor merges the final image as in the BT method [S1].

## How it works
The DSH method operates in three phases: preprocessing, dynamic assignment, and concurrent hierarchical compositing.

### Inputs and Outputs
*   **Inputs:**
    *   $n$: Number of processors (nodes).
    *   $D$: Number of screen divisions (regions). Constraint: $D \ge n^{2/3}$ [S1].
    *   $B = \{\tilde{B}^d_q \mid 1 \le q \le n, 1 \le d \le D\}$: Set of bounding rectangles of divided-subimages $\tilde{I}^d_q$.
    *   $Z = \{\tilde{B}_q \mid 1 \le q \le n\}$: Set of bounding rectangles of projected subvolumes $\tilde{V}_q$.
    *   $\tilde{I}_q$: The rendered subimage for processor $q$.
*   **Outputs:**
    *   $O = \{O_d \mid 1 \le d \le D\}$: Set of compositing order lists for each region.
    *   The final composited image.

### Data Structures
*   **Bounding Rectangle ($\tilde{B}$):** Defined by coordinates $(x_{min}, y_{min}, x_{max}, y_{max})$ and area $A(\tilde{B}) = (x_{max}-x_{min})(y_{max}-y_{min})$.
*   **Processor Set ($W_d$):** A list of processor indices $q$ that have non-blank pixels within region $d$.
*   **Compositing Order ($O_d$):** A list of pairs $(q_s, q_l)$ representing sender-receiver relationships for each compositing stage in region $d$.

### Algorithm Steps

1.  **Screen Division:**
    For each processor $q$ ($1 \le q \le n$), divide the subimage $\tilde{I}_q$ into $D$ non-overlapping tile-shaped regions to obtain divided-subimages $\tilde{I}^1_q, \tilde{I}^2_q, \dots, \tilde{I}^D_q$ [S1].

2.  **Preprocessing (Broadcast):**
    Broadcast the coordinates of all bounding rectangles $\tilde{B}_q$ and $\tilde{B}^d_q$ from all processors to all other processors at the beginning of the compositing phase. This allows every processor to determine the processor sets $W_d$ [S1].

3.  **Determine Processor Sets and Compositing Order:**
    For each region $d$ ($1 \le d \le D$):
    a.  Initialize $W_d = \emptyset$ and $O_d = \emptyset$.
    b.  Identify processors with non-blank pixels in region $d$. A processor $q$ is included in $W_d$ if the bounding rectangle $\tilde{B}^d_q$ is within region $d$ (or intersects it, effectively meaning it has data there). To optimize, check if $\tilde{B}^d_q \cap \tilde{B}_q \neq \emptyset$ [S1].
    c.  While $W_d$ is not empty, determine the compositing stage for region $d$:
        i.  Let $W_{d,stg} = W_d$.
        ii. Select the processor $q_l \in W_{d,stg}$ with the largest bounding rectangle area $A(\tilde{B}^d_{q_l})$ as the receiver [S1].
        iii. Select the processor $q_s \in W_{d,stg}$ with the smallest bounding rectangle area $A(\tilde{B}^d_{q_s})$ as the sender [S1].
        iv. Pair $(q_s, q_l)$ and add to $O_d$.
        v.  Remove $q_s$ and $q_l$ from $W_{d,stg}$. If $|W_{d,stg}|$ is odd, one processor remains unpaired (isolated) and stays in $W_d$ for the next stage.
        vi. Update $W_d = W_{d,stg}$ (processors not paired are removed from the current stage's active set but remain for subsequent stages if they still have data, effectively reducing the set size) [S1].
        *Note: The sources describe an iterative pairing process where $|W_d|$ reduces. In the worst case, isolated processors exist between every pair, reducing $|W_d|$ by a factor of $2/3$ per stage [S1].*

4.  **Concurrent Hierarchical Compositing:**
    For each region $d$ ($1 \le d \le D$), execute the compositing stages defined in $O_d$ concurrently across all regions.
    a.  For each stage $k$ in $O_d$:
        i.  For each pair $(q_s, q_l)$ in stage $k$:
            *   Processor $q_s$ transmits its non-blank pixels within $\tilde{B}^d_{q_s}$ to $q_l$.
            *   Processor $q_l$ composites the received pixels with its current buffer for region $d$.
    b.  Use the **Receiver First (RF)** scheme: The receiver $q_l$ is selected based on having the larger bounding rectangle, and the sender $q_s$ is the one with the smaller bounding rectangle [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def determine_compositing_order(n, D, B_divided, B_subvolume):
    """
    Determines the processor sets Wd and compositing orders Od for all regions.
    B_divided: dict mapping (q, d) to bounding rectangle B_dq
    B_subvolume: dict mapping q to bounding rectangle B_q
    """
    O = {} # O[d] = list of stages, each stage is list of (sender, receiver)
    W = {} # W[d] = set of active processors for region d
    
    # Step 1: Initialize Wd for all regions
    for d in range(1, D + 1):
        W[d] = set()
        for q in range(1, n + 1):
            # Check if processor q has data in region d
            # Using the bounding rectangle of the divided subimage
            B_dq = B_divided[(q, d)]
            if B_dq is not None and not is_empty(B_dq):
                W[d].add(q)
    
    # Step 2: Determine hierarchical order for each region
    for d in range(1, D + 1):
        stages = []
        current_processors = set(W[d])
        
        while len(current_processors) > 1:
            stage_pairs = []
            next_processors = set()
            
            # Sort processors by bounding rectangle area descending to find receiver
            # and ascending to find sender
            sorted_procs = sorted(current_processors, 
                                  key=lambda q: area(B_divided[(q, d)]))
            
            # Receiver First Scheme: Pick largest as receiver, smallest as sender
            # We pair them up. If odd number, the last one remains.
            # The source says: select ql (largest) then qs (smallest) from remaining?
            # Actually, standard pairing in BT-like methods often pairs adjacent in sorted list.
            # Source: "select processor ql such that ql has the largest... select qs... 
            # such that qs has the smallest...". This implies picking extremes.
            
            # Let's implement the extreme pairing as described:
            # Re-sort to ensure we pick extremes correctly if we remove them
            procs_list = sorted(current_processors, 
                                key=lambda q: area(B_divided[(q, d)]))
            
            while len(procs_list) >= 2:
                ql = procs_list[-1] # Largest area (Receiver)
                qs = procs_list[0]  # Smallest area (Sender)
                
                stage_pairs.append((qs, ql))
                
                # Remove from list
                procs_list.pop()
                procs_list.pop(0)
            
            # If one processor remains, it is isolated for this stage
            if procs_list:
                next_processors.add(procs_list[0])
            
            stages.append(stage_pairs)
            current_processors = next_processors
            
        O[d] = stages
        
    return O

def composite_region_dsh(region_d, W_d, O_d, B_divided, images):
    """
    Performs compositing for a specific region d.
    images: dict mapping q to the full subimage I_q
    """
    # Initialize local buffer for region d for each processor in W_d
    local_buffers = {}
    for q in W_d:
        B_dq = B_divided[(q, region_d)]
        # Extract the region d part of the image
        local_buffers[q] = extract_region(images[q], region_d, B_dq)
        
    # Execute stages
    for stage in O_d[region_d]:
        for qs, ql in stage:
            # Sender qs transmits to Receiver ql
            # Transmission is limited to the bounding rectangle of the sender
            B_qs = B_divided[(qs, region_d)]
            pixels_to_send = extract_region(local_buffers[qs], region_d, B_qs)
            
            # Receiver composites
            # Standard front-to-back compositing
            local_buffers[ql] = composite_pixels(local_buffers[ql], pixels_to_send)
            
    # The final buffer of the last remaining processor (or any processor if fully merged)
    # In DSH, all processors in W_d eventually contribute. 
    # The last processor in the chain holds the result for this region.
    # However, since regions are merged concurrently, we need to gather results.
    # The source implies that at most D processors merge the final image simultaneously.
    # This function returns the partial result for region d.
    return local_buffers
```

### Parameters and Constants
*   $D$: Number of screen divisions. Optimal $D_m$ is determined by increasing $D$ from $n$ until $T_{DSH}$ starts to increase; $D_m$ is the value giving the last decrease. Constraint $D \ge n^{2/3}$ [S1].
*   $c$: Proportionality constant for the number of non-blank pixels transmitted by a processor at a stage [S1].
*   $e$: Proportionality constant for the number of processors engaged in a region [S1].
*   $p$: Number of pixels in the final image (or related metric, context implies $p n^{-2/3}$ is non-blank pixels per processor) [S1].
*   $g$: Network latency or per-pixel transmission time constant [S1].
*   $t_c$: Compositing time per pixel [S1].
*   $o$: Overhead per compositing stage [S1].

### Edge Cases and Pitfalls
*   **Bounding Rectangle Growth:** If $D$ is too small, bounding rectangles can grow to the size of the entire region, causing repeated transmission of large areas. Increasing $D$ mitigates this [S1].
*   **Excessive Division:** If $D$ is extremely large relative to $n$, the number of compositing stages increases rapidly, decreasing performance [S1].
*   **Isolated Processors:** In the worst case, isolated processors exist between every pair, reducing the effective processor set size by a factor of $2/3$ per stage [S1].

## Subtopics

### Region-First (RF) scheme
A compositing pairing scheme used within DSH where the receiver $q_l$ is selected first (based on having the largest bounding rectangle) and the sender $q_s$ is selected from the remaining processors (based on having the smallest bounding rectangle). This scheme transmits 4% fewer pixels on average than the Sender First (SF) scheme because it allows fewer bounding rectangles to grow during compositing [S1].

### Screen-First (SF) Compositing Scheme
A compositing pairing scheme compared against RF in DSH. In SF, the sender is selected first. It transmits slightly more pixels on average than RF and is not used in the optimal DSH implementation [S1].

## Operating characteristics
*   **Time Complexity (Stages):** The number of compositing stages $s_{DSH}$ is bounded by $s_{DSH} \le D n^{-2/3} \lceil \log_{3/2}(c n^{1/3}) \rceil$ [S1]. If concurrent compositing is performed in serial, $s_{DSH} = D n^{-2/3} \log_{3/2}|W_d|$. If performed fully independently, $s_{DSH} = \max(D n^{-2/3}, \log_{3/2}|W_d|)$ [S1].
*   **Communication Complexity (Pixels):** The number of transmitted pixels per processor $c_{DSH}$ is $c_{DSH} = s_{DSH} \cdot (e p n^{-2/3} / D)$ [S1].
*   **Total Time:** $t_{DSH} \le p (g + t_c) \cdot \{ e n^{-4/3} \lceil \log_{3/2}(c n^{1/3}) \rceil \} + o \cdot \{ D n^{-2/3} \lceil \log_{3/2}(c n^{1/3}) \rceil \}$ [S1].
*   **Performance:** On a 64-node PC cluster, DSH renders a 512x512x224 voxel volume at approximately 8 fps [S1]. The maximum speedup of DSH over BS ranges from 1.4 to 2.2 when $n=64$ and $D=64$ [S1]. The ratio of transmitted pixels $C_{DSH}/C_{BS}$ decreases from 0.87 at $n=2$ to 0.22 at $n=64$ [S1].
*   **Strengths:** Eliminates redundant communication; simplifies communication patterns via tile-shaped regions; allows parallel final merge; scales well with increasing $n$.
*   **Limitations:** Requires preprocessing broadcast of bounding rectangles; performance depends on the efficiency of concurrent compositing; requires careful tuning of $D$ [S1].

## Background
The DSH method was developed by Ino et al. and presented in 2003 [S1]. It was designed for sort-last parallel volume rendering on distributed memory multiprocessors to address the limitations of existing methods like BS and BT, specifically focusing on maintaining high performance as the number of processors $n$ increases [S1]. It builds on the DS method's screen division and the BT method's hierarchical compositing, but introduces dynamic assignment and tile-shaped regions to optimize communication [S1].

## Sources
[S1] A Divided-Screenwise Hierarchical Compositing for Sort-Last Parallel Volume Rendering, http://www-ppl.ist.osaka-u.ac.jp/research/papers/200304_ino_ipdps.pdf, fetched 2026-10-29
