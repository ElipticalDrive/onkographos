# Greedy Merging for VV Codes

Also called: Least-probable leaf merging

## Overview
Greedy merging is a heuristic method for constructing Variable-length Variable-length (VV) codes by iteratively merging the least-probable leaf in a parse tree to reduce code length, aiming to minimize coding rate [S1, unofficial]. It builds on [Variable Bit Length Encoding](7195-variable-bit-length-encoding.md) [G]. Unlike exhaustive search, it is not guaranteed to produce the optimal coding rate [S1, unofficial].

## How it works
**Inputs:** Source symbol probabilities (weights) and the starting string length $W$ [S1, unofficial].
**Outputs:** A parse tree defining the VV code [S1, unofficial].

**Data Structures:**
*   **Parse Tree:** A binary tree where leaves represent codewords. Each node stores a `weight` (probability) [S1, unofficial].
*   **Leaf Set:** A collection of current leaf nodes, sortable by weight [G].

**Steps:**
1.  Initialize the parse tree as a full binary tree of depth $W$, where each leaf corresponds to a unique binary string of length $W$. Assign weights to leaves based on symbol probabilities [G].
2.  Identify the leaf node $l_{min}$ with the smallest weight in the current parse tree [S1, unofficial].
3.  Merge $l_{min}$:
    *   Select the parent node $p$ of $l_{min}$.
    *   Prune the children of $p$ (effectively removing $l_{min}$ and its sibling from the set of codewords).
    *   Convert $p$ into a leaf node. The new weight of $p$ is the sum of the weights of its former children [S1, unofficial].
4.  Update the coding rate. The coding rate $R$ is defined as the ratio of the weighted sum of code lengths to the weighted sum of parse tree path lengths [S1, unofficial].
5.  Repeat steps 2–4 until the longest code length in the tree equals the starting string length $W$ [S1, unofficial].

**Formula:**
The coding rate $R$ is calculated as:
$$ R = \frac{\sum_{i} w_i l_i}{\sum_{i} w_i d_i} $$
Where:
*   $w_i$ is the weight (probability) of leaf $i$ [S1, unofficial].
*   $l_i$ is the length of the codeword for leaf $i$ [S1, unofficial].
*   $d_i$ is the depth of leaf $i$ in the parse tree (weighted parse tree path length) [S1, unofficial].

**Parameters:**
*   $W$: Starting string length (e.g., 7, 8, 9, 10 bits) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Suboptimality:** Greedy merging is not guaranteed to produce the optimal coding rate. It may produce a suboptimal code compared to non-greedy selection [S1, unofficial].
*   **Counterexample:** In a block size of 3 with $p=0.6, q=0.4$, greedy merging on leaf '111' (weight 0.064) yields a coding rate of 0.98. A non-greedy merge on leaf '011' (weight 0.096) yields a coding rate of 0.9797 [S1, unofficial].
*   **Uncertainty of Change:** At each step, the change in the overall coding rate may go up or down, making it difficult to predict the global impact of a local merge without full recalculation [S1, unofficial].

## Subtopics

### Exhaustive Search for Optimal VV Codes
The only known method for finding an optimal VV code, involving trying all possible merge sequences and parse trees [S1, unofficial]. The search space is vast because there are a variety of ways to form parse trees, and for each tree, Huffman code lengths must be computed [S1, unofficial]. Huffman code adaptation methods cannot be used to simplify this, as VV code changes alter the input set itself [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The coding rate is resilient to changes if the starting string length is longer (e.g., the (10,10) coding rate remains largely constant, changing from 0.476691 to 0.476732 as the longest code length changes from 30 to 15) [S1, unofficial].
*   **Strengths:** Simple to implement; efficient; provides a good approximation of optimal codes [G].
*   **Limitations:** Not optimal; may produce suboptimal codes in specific probability distributions [S1, unofficial].
*   **When to use:** When computational resources are limited and an approximate optimal VV code is sufficient [G].
*   **When not to use:** When the absolute optimal coding rate is required and computational cost is not a constraint [G].

## Background
Discussed in the context of length-limited data transformation and compression [S1, unofficial]. The method is compared to Tunstall–Huffman approaches, noting similarities in the reasoning behind merging and extension processes [S1, unofficial]. The suboptimality of the greedy approach was proven by counterexample [S1, unofficial].

## Sources
[S1] Length-Limited Data Transformation and Compression, https://digital.library.unt.edu/ark:/67531/metadc874691/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
