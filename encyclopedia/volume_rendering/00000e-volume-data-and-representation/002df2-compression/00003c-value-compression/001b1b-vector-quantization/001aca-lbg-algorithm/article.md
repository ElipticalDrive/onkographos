# LBG Algorithm
Also called: Linde-Buzo-Gray algorithm, Lloyd-Max algorithm, k-means clustering
## Overview
The Linde-Buzo-Gray (LBG) algorithm is an iterative vector quantization method that partitions a set of input vectors into quantization regions and updates codebook entries to minimize the distortion between input vectors and their assigned codebook entries [S1, unofficial]. It solves the problem of finding an optimal codebook for lossy compression by iteratively refining centroid positions [S1, unofficial]. It sets itself apart from its sibling [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md) by using a flat, iterative refinement process rather than a tree-based hierarchy, and from [Modified Vector Quantization](1130-modified-vector-quantization-lbg-variant.md) by relying on random initialization and nearest-neighbor searches rather than deterministic splitting [S1, S1, unofficial].
## How it works
**Inputs:**
- Set of input vectors $I \subset \mathbb{R}^n$.
- Target number of codebook entries $m = 2^r$, where $r$ is the bit-rate [S1, unofficial].
- Convergence threshold $\epsilon$ [S1, unofficial].
- Distance metric $\delta(X, Y)$, typically squared Euclidean distance [S1, unofficial].

**Outputs:**
- Final codebook $C = \{Y_i\}_{i=1}^m$ [S1, unofficial].
- Partition $\{V_i\}_{i=1}^m$ of $I$, where each $X \in I$ is mapped to an index [S1, unofficial].

**Data Structures:**
- Codebook $C$: Array of $m$ vectors, each of dimension $n$.
- Input set $I$: Array of vectors, each of dimension $n$.
- Quantization regions $V_i$: Implicitly defined by assignment; can be stored as an index map mapping each $X \in I$ to an integer $i \in \{1, \dots, m\}$.

**Steps:**
1. **Initialize:** Start with an initial codebook $C^{(0)} = \{Y_i^{(0)}\}_{i=1}^m \subset \mathbb{R}^n$. Set iteration counter $k = 0$, previous distortion $D^{(0)} = 0$, and select convergence threshold $\epsilon$ [S1, unofficial]. The initial codebook is usually obtained by a splitting technique: start with the centroid of $I$ as one entry, add a random offset to create a second entry, run LBG until convergence, and repeat doubling until $m$ entries are reached [S1, unofficial].
2. **Assign regions:** For each codebook entry $Y_i^{(k)}$, find the quantization region $V_i^{(k)} = \{X \in I : \delta(X, Y_i^{(k)}) < \delta(X, Y_j^{(k)}) \text{ for all } j \neq i, j=1,\dots,m\}$ [S1, unofficial]. This assigns each input vector to its nearest codebook entry.
3. **Compute distortion:** Calculate total distortion $D^{(k)} = \sum_{i=1}^m \sum_{X \in V_i^{(k)}} \delta(X, Y_i^{(k)})$ [S1, unofficial].
4. **Check convergence:** If $\frac{D^{(k-1)} - D^{(k)}}{D^{(k)}} < \epsilon$, stop. Otherwise, continue [S1, unofficial].
5. **Update codebook:** Increment $k$. Find new codebook $\{Y_i^{(k)}\}_{i=1}^m$ by calculating the centroids of each cell $V_i^{(k-1)}$. Go to step 2 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lbq_algorithm(I, m, epsilon, delta_func):
    """
    I: list of input vectors (each a list/array of floats)
    m: number of codebook entries
    epsilon: convergence threshold
    delta_func: distance function delta(X, Y)
    """
    # Step 1: Initialize
    # Splitting technique to generate initial codebook
    C = initialize_codebook_splitting(I, m, delta_func)
    k = 0
    D_prev = 0.0
    
    while True:
        # Step 2: Assign regions
        # V is a list of lists, where V[i] contains all X in I closest to C[i]
        V = [[] for _ in range(m)]
        for X in I:
            # Find nearest codebook entry
            min_dist = float('inf')
            best_idx = -1
            for i in range(m):
                d = delta_func(X, C[i])
                if d < min_dist:
                    min_dist = d
                    best_idx = i
            V[best_idx].append(X)
        
        # Step 3: Compute distortion
        D_curr = 0.0
        for i in range(m):
            for X in V[i]:
                D_curr += delta_func(X, C[i])
        
        # Step 4: Check convergence
        if k > 0:
            if (D_prev - D_curr) / D_curr < epsilon:
                break
        
        # Step 5: Update codebook
        new_C = []
        for i in range(m):
            if len(V[i]) == 0:
                # Pitfall: Empty cell problem. 
                # Standard LBG does not detect these automatically [S1, unofficial].
                # [G] Keep previous centroid or reinitialize randomly to avoid collapse.
                new_C.append(C[i])
            else:
                # Calculate centroid
                centroid = [0.0] * len(I[0])
                for X in V[i]:
                    for j in range(len(X)):
                        centroid[j] += X[j]
                for j in range(len(X)):
                    centroid[j] /= len(V[i])
                new_C.append(centroid)
        C = new_C
        D_prev = D_curr
        k += 1
        
    return C, V
