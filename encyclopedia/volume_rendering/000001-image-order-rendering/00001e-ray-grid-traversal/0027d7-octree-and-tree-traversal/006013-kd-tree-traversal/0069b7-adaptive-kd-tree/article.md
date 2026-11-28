# Adaptive Kd Tree


## Overview
The Adaptive Kd Tree is a hierarchical acceleration structure for volume rendering that stores data values exclusively in the leaves of the tree [S1, unofficial]. It addresses the problem of efficient ray traversal through sparse volumetric data by allowing subdivision in all directions, offering more flexibility than octrees which are restricted to center-based subdivisions [S1, unofficial]. It is distinguished from its sibling, the Hybrid Kd Tree, by its specific leaf-only storage strategy and construction method [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A set of volumetric data points or voxel values.
*   Output: A balanced kd tree structure where internal nodes contain splitting planes and leaf nodes contain the actual data values.

**Data Structures:**
The tree is a binary tree. Internal nodes store the axis and position of the splitting plane. Leaf nodes store the volumetric values associated with the region they cover. The tree is constructed to be balanced.

**Construction Steps:**
1.  Collect all data points or voxel centers.
2.  Select a split direction (e.g., x, y, or z axis).
3.  Sort the points with respect to the chosen split direction.
4.  Identify the median element in the sorted list.
5.  Split the data at the median, creating two child nodes.
6.  Recursively apply steps 2-5 to each child node until a stopping criterion is met (e.g., leaf nodes contain single values or a minimum region size).
7.  Store the data values only in the resulting leaf nodes [S1, unofficial].

**Formulas:**
Let $P$ be the set of points. For a split direction $d$, let $P_{sorted}$ be $P$ sorted by coordinate $d$. The split point $s$ is:
$$ s = \text{median}(P_{sorted}) $$
The left child contains points $p \in P$ where $p_d < s$, and the right child contains points where $p_d \ge s$.

**Parameters:**
*   Split direction selection: Not specified in sources [G]. Typically chosen to minimize variance or balance node counts.
*   Stopping criterion: Not specified in sources [G].

**Edge Cases and Pitfalls:**
*   Balancing: Unbalanced kd trees are difficult to balance and can degrade performance. The median split method ensures balance [S1, unofficial].
*   Stack usage: Traditional kd tree traversal requires a stack for recursive traversal. The adaptive kd tree itself does not eliminate the need for a stack during traversal, though related algorithms like the kd restart algorithm do [S1, unofficial].

## Subtopics

### KD-Tree for Volumetric Data
A variant of BSP using splitting planes aligned with data axes for adaptive partitioning [S2, unofficial]. It constructs shallow hierarchies in real-time for moderately sized volumes, supporting efficient empty space skipping in sparse rendering scenarios [S2, unofficial]. It is particularly beneficial in ray casting where it reduces traversal costs for volumetric data exhibiting high sparsity [S2, unofficial]. Construction involves selecting splitting planes to balance empty and filled regions [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N \log N)$ due to sorting at each level for median selection [S1, unofficial]. Traversal complexity depends on tree balance and data sparsity.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of leaf nodes storing values [S1, unofficial].
*   **Strengths:** Balanced structure ensures predictable traversal performance. Leaf-only storage can be memory efficient for sparse data if internal nodes are small.
*   **Limitations:** Construction can be costly for large datasets. Not explicitly stack-less, so traversal may require stack management unless combined with specific traversal algorithms like kd restart [S1, unofficial].
*   **When to use:** When data sparsity is high and a balanced tree is desired for consistent traversal performance.
*   **When not to use:** When real-time construction is required for very large volumes, or when stack-less traversal is a strict requirement without additional algorithmic modifications.

## Background
The Adaptive Kd Tree was introduced as a variant by Friedman et al. [S1, unofficial]. It is used in volume rendering to accelerate ray traversal through volumetric datasets. The approach contrasts with traditional kd trees that may store data in internal nodes or use different balancing strategies.

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
[S2] volume rendering, https://grokipedia.com/page/Volume_rendering, fetched 2026-10-26 (unofficial)
