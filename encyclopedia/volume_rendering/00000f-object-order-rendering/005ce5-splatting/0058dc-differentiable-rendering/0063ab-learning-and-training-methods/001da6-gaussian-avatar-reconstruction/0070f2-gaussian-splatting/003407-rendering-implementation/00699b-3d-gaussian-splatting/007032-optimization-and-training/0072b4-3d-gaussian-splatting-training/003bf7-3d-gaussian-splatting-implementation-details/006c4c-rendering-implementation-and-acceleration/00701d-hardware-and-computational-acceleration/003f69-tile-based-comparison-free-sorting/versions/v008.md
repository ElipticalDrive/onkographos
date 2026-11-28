# Tile-Based Comparison-Free Sorting

Also called: Comparison-Free Depth Sorting, Tile-Local Sorting

## Overview
Tile-Based Comparison-Free Sorting restricts depth sorting operations to individual image tiles to avoid global synchronization, reduce hardware complexity, and achieve fixed latency per output [S1]. It solves the bottleneck of sorting millions of Gaussian points globally by leveraging the fact that alpha blending only requires correct depth ordering within the local tile [S1]. Unlike full-scene radix sort, which incurs high memory overhead and variable latency, this method provides deterministic, linear delay cycles suitable for low-power hardware [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of 3D Gaussians projected into screen space, each associated with a tile ID and a view-space depth value.
*   **Output:** Gaussians ordered front-to-back within each tile, ready for alpha blending.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian primitive carries a `tile_id` (identifying the $16 \times 16$ tile it overlaps) and `depth` (view-space depth).
*   **Element Vector Table (EVT):** A bit-vector of size $N$ (where $N$ is the number of Gaussians in the current tile/batch), initialized to all 1s to track unsorted elements [S1].

**Algorithm Steps:**
The sorting process identifies the Largest Element (LE) iteratively until all elements are sorted [S1].

1.  **Initialization:** Initialize the EVT with all 1s. This indicates that all $N$ elements are currently candidates for being the largest remaining element [S1].
2.  **Iterative Selection:** Repeat the following steps $N$ times (once for each Gaussian to be sorted):
    a.  **Concurrent Phase:** Process the $N$ $n$-bit depth values using $k$ parallel clusters. Each cluster handles a partition of the elements. Starting from the most significant bit (MSB), the clusters filter out smaller values through bitwise AND operations and multiplexer-based selection in cascaded blocks [S1].
    b.  **Sequential Phase:** Pass the outputs from the concurrent phase to a linear array of $k$ cascaded blocks to further refine the selection and identify the single Largest Element (LE) among the candidates marked by the EVT [S1].
    c.  **Conflict Resolution:** If multiple elements have identical depth values, the EVT may return multiple candidates. Resolve this using bitwise inverter, bitwise AND, and an adder to compute the unique index of the first candidate:
        $$FO_{NEW} = Fo \& (\sim Fo + 1'b1)$$
        where $Fo$ is the output vector from the selection phase [S1].
    d.  **Update EVT:** Clear the bit corresponding to the identified LE in the EVT (set to 0). This removes the sorted element from future consideration [S1].
3.  **Termination:** The loop ends when the EVT contains all 0s, signaling that all $N$ elements have been sorted [S1].

**Parameters and Optimizations:**
*   **Sorting Key:** The primary key is $\langle \text{tile-id}, \text{depth} \rangle$ [S1]. Sorting is performed independently per tile.
*   **Sign Bit Skipping:** Since near-plane culling excludes points with depth $< 0$, the sign bit of the depth value is always 0. The algorithm skips operations on the sign bit to reduce computational overhead [S1].
*   **Bit Width ($n$):** Determined by the precision required for view-space depth.

**Edge Cases and Pitfalls:**
*   **Duplicate Depths:** The conflict resolution logic ($FO_{NEW}$) is critical to handle duplicate depth values correctly, ensuring deterministic ordering even when Gaussians share the same depth [S1].
*   **Variable N:** The number of Gaussians per tile varies significantly across scenes and views. The hardware must handle this variability without pipeline stalls [S1].

## Subtopics

### Parallel Comparison-Free Sorting
A tile-based, comparison-free sorting algorithm for Gaussian points that uses an Element Vector Table (EVT) to identify the largest element in $O(N)$ deterministic cycles, avoiding the high memory overhead of radix sort [S1]. It employs concurrent and sequential phases to filter elements based on bit significance, optimized by skipping the sign bit and resolving data conflicts via bitwise operations [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ deterministic cycles for sorting $N$ elements [S1].
*   **Hardware Complexity:** Low gate count compared to radix sort, as it uses simple logic gates (AND, OR, MUX) rather than complex parallel scans or prefix sums [S1].
*   **Latency:** Fixed, predictable latency per output, avoiding the latency spikes associated with global synchronization in full-scene sorting [S1].
*   **Performance:** Achieved 129 FPS at Full HD resolution on the described hardware accelerator, offering 3× higher throughput and 5.98× smaller area compared to prior 3DGS accelerators (e.g., GSCore) [S1].
*   **Strengths:** Ideal for low-power AR/VR hardware and predictable scheduling; avoids excess traffic from global resorting [S1].
*   **Limitations:** Requires hardware implementation to realize the deterministic cycle benefits; software emulation may not match the efficiency gains [S1].

## Background
Originating from hardware accelerator designs for 3D Gaussian Splatting, this method addresses the computational bottleneck of sorting millions of Gaussian points [S1]. Traditional global sorting is resource-intensive and unsuitable for real-time rendering on constrained hardware [S1]. The approach builds on parallel comparison-free sorting algorithms [21, 22] (cited in [S1]) but introduces optimizations specific to 3DGS, such as sign bit skipping and conflict resolution for duplicate depths [S1]. It was implemented in a hardware accelerator achieving real-time Full HD rendering [S1].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