```

**Parameters:**
- $\epsilon$: Convergence threshold for relative distortion change. Typical value not specified, but used to stop iteration when gain is negligible [S1, unofficial].
- $m$: Number of codebook entries, determined by bit-rate $r$ such that $m = 2^r$ [S1, unofficial].

**Pitfalls:**
- **Empty cell problem:** Codebook entries may collapse during refinement, resulting in empty cells that waste codebook entries. The standard LBG does not detect these automatically [S1, unofficial].
- **Performance:** Repeated nearest neighbor searches in step 2 slow down performance significantly [S1, unofficial].
- **Initialization sensitivity:** The algorithm's result depends on the initial codebook. Poor initialization can lead to suboptimal local minima [S1, unofficial].

## Subtopics

### [Modified Vector Quantization (LBG variant)](1130-modified-vector-quantization-lbg-variant.md)
Modified Vector Quantization is a vector quantization algorithm that uses a distortion-based splitting technique to generate the initial codebook, avoiding the time-consuming nearest neighbor searches of the standard LBG algorithm and solving the empty cell problem. It sets itself apart from standard LBG by replacing the random initialization and iterative Lloyd-Max refinement with a deterministic, greedy splitting process driven by residual distortion.

### Agglomerative Information Bottleneck
A technique that extracts a compact representation of variable $X$ (denoted $\hat{X}$) with minimal loss of mutual information with respect to variable $Y$, using hard partitions where each $x$ is assigned to only one cluster [S2, unofficial]. An optimal merge of $l$ components can be obtained by $l - 1$ consecutive optimal merges of pairs of components [S2, unofficial]. The decrease in mutual information due to merging components $x_1, \dots, x_l$ is given by $\delta I_{\hat{x}} = p(\hat{x}) JS(\pi_1, \dots, \pi_l; p_1, \dots, p_l) \geq 0$, where weights $\pi_k = p(x_k)/p(\hat{x})$ and distributions $p_k = p(Y|x_k)$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k \cdot m \cdot |I| \cdot n)$ per iteration, where $k$ is the number of iterations, $m$ is the codebook size, $|I|$ is the number of input vectors, and $n$ is the dimensionality. The nearest neighbor search in step 2 is the bottleneck [S1, unofficial].
- **Memory Complexity:** $O(m \cdot n + |I| \cdot n)$ to store the codebook and input vectors [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance degrades significantly with large $|I|$ due to repeated nearest neighbor searches [S1, unofficial].
- **Strengths:** Simple to implement; guarantees convergence to a local minimum of the distortion [S1, unofficial].
- **Limitations:** Sensitive to initialization; suffers from empty cell problem; slow for large datasets [S1, unofficial].
- **When to use:** Small to medium-sized datasets where simplicity is preferred; when a good initial codebook can be provided.
- **When not to use:** Large datasets where performance is critical; when empty cells are a significant concern.

## Background
The LBG algorithm was introduced by Linde, Buzo, and Gray [S1, unofficial]. It is a modification of the Lloyd-Max algorithm for scalar quantization, extended to vector quantization [S1, unofficial]. It is widely used in image and speech compression, as well as in volume rendering for value compression [S1, unofficial]. The algorithm is general enough to deal with high-dimensional multi-parameter data [S1, unofficial].
## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S2] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
