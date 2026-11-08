# Modified Vector Quantization (LBG variant)
Also called: Distortion-based splitting LBG, Modified LBG

## Overview
Modified Vector Quantization is a vector quantization algorithm that uses a distortion-based splitting technique to generate the initial codebook, avoiding the time-consuming nearest neighbor searches of the standard LBG algorithm and solving the empty cell problem [S1, unofficial]. It sets itself apart from standard LBG by replacing the random initialization and iterative Lloyd-Max refinement with a deterministic, greedy splitting process driven by residual distortion [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A set of input vectors $I = \{X_1, X_2, \dots, X_N\}$ and a target codebook size $M$ (determined by bit-size, e.g., 8 bits $\rightarrow$ 256 entries) [S1, unofficial].
**Outputs:** A refined codebook $C = \{Y_1, Y_2, \dots, Y_M\}$ [S1, unofficial].

**Data Structures:**
1. **To-Do List:** A double-linked list or priority queue storing tuples $(D_j, Y_j, I_j)$, where $D_j$ is the cell distortion, $Y_j$ is the cell centroid, and $I_j$ is the set of indices of input vectors assigned to the cell. The list is sorted in descending order by $D_j$ [S2, unofficial].
2. **Codebook:** Array of $M$ vectors [S1, unofficial].

**Steps:**
1. **Initialization:** Start with a single entry in the codebook [S1, unofficial]. Set $Y_1$ as the geometric center (centroid) of all input vectors $I$ [S2, unofficial]. Compute initial distortion $D_1 = \sum_{X \in I} \delta(X, Y_1)$, where $\delta$ is the distortion metric [S2, unofficial]. Insert the group $(D_1, Y_1, I_1 = \{i \in I : X_i \in V_1\})$ into the To-Do list [S2, unofficial]. Current codebook size $k=1$.
2. **Distortion-Based Splitting Loop:** While $k < M$: Identify the entry $j$ with the largest distortion $D_j$ in the To-Do list [S1, unofficial]. Split this entry into two new entries such that the distortion is roughly equalized between them [S1, unofficial]. Remove the old entry $j$ from the To-Do list [S2, unofficial]. Compute the new centroids $Y_{new1}, Y_{new2}$ and distortions $D_{new1}, D_{new2}$ for the two sub-cells [S2, unofficial]. Insert the two new groups $(D_{new1}, Y_{new1}, I_{new1})$ and $(D_{new2}, Y_{new2}, I_{new2})$ into the To-Do list [S2, unofficial]. Increment $k$ by 1, as only one new entry is generated per split step [S1, unofficial].
3. **Refinement:** Once $M$ entries are reached, the generated initial codebook is refined using the standard LBG algorithm (Lloyd-Max iterations) to produce the final codebook [S1, unofficial]. This step corrects any sub-optimality introduced by the greedy splitting [G].

**Parameters:**
* Target codebook size $M$: Determined by the desired bit-depth [S1, unofficial].
* Distortion metric $\delta$: Typically squared Euclidean distance $\|X - Y\|^2$ [G].

**Edge Cases and Pitfalls:**
* **Empty Cells:** The distortion-based splitting avoids empty cells, which are common in standard LBG and significantly increase compression distortion [S1, unofficial].
* **Precision:** Computing exact distortion sums for large datasets can be computationally expensive; approximations may be used during the splitting phase [S2, unofficial].
* **Splitting Heuristic:** The "roughly equalized distortion" split is a heuristic [G]. If the split is not optimal, the final LBG refinement is critical [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_lbg(vectors, target_size):
    # Step 1: Initialization
    dim = len(vectors[0])
    centroid = [sum(v[i] for v in vectors) / len(vectors) for i in range(dim)]
    distortion = sum(sum((v[i] - centroid[i])**2 for i in range(dim)) for v in vectors)
    todo = [(distortion, centroid, list(range(len(vectors))))]
    todo.sort(key=lambda x: x[0], reverse=True)
    k = 1

    # Step 2: Distortion-Based Splitting Loop
    while k < target_size:
        d_j, y_j, idx_j = todo.pop(0)
        idx1, idx2 = split_indices(vectors, idx_j, y_j)
        y1 = [sum(vectors[i][d] for i in idx1) / len(idx1) for d in range(dim)]
        y2 = [sum(vectors[i][d] for i in idx2) / len(idx2) for d in range(dim)]
        d1 = sum(sum((vectors[i][d] - y1[d])**2 for d in range(dim)) for i in idx1)
        d2 = sum(sum((vectors[i][d] - y2[d])**2 for d in range(dim)) for i in idx2)
        todo.append((d1, y1, idx1))
        todo.append((d2, y2, idx2))
        todo.sort(key=lambda x: x[0], reverse=True)
        k += 1

    # Step 3: Refinement
    initial_codebook = [y for _, y, _ in todo]
    final_codebook = lbg_refinement(initial_codebook, vectors)
    return final_codebook

def split_indices(vectors, indices, centroid):
    # Heuristic split to roughly equalize distortion [G]
    dists = [(sum((vectors[i][d] - centroid[d])**2 for d in range(len(centroid))), i) for i in indices]
    dists.sort()
    mid = len(dists) // 2
    return [i for _, i in dists[:mid]], [i for _, i in dists[mid:]]

def lbg_refinement(codebook, vectors):
    # Standard Lloyd-Max iterations [G]
    dim = len(codebook[0])
    for _ in range(10):
        assignments = [[] for _ in codebook]
        for v in vectors:
            best = min(range(len(codebook)), key=lambda c: sum((v[d] - codebook[c][d])**2 for d in range(dim)))
            assignments[best].append(v)
        new_codebook = []
        for group in assignments:
            if group:
                new_codebook.append([sum(v[d] for v in group) / len(group) for d in range(dim)])
            else:
                new_codebook.append(codebook[len(new_codebook)])
        codebook = new_codebook
    return codebook
```

## Subtopics

### PCA-Based Splitting Initialization
An improved splitting technique to generate the initial codebook for LBG, using Principal Component Analysis (PCA) to find optimal splitting planes based on variances of subregions, avoiding empty cells and improving convergence speed [S2, unofficial]. It sets itself apart from the basic distortion-based split by using the covariance matrix of the data points in the cell to determine the splitting direction, ensuring the split aligns with the data's principal axes [S2, unofficial]. Choose this when the data has strong directional variance and high-dimensional input vectors [G]. It is linked to [Modified Vector Quantization (LBG variant)](841-modified-vector-quantization.md) [G].

## Operating characteristics
**Time Complexity:** The splitting phase avoids the $O(N \cdot M)$ nearest neighbor searches of standard LBG initialization, offering a considerable speed-up [S2, unofficial]. However, sorting the To-Do list and computing distortions adds overhead [S2, unofficial]. The final LBG refinement still requires nearest neighbor searches [S1, unofficial].
**Memory Complexity:** Requires storing the To-Do list, which can grow to $O(M)$ entries, each containing a set of indices [S2, unofficial]. This can be memory-intensive for large $N$ [G].
**Measured Performance:** Faster than standard LBG initialization, especially for large codebooks [S2, unofficial]. Solves the empty cell problem, leading to better compression quality [S1, unofficial].
**Strengths:** Avoids empty cells, faster initialization, deterministic [S1, unofficial].
**Limitations:** Splitting heuristic may not be globally optimal; final refinement is still required [S1, unofficial].
**When to use:** When initialization speed is critical and empty cells are a concern [G].
**When not to use:** When memory is extremely constrained or when the data distribution is uniform (standard LBG may suffice) [G].

## Background
Originated from the need to improve the initialization of the LBG algorithm for volume data compression [S1, unofficial]. The splitting technique was originally proposed by Pauly et al. [41] for hierarchical clustering problems and adapted for vector quantization [S1, unofficial]. A more detailed description was provided by Schneider [25] [S1, unofficial]. The PCA-based variant was described in applications ranging from data clustering to load balancing, and used by Pauly et al. [2002] and Lensch et al. [2001] for hierarchical clustering of point sets and scanned BRDFs [S2, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
