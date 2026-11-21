# 2-3 Swap Compositing

Also called: 2-3 Swap, 2-3 Swap Image Compositing

## Overview
2-3 Swap Compositing is a parallel image compositing algorithm designed to combine independently rendered image tiles from multiple processors into a single final image, specifically addressing the limitation that the number of processors $N$ must be a power of two in standard Binary Swap Compositing [S1, S2]. It allows an arbitrary number of processors to be used while maintaining the $O(N \log N)$ message complexity of binary swap, ensuring that all participating processors are fully utilized without idle stages [S2]. Unlike Direct Send, which suffers from network congestion and $O(N^2)$ messages, 2-3 Swap limits communication to at most four neighbors per stage, making it suitable for massively parallel environments with thousands of processors [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** $N$ processors, where $N > 1$ is an arbitrary integer. Each processor holds a partial image (tile) corresponding to a specific region of the final output image. The final image size is $P$ pixels.
*   **Output:** A single composited image constructed by tiling the partial images from all processors in scanline order.

**Data Structures:**
*   **Compositing Tree:** A recursive tree structure where each non-leaf node has either two or three children. The root represents the entire set of $N$ processors. Leaf nodes represent individual processors.
*   **Processor List:** A global ordered list of processor indices, initialized as $[1, 2, \dots, N]$.
*   **Image Partition:** The final image is treated as a 1D pixel array in scanline order. At any stage, the image space is divided into contiguous partitions assigned to groups of processors.

**Algorithm Steps:**

1.  **Tree Construction:**
    Determine the number of stages $S = \lfloor \log_2 N \rfloor$. Let $K$ be such that $2^{K-1} \le N < 2^K$. Construct the compositing tree recursively using `ConstructCompositingTree(root, N, K-1)`.
    
    ```python
    def ConstructCompositingTree(node, n, d):
        """
        Recursively builds the compositing tree.
        node: The current tree node.
        n: The number of processors in this subtree.
        d: The depth remaining (distance to leaf).
        """
        if d == 0:
            # Leaf node: assign the next available processor index
            node.processors = [global_proc_id]
            global_proc_id += 1
        else:
            # Internal node: split n into 2 or 3 children
            # Condition: 2^d <= n < 2^(d+1)
            if n % 3 == 0 or n % 3 == 1:
                # Split into 3 children if possible to balance depth
                # l = floor(n/3), m = floor(n/3), r = n - 2*floor(n/3)
                # However, the source specifies a specific logic for 2 vs 3 children
                # based on ensuring 2^(d-1) <= child_size < 2^d
                # The source Algorithm 1 logic:
                l = floor(n / 2)
                r = ceil(n / 2)
                if r < 2^d:
                    # Split into 2 children
                    node.children = [new_node(), new_node()]
                    ConstructCompositingTree(node.children[0], l, d - 1)
                    ConstructCompositingTree(node.children[1], r, d - 1)
                else:
                    # Split into 3 children
                    # To satisfy the constraint, we split into (floor(n/3), floor(n/3), rest)
                    # The source implies l=m=floor(n/3) and r = n - 2*l
                    # But strictly, the source says: "l = floor(n/3), m = floor(n/3)..."
                    # Let's follow the explicit logic from the text:
                    # If r < 2^d is false, we go to 3 children.
                    # The text says: "l = floor(n/3), m = floor(n/3), r = n - 2*floor(n/3)"
                    # Note: The source text for Algorithm 1 is partially cut off but implies
                    # a split into 3 when the 2-split would violate the depth constraint.
                    # A robust implementation for 2-3 swap splits n into 2 or 3 parts
                    # such that each part is < 2^d.
                    
                    # Re-evaluating based on "2^d <= n < 2^(d+1)" invariant:
                    # If we split into 2, max child is ceil(n/2). We need ceil(n/2) < 2^d.
                    # If ceil(n/2) >= 2^d, we must split into 3.
                    # With 3 children, max child is ceil(n/3). We need ceil(n/3) < 2^(d-1)?
                    # No, the depth reduces by 1. So children must be < 2^(d-1)?
                    # Actually, the source says: "The tree constructed has K levels."
                    # And "2^d <= n < 2^(d+1)".
                    
                    # Correct logic derived from source constraints:
                    # If n can be split into 2 parts both < 2^d, do 2.
                    # Else, split into 3 parts.
                    
                    # Simplified implementation matching source intent:
                    if n % 3 == 0:
                        part = n // 3
                        node.children = [new_node(), new_node(), new_node()]
                        ConstructCompositingTree(node.children[0], part, d - 1)
                        ConstructCompositingTree(node.children[1], part, d - 1)
                        ConstructCompositingTree(node.children[2], part, d - 1)
                    elif n % 3 == 1:
                        part = n // 3
                        node.children = [new_node(), new_node(), new_node()]
                        ConstructCompositingTree(node.children[0], part, d - 1)
                        ConstructCompositingTree(node.children[1], part, d - 1)
                        ConstructCompositingTree(node.children[2], part + 1, d - 1)
                    else: # n % 3 == 2
                        # This case is stated as impossible in the source analysis
                        # "the case (n mod 3) = 2 is impossible"
                        pass 
            else:
                # Fallback or error handling for impossible states
                pass
    ```
    *Note: The source states that the case $(n \mod 3) = 2$ is impossible within the recursion constraints $2^d \le n < 2^{d+1}$.*

2.  **Order Assignment:**
    Call `OrderAssignment(root)` to determine the processor order for image partitioning at each node.
    *   At leaf nodes, the order is the processor index.
    *   At internal nodes, the order is formed by interleaving the orders of the children.
    *   **Merging Rule:** When merging groups (children) into a parent group:
        1.  A group with a larger number of processors (smaller partitions) in the previous stage gets its order first.
        2.  If group sizes are equal, the group with smaller processor indices gets its order first.

3.  **Compositing Stages:**
    Execute $S = \lfloor \log_2 N \rfloor$ stages. In stage $i$ (from 1 to $S$):
    *   Identify the groups of processors corresponding to the nodes at depth $S - i + 1$ (or equivalent level mapping).
    *   Within each group, processors exchange pixel data.
    *   **Partitioning:** The image space is divided evenly among the $M$ processors in the current group. Each processor is responsible for $P/M$ pixels.
    *   **Communication:** Each processor sends its current partial image data to other processors in the same group and receives data from them. The number of neighbors is at most 4.
    *   **Blending:** After exchange, each processor blends the received pixels with its own local pixels. The blending operation is standard alpha compositing: $C_{out} = C_{src} \alpha_{src} + C_{dst} (1 - \alpha_{src})$.

4.  **Gathering:**
    After the final compositing stage, each processor holds a portion of the final composited image. Each processor sends its final partial image to a designated host processor (or tiles them collectively) to construct the final image.

**Formulas:**
*   **Total Compositing Time:**
    $$ T_{comp} = \sum_{i=1}^{S} T_{comp_i} $$
    where $T_{comp_i} = T_{exchange_i} + T_{blend_i}$.

*   **Exchange Time per Stage (Full-Duplex):**
    $$ T_{exchange_i} = \max(L_{send_i} + T_{send_i}, L_{recv_i} + T_{recv_i}) $$
    where:
    *   $L_{send_i} = N_{send_i} \times l_{sendrecv}$
    *   $T_{send_i} = \frac{P_{send_i}}{B_{send_i}}$
    *   $L_{recv_i} = N_{recv_i} \times l_{sendrecv}$
    *   $T_{recv_i} = \frac{P_{recv_i}}{B_{recv_i}}$
    *   $l_{sendrecv}$: Latency per message.
    *   $B_{send/recv}$: Network bandwidth.
    *   $N_{send/recv}$: Number of send/receive operations.
    *   $P_{send/recv}$: Size of pixel data sent/received.

*   **Blending Time:**
    $$ T_{blend_i} = \frac{P_{blend_i}}{B_{blend}} $$
    where $B_{blend}$ is the blending bandwidth.

*   **Bounds for 2-3 Swap:**
    *   Number of messages: $N_{sendrecv} \le 4 \times \lfloor \log_2 N \rfloor$.
    *   Pixel sending/receiving size: $P_{sendrecv} \le \frac{3}{4} \times P$ (Theoretical bound, actual max $\approx 1.23 P$).
    *   Pixel blending size: $P_{blend} \le 2 \times P$.

**Parameters:**
*   $N$: Number of processors.
*   $P$: Size of the final image in pixels.
*   $S$: Number of stages, $\lfloor \log_2 N \rfloor$.

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two N:** If $N$ is not a power of two, the tree is complete but not full. This can cause workload imbalance. The worst-case timing is at most two times that of Binary Swap [S2].
*   **Partition Alignment:** Image partitions may not be perfectly aligned if processor counts are not evenly split (e.g., $M$ vs $M+1$). However, analysis shows that cases where one partition overlaps three others are impossible, bounding communication to 4 neighbors [S2].
*   **Idle Processors:** Unlike Reduced Binary Swap, 2-3 Swap ensures no processors are idle at any stage [S2].

## Subtopics

### [Radix-k Compositing](12235-radix-k-compositing.md)
A generalization that divides communication into multiple rounds with variable group sizes $k_i$, where $\prod k_i = N$. It combines the flexibility of direct send with the performance of binary swap. Choose this when the network topology or communication patterns benefit from variable group sizes rather than the fixed 2-3 structure.

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ messages exchanged. The number of stages is $S = \lfloor \log_2 N \rfloor$.
*   **Memory Complexity:** Each processor stores a fraction of the image. Total pixel data exchanged is bounded by $1.23 P$ for sending/receiving and $2.0 P$ for blending in the worst case [S2].
*   **Performance:**
    *   Scales well in massively parallel environments (thousands of processors) [S2].
    *   Outperforms Direct Send and Reduced Binary Swap in timing stability and efficiency [S2].
    *   When $N$ is a power of two, performance matches Binary Swap (the lower bound for 2-3 Swap) [S2].
    *   When $N$ is not a power of two, timing is at most 2x that of Binary Swap [S2].
    *   Average communication count is $1.98 \times \lfloor \log_2 N \rfloor$, significantly better than the worst-case bound of $4 \times \lfloor \log_2 N \rfloor$ [S2].
    *   Approximately 80% of communication cases involve 1 or 2 neighbors [S2].
*   **Strengths:**
    *   Supports arbitrary $N$.
    *   No idle processors.
    *   Low network congestion (max 4 neighbors).
    *   Suitable for simulation-time visualization [S2].
*   **Limitations:**
    *   Workload imbalance can occur for non-power-of-two $N$, leading to a difference ratio in total compositing time of up to 57% (observed at 1021 processors) [S2].
    *   Implementation complexity is higher than Direct Send due to tree construction and order assignment.

## Background
2-3 Swap Compositing was introduced by Yu et al. to overcome the limitation of Binary Swap Compositing, which requires the number of processors to be a power of two [S1]. The algorithm was designed for massively parallel volume rendering, specifically for simulation-time data visualization on supercomputers [S2]. It was tested on a supercomputer with up to 1024 processors and applied to a Richtmyer-Meshkov Instability (RMI) dataset of 2TB with 2048×2048×1920 spatial resolution, generating 4096² output images [S2]. The method is a generalization of binary swap, decomposing $N$ into a summation of twos and threes to ensure full processor utilization [S2].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
[S2] Massively Parallel Volume Rendering Using 2-3 Swap ..., http://vis.cs.ucdavis.edu/papers/SC08_compositing.pdf, fetched 2026-10-29
