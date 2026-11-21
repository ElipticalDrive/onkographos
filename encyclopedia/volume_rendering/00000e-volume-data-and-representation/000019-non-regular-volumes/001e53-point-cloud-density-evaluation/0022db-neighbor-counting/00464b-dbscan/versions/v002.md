# DBSCAN

Also called: Density-Based Spatial Clustering of Applications with Noise

## Overview
DBSCAN is a density-based clustering algorithm that identifies arbitrary-shaped clusters and is robust to noise and data heterogeneity [S1, unofficial]. It solves the problem of grouping spatial points into dense regions without requiring a pre-specified number of clusters, distinguishing itself from centroid-based methods like K-Means by its ability to handle non-convex shapes and outliers [S1, unofficial]. It builds on [Neighbor Counting](2905-neighbor-counting.md) by using range-queries to evaluate local point density.

## How it works
**Inputs:** A dataset $P$ of points, a search radius $\varepsilon$ (eps), and a minimum number of points $minPts$.
**Outputs:** A label for each point in $P$: a cluster ID (integer $\ge 0$), or NOISE.

**Data Structures:**
- $P$: List of points.
- $labels$: Array of size $|P|$, initialized to UNCLASSIFIED.
- $S$: A list/queue of points to be expanded.

**Definitions:**
- **Core-point:** A point $p$ is a core-point if the number of points in its $\varepsilon$-neighborhood $V_\varepsilon(p)$ is $\ge minPts$.
- **Border-point:** A point $p_a$ is a border-point if it is density-reachable from a core-point $p_b$ but $|V_\varepsilon(p_a)| < minPts$.
- **Noise:** Points that are neither core-points nor border-points.

**Algorithm:**
1. Initialize $clusterID \leftarrow 0$.
2. For each point $p \in P$, set $labels[p] \leftarrow \text{UNCLASSIFIED}$.
3. For each point $p \in P$:
    a. If $labels[p] \neq \text{UNCLASSIFIED}$, continue.
    b. Call $ExpandCluster(p, \varepsilon, minPts, P, labels, clusterID)$.
    c. If $ExpandCluster$ returns `True` (meaning $p$ was a core-point and a cluster was formed), increment $clusterID \leftarrow clusterID + 1$.

**ExpandCluster(p, $\varepsilon$, minPts, P, labels, clusterID):**
1. Perform a range-query on $P$ with center $p$ and radius $\varepsilon$ to retrieve all neighbors $N_p$.
2. If $|N_p| < minPts$:
    a. Set $labels[p] \leftarrow \text{NOISE}$.
    b. Return `False`.
3. Set $labels[p] \leftarrow clusterID$.
4. Initialize $S$ as a list containing all points in $N_p$ except $p$.
5. While $S$ is not empty:
    a. Remove a point $q$ from $S$.
    b. If $labels[q] == \text{NOISE}$, set $labels[q] \leftarrow clusterID$ [G]. (Note: Standard DBSCAN often leaves NOISE as NOISE unless it becomes reachable from a core point; however, if a point was initially NOISE but is found in a neighbor list of a core point, it becomes a border point. If it was previously labeled NOISE from a prior failed expansion, it should be updated to the current cluster if it is now reachable. The source implies border points are reachable from core points. If $q$ was NOISE, it is now a border point of the current cluster).
    c. If $labels[q] \neq \text{UNCLASSIFIED}$, continue.
    d. Set $labels[q] \leftarrow clusterID$.
    e. Perform a range-query on $P$ with center $q$ and radius $\varepsilon$ to retrieve neighbors $N_q$.
    f. If $|N_q| \ge minPts$:
        i. For each point $r \in N_q$:
            1. If $labels[r] == \text{UNCLASSIFIED}$, add $r$ to $S$.
            2. If $labels[r] == \text{NOISE}$, set $labels[r] \leftarrow clusterID$ [G]. (Border points can be reclassified if they are within the neighborhood of a newly discovered core point).
6. Return `True`.

**Parameters:**
- $\varepsilon$ (eps): The search radius for range-queries. Typical values depend on the scale of the data.
- $minPts$: The minimum number of points required to form a dense region. Typical values are $\ge 4$ for low-dimensional data.

**Edge Cases:**
- If $minPts$ is too high, most points may be labeled NOISE.
- If $\varepsilon$ is too small, clusters may be fragmented.
- Precision issues in distance calculations near the boundary $\varepsilon$ can affect core-point classification.

## Subtopics

### [Density-based Clustering with Weighted Dissimilarity](17596-density-based-clustering-with-weighted-dissimilarity.md)
This method is a density-based clustering algorithm designed to identify similar phases or states across multiple simulation sequences by treating segmented time steps as data points. It distinguishes itself from standard DBSCAN by employing a weighted dissimilarity measure that incorporates temporal indices and sequence identifiers, allowing it to group similar temporal progressions even when they are not strictly synchronous. It builds on [DBSCAN](11722-dbscan.md) but modifies the distance metric and neighborhood density calculation to account for the hierarchical structure of simulation data.

### HDBSCAN
HDBSCAN is a hierarchical density-based clustering algorithm. It constructs a cluster hierarchy using a minimum spanning tree of the mutual reachability graph.
**Steps:**
1. Check if the `hdbscan` package is available; raise `ImportError` if not [S2, unofficial].
2. Construct keyword arguments dictionary with `min_cluster_size`; add `min_samples` if not None; add `cluster_selection_epsilon` if `eps` is not None [S2, unofficial].
3. Instantiate `hdbscan.HDBSCAN` with the constructed kwargs [S2, unofficial].
4. Fit the clusterer to features and predict labels [S2, unofficial].
5. Cast labels to `np.int32` and return [S2, unofficial].
**Parameters:**
- `min_cluster_size`: int, the minimum number of samples in a group for that group to be considered a cluster [S2, unofficial].
- `min_samples`: int | None, optional parameter passed to HDBSCAN [S2, unofficial].
- `eps`: float | None, mapped to `cluster_selection_epsilon` in HDBSCAN [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ with efficient spatial indexing (e.g., KD-Tree, Ball-Tree) for range queries, or $O(N^2)$ with brute-force search [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store the dataset and labels.
- **Strengths:** Identifies arbitrary-shaped clusters; robust to noise; does not require specifying the number of clusters; handles data heterogeneity well [S1, unofficial].
- **Limitations:** Sensitive to parameter selection ($\varepsilon$, $minPts$); struggles with varying densities in the same dataset; performance degrades in very high dimensions without dimensionality reduction [S1, unofficial].
- **When to use:** When clusters are non-convex and of varying densities, and noise is present.
- **When not to use:** When clusters are convex and of similar density (K-Means is better), or when data dimensionality is very high (curse of distance).

## Background
DBSCAN was proposed by Ester et al. in 1996 [S1, unofficial]. It is widely used in spatial data mining, volume exploration, and clustering of point clouds. It is particularly valued for its efficiency and ability to handle noise in volumetric data analysis [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
