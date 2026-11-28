# VQ with Undefined Data

Also called: Partial VQ, VQ for undefined residuals

## Overview
This method extends vector quantization (VQ) to handle residual blocks where some data values are undefined, such as those outside the defined domain bounds [S1, unofficial]. It solves the problem of quantization error accumulation in sparse or irregularly shaped volumes by ignoring undefined components during distance calculations and centroid updates [S1, unofficial]. Unlike standard VQ which treats all dimensions equally, this approach preserves convergence properties while reducing error for valid data points [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of residual vectors $V = \{v_1, v_2, \dots, v_N\}$ where each vector $v_i \in \mathbb{R}^D$ may contain undefined components.
- Input: A set of cluster centroids $C = \{c_1, c_2, \dots, c_K\}$ where $c_j \in \mathbb{R}^D$.
- Input: A mask $M_i \in \{0, 1\}^D$ for each vector $v_i$, where $M_{i,d} = 1$ if component $d$ is defined, and $0$ otherwise.
- Output: Cluster assignments $A = \{a_1, \dots, a_N\}$ where $a_i \in \{1, \dots, K\}$.
- Output: Updated centroids $C' = \{c'_1, \dots, c'_K\}$.

**Data Structures:**
- Vectors are stored as dense arrays of size $D$.
- Masks are stored as boolean or integer arrays of size $D$.
- Centroids are stored as dense arrays of size $D$.

**Algorithm Steps:**
The method modifies the standard k-means clustering algorithm [S1, unofficial].

1. **Initialization:** Initialize centroids $C$ using a standard method (e.g., k-means++). [G]

2. **Assignment Step:** For each vector $v_i$:
   - Compute the distance to each centroid $c_j$, ignoring undefined components.
   - The squared distance is:
     $$ d^2(v_i, c_j) = \sum_{d=1}^{D} M_{i,d} \cdot (v_{i,d} - c_{j,d})^2 $$
   - Assign $v_i$ to the cluster $a_i$ with the minimum distance:
     $$ a_i = \arg\min_{j} d^2(v_i, c_j) $$

3. **Update Step:** For each cluster $j \in \{1, \dots, K\}$:
   - Let $S_j = \{i \mid a_i = j\}$ be the set of indices of vectors assigned to cluster $j$.
   - If $S_j$ is empty, keep $c_j$ unchanged. [G]
   - Otherwise, update the centroid $c'_j$ by computing the mean of the defined components of the assigned vectors:
     $$ c'_{j,d} = \frac{\sum_{i \in S_j} M_{i,d} \cdot v_{i,d}}{\sum_{i \in S_j} M_{i,d}} $$
     Note: The denominator is the count of defined components for dimension $d$ across all vectors in $S_j$. If the denominator is zero, $c'_{j,d}$ remains unchanged or is set to zero. [G]

4. **Convergence:** Repeat steps 2 and 3 until the assignments $A$ do not change or the change in centroids is below a threshold.

**Parameters:**
- $K$: Number of clusters. Typical values depend on the desired compression ratio.
- $\tau$: Convergence threshold. Not given in the sources.

**Edge Cases and Pitfalls:**
- If a vector has all components undefined, it cannot be assigned meaningfully. [G]
- If a cluster has no vectors assigned, the centroid update step is skipped. [G]
- Precision issues may arise if the number of defined components is very small, leading to unstable means. [G]

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N \cdot K \cdot D \cdot T)$, where $T$ is the number of iterations. The mask operations add a small constant overhead.
- **Memory Complexity:** $O(N \cdot D + K \cdot D)$ for storing vectors and centroids.
- **Strengths:** Reduces quantization error for valid data by ignoring undefined components. Preserves convergence properties of k-means.
- **Limitations:** Requires mask information. Performance may degrade if many components are undefined.
- **When to use:** When dealing with sparse volumes or domains with irregular boundaries.
- **When not to use:** When all data components are defined (standard VQ is simpler).

## Background
This method is an extension of k-means clustering to partially defined data, based on the property that k-means can handle missing values by ignoring them during distance and mean calculations [LFWV03, cited in S1, unofficial]. It was applied to volume rendering compression in "Compressed Random-Access Trees for Spatially Coherent Data" [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
