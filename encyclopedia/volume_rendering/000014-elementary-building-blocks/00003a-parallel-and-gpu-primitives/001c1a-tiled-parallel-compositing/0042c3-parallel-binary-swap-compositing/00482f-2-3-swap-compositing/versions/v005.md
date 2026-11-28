# 2-3 Swap Compositing

Also called: 2-3 Swap, 2-3 Swap Image Compositing

## Overview
2-3 Swap Compositing is a parallel image compositing algorithm designed to combine independently rendered image tiles from multiple processors into a single final image, specifically addressing the limitation that the number of processors $N$ must be a power of two in standard Binary Swap Compositing [S1, S2]. It allows an arbitrary number of processors to be used while maintaining the $O(N \log N)$ message complexity of binary swap, ensuring that all participating processors are fully utilized without idle stages [S2]. Unlike Direct Send, which suffers from network congestion and $O(N^2)$ messages, 2-3 Swap limits communication to at most four neighbors per stage, making it suitable for massively parallel environments with thousands of processors [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** $N$ processors, where $N > 1$ is an arbitrary integer. Each processor holds a partial image (tile) corresponding to a specific region of the final output image [S2]. The final image size is $P$ pixels [S2].
*   **Output:** A single composited image constructed by tiling the partial images from all processors in scanline order [S2].

**Data Structures:**
*   **Compositing Tree:** A recursive tree structure where each non-leaf node has either two or three children [S2]. The root represents the entire set of $N$ processors [S2]. Leaf nodes represent individual processors [S2].
*   **Processor List:** A global ordered list of processor indices, initialized as $[1, 2, \dots, N]$ [S2].
*   **Image Partition:** The final image is treated as a 1D pixel array in scanline order [S2]. At any stage, the image space is divided into contiguous partitions assigned to groups of processors [S2].

**Algorithm Steps:**

1.  **Tree Construction:**
    Determine the number of stages $S = \lfloor \log_2 N \rfloor$ [S2]. Let $K$ be such that $2^{K-1} \le N < 2^K$ [S2]. Construct the compositing tree recursively using `ConstructCompositingTree(root, N, K-1)` [S2].
    
    ```python
    class Node:
        def __init__(self):
            self.children = []
            self.processors = []

    def ConstructCompositingTree(node, n, d, proc_counter):
        """
        Recursively builds the compositing tree.
        node: The current tree node.
        n: The number of processors in this subtree.
        d: The depth remaining (distance to leaf).
        proc_counter: Mutable list holding the global processor index counter.
        """
        if d == 0:
            # Leaf node: assign the next available processor index
            node.processors = [proc_counter[0]]
            proc_counter[0] += 1
        else:
            # Internal node: split n into 2 or 3 children
            l = n // 2
            r = (n + 1) // 2  # ceil(n/2)
            if r < 2 ** d:
                # Split into 2 children
                node.children = [Node(), Node()]
                ConstructCompositingTree(node.children[0], l, d - 1, proc_counter)
                ConstructCompositingTree(node.children[1], r, d - 1, proc_counter)
            else:
                # Split into 3 children
                # The case (n mod 3) == 2 is mathematically impossible under the invariant 2^d <= n < 2^(d+1)
                l = n // 3
                m = l
                r = n - 2 * l
                node.children = [Node(), Node(), Node()]
                ConstructCompositingTree(node.children[0], l, d - 1, proc_counter)
                ConstructCompositingTree(node.children[1], m, d - 1, proc_counter)
                ConstructCompositingTree(node.children[2], r, d - 1, proc_counter)
    ```

2.  **Order Assignment:**
    Call `OrderAssignment(root)` to determine the processor order for image partitioning at each node [S2]. At leaf nodes, the order is the processor index [S2]. At internal nodes, the order is formed by interleaving the orders of the children [S2]. When merging groups (children) into a parent group:
    *   Rule 1: A group with a larger number of processors (smaller partitions) in the previous stage gets its order first [S2].
    *   Rule 2: If group sizes are equal, the group with smaller processor indices gets its order first [S2].
    
    ```python
    def OrderAssignment(node):
        if not node.children:
            return node.processors
        # Recursively get children orders
        child_orders = [OrderAssignment(child) for child in node.children]
        # Sort children: larger processor count first, then smaller min index
        child_orders.sort(key=lambda order: (-len(order), order[0]))
        # Concatenate in sorted order
        node.processors = []
        for order in child_orders:
            node.processors.extend(order)
        return node.processors
    ```

3.  **Compositing Stages:**
    Execute $S = \lfloor \log_2 N \rfloor$ stages [S2]. In stage $i$ (from 1 to $S$):
    *   Identify the groups of processors corresponding to the nodes at depth $S - i + 1$ [S2].
    *   Within each group, processors exchange pixel data [S2].
    *   **Partitioning:** The image space is divided evenly among the $M$ processors in the current group. Each processor is responsible for $P/M$ pixels [S2].
    *   **Communication:** Each processor sends its current partial image data to other processors in the same group and receives data from them. The number of neighbors is at most 4 [S2].
    *   **Blending:** After exchange, each processor blends the received pixels with its own local pixels using standard alpha compositing: $C_{out} = C_{src} \alpha_{src} + C_{dst} (1 - \alpha_{src})$ [G].

4.  **Gathering:**
    After the final compositing stage, each processor holds a portion of the final composited image [S2]. Each processor sends its final partial image to a designated host processor, which tiles them to produce the final image [S2].

**Formulas:**
*   **Total Compositing Time:**
    $$ T_{comp} = \sum_{i=1}^{S} T_{comp_i} $$
    where $T_{comp_i} = T_{exchange_i} + T_{blend_i}$ [S2].

*   **Exchange Time per Stage (Full-Duplex):**
    $$ T_{exchange_i} = \max(L_{send_i} + T_{send_i}, L_{recv_i} + T_{recv_i}) $$
    where:
    *   $L_{send_i} = N_{send_i} \times l_{sendrecv}$
    *   $T_{send_i} = \frac{P_{send_i}}{B_{send_i}}$
    *   $L_{recv_i} = N_{recv_i} \times l_{sendrecv}$
    *   $T_{recv_i} = \frac{P_{recv_i}}{B_{recv_i}}$
    *   $l_{sendrecv}$: Latency per message [S2].
    *   $B_{send/recv}$: Network bandwidth [S2].
    *   $N_{send/recv}$: Number of send/receive operations [S2].
    *   $P_{send/recv}$: Size of pixel data sent/received [S2].

*   **Blending Time:**
    $$ T_{blend_i} = \frac{P_{blend_i}}{B_{blend}} $$
    where $B_{blend}$ is the blending bandwidth [S2].

*   **Bounds for 2-3 Swap:**
    *   Number of messages: $N_{sendrecv} \le 4 \times \lfloor \log_2 N \rfloor$ [S2].
    *   Pixel sending/receiving size: $P_{sendrecv} \le \frac{3}{4} \times P$ (theoretical bound; actual maximum $\approx 1.23 P$) [S2].
    *   Pixel blending size: $P_{blend} \le 2 \times P$ [S2].

**Parameters:**
*   $N$: Number of processors [S2].
*   $P$: Size of the final image in pixels [S2].
*   $S$: Number of stages, $\lfloor \log_2 N \rfloor$ [S2].

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two N:** If $N$ is not a power of two, the tree is complete but not full. This can cause workload imbalance. The worst-case timing is at most two times that of Binary Swap [S2].
*   **Partition Alignment:** Image partitions may not be perfectly aligned if processor counts are not evenly split (e.g., $M$ vs $M+1$). However, analysis shows that cases where one partition overlaps three others are impossible, bounding communication to 4 neighbors [S2].
*   **Idle Processors:** Unlike Reduced Binary Swap, 2-3 Swap ensures no processors are idle at any stage [S2].

## Subtopics

### [Radix-k Compositing](12235-radix-k-compositing.md)
A generalization that divides communication into multiple rounds with variable group sizes $k_i$, where $\prod k_i = N$. It combines the flexibility of direct send with the performance of binary swap. Choose this when the network topology or communication patterns benefit from variable group sizes rather than the fixed 2-3 structure.

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ messages exchanged. The number of stages is $S = \lfloor \log_2 N \rfloor$ [S2].
*   **Memory Complexity:** Each processor stores a fraction of the image. Total pixel data exchanged is bounded by $1.23 P$ for sending/receiving and $2.0 P$ for blending in the worst case [S2].
*   **Performance:**
    *   Scales well in massively parallel environments (thousands of processors) [S2].
    *   Outperforms Direct Send and Reduced Binary Swap in timing stability and efficiency [S2].
    *   When $N$ is a power of two, performance matches Binary Swap (the lower bound for 2-3 Swap) [S2].
    *   When $N$ is not a power of two, timing is at most 2x that of Binary Swap [S2].
    *   Average communication count is $1.98 \times \lfloor \log_2 N \rfloor$, significantly better than the worst-case bound of $4 \times \lfloor \log_2 N \rfloor$ [S2].
    *   Approximately 80% of communication cases involve 1 or 2 neighbors [S2].
*   **Strengths:**
    *   Supports arbitrary $N$ [S2].
    *   No idle processors [S2].
    *   Low network congestion (max 4 neighbors) [S2].
    *   Suitable for simulation-time visualization [S2].
*   **Limitations:**
    *   Workload imbalance can occur for non-power-of-two $N$, leading to a difference ratio in total compositing time of up to 57% (observed at 1021 processors) [S2].
    *   Implementation complexity is higher than Direct Send due to tree construction and order assignment [G].

## Background
2-3 Swap Compositing was introduced by Yu et al. to overcome the limitation of Binary Swap Compositing, which requires the number of processors to be a power of two [S1]. The algorithm was designed for massively parallel volume rendering, specifically for simulation-time data visualization on supercomputers [S2]. It was tested on a supercomputer with up to 1024 processors and applied to a Richtmyer-Meshkov Instability (RMI) dataset of 2TB with 2048×2048×1920 spatial resolution, generating 4096² output images [S2]. The method is a generalization of binary swap, decomposing $N$ into a summation of twos and threes to ensure full processor utilization [S2].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
[S2] Massively Parallel Volume Rendering Using 2-3 Swap ..., http://vis.cs.ucdavis.edu/papers/SC08_compositing.pdf, fetched 2026-10-29
