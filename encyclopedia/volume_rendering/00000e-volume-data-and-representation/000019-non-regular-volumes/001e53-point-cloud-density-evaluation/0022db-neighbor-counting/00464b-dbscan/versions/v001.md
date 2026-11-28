# DBSCAN
Also called: Density-Based Spatial Clustering of Applications with Noise

## Overview
DBSCAN is a density-based clustering algorithm that identifies arbitrary-shaped clusters in point cloud data while robustly handling noise and data heterogeneity [S1, unofficial]. It solves the problem of grouping spatially dense regions without requiring a pre-specified number of clusters, distinguishing itself from centroid-based methods like K-Means by its ability to detect non-convex shapes and its explicit classification of outliers as noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of points $P$, a search radius $\epsilon$ (eps), and a minimum point count $minPts$.
- Output: A label for each point in $P$ (cluster ID, NOISE, or UNCLASSIFIED during processing).

**Data Structures:**
- $P$: List of points.
- $labels$: Array mapping each point index to a cluster ID or noise label.
- $S$: A stack or queue of points to be expanded.

**Definitions:**
- **Core-point**: A point $p$ is a core-point if the number of points in its $\epsilon$-neighborhood $V_\epsilon(p)$ is greater than or equal to $minPts$ [S1, unofficial].
- **Border-point**: A point $p_a$ is a border-point if it is directly density-reachable from a core-point $p_b$ but $|V_\epsilon(p_a)| < minPts$ [S1, unofficial].
- **Noise**: Points that are neither core-points nor border-points [S1, unofficial].

**Algorithm Steps:**
1. Initialize $clusterID = 0$.
2. Label all points in $P$ as UNCLASSIFIED.
3. Iterate through each point $p$ in $P$:
   a. If $p$ is UNCLASSIFIED:
      i. Call $ExpandCluster(p, \epsilon, minPts, P, labels, clusterID)$.
      ii. If $ExpandCluster$ successfully forms a cluster (i.e., $p$ is a core-point), increment $clusterID$.
4. Return $labels$.

**ExpandCluster Routine:**
1. Perform a range query on $P$ centered at $p$ with radius $\epsilon$ to retrieve neighbors $N_p$.
2. If $|N_p| < minPts$:
   a. Label $p$ as NOISE.
   b. Return False.
3. Label $p$ as the current $clusterID$.
4. Initialize $S$ with all points in $N_p$ except $p$.
5. While $S$ is not empty:
   a. Pop a point $q$ from $S$.
   b. If $q$ is labeled NOISE, label it as border-point of the current cluster (or keep as NOISE depending on implementation; standard DBSCAN often re-labels noise points encountered during expansion as border-points if they are within $\epsilon$ of a core-point, but strictly, noise points are those never reached by a core-point. The source notes state: "Assign all reached points to the current cluster" [S1, unofficial]).
   c. If $q$ is already labeled (as a different cluster or the current one), continue.
   d. If $q$ is UNCLASSIFIED:
      i. Label $q$ as the current $clusterID$.
      ii. Perform a range query on $q$ to get $N_q$.
      iii. If $|N_q| \ge minPts$:
           1. $q$ is a core-point. Add all points in $N_q$ to $S$ (if not already processed/visited).
   e. If $q$ is UNCLASSIFIED and $|N_q| < minPts$, it remains a border-point (or is labeled as such) and is not added to $S$ for further expansion.

**Parameters:**
- $\epsilon$ (eps): The search radius for range-queries [S1, unofficial].
- $minPts$: The minimum number of points required to form a dense region (core point) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: Distance calculations must be consistent. In high-dimensional spaces, the "curse of dimensionality" may render $\epsilon$ ineffective [S1, unofficial].
- **Boundaries**: Points near the boundary of the dataset may have fewer neighbors, potentially being misclassified as noise or border-points if $\epsilon$ is too small relative to local density variations.
- **Degenerate Input**: If $minPts$ is set too high, no core-points may exist, resulting in all points being labeled NOISE. If $\epsilon$ is too large, all points may merge into a single cluster.

## Subtopics

### HDBSCAN
HDBSCAN (Hierarchical DBSCAN) is a variant that extends DBSCAN by building a hierarchy of clusters and selecting the most stable ones based on cluster size and density [S2, unofficial]. It differs from standard DBSCAN by automatically determining the optimal number of clusters and handling varying densities more effectively. It is chosen when the dataset has clusters of varying densities or when the optimal $\epsilon$ and $minPts$ are unknown. Parameters include $min\_cluster\_size$ (minimum samples for a cluster), $min\_samples$ (optional, passed to HDBSCAN), and $eps$ (mapped to $cluster\_selection\_epsilon$) [S2, unofficial]. Implementation involves constructing kwargs with these parameters, instantiating `hdbscan.HDBSCAN`, fitting to features, and predicting labels cast to `np.int32` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \log N)$ with efficient spatial indexing (e.g., KD-Tree, Ball-Tree), otherwise $O(N^2)$ for brute-force range queries [S1, unofficial].
- **Memory Complexity**: $O(N)$ to store points and labels [S1, unofficial].
- **Strengths**: Identifies arbitrary-shaped clusters; robust to noise; no need to specify number of clusters [S1, unofficial].
- **Limitations**: Sensitive to parameter selection ($\epsilon$, $minPts$); struggles with varying densities (addressed by HDBSCAN); performance degrades in high dimensions [S1, unofficial].
- **When to use**: When clusters are dense, separated, and of arbitrary shape; when noise is present.
- **When not to use**: When clusters have significantly different densities (use HDBSCAN); in very high-dimensional spaces without dimensionality reduction.

## Background
DBSCAN was proposed by Ester et al. in 1996 [S1, unofficial]. It is widely used in spatial data mining, volume data exploration, and point cloud processing due to its efficiency and robustness [S1, unofficial]. It is particularly valued in volumetric data analysis for its ability to handle heterogeneous artifacts and its computational efficiency compared to other clustering techniques [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
