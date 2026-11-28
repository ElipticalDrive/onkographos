# Two-stage k-means Streamline Clustering

Also called: Two-stage k-means Clustering

## Overview
Two-stage k-means streamline clustering abstracts 3D vector fields by decomposing large streamline sets into coherent bundles and extracting representative paths for visualization [S1]. It solves visual clutter and high computational complexity by reducing raw streamline data to concise representative paths using low-dimensional feature vectors instead of full point clouds [S1].

## How it works
**Inputs:** A set of generated streamlines from a 3D vector field [S1].
**Outputs:** A set of representative streamlines grouped into sub-bundles, ready for illustrative rendering (e.g., as streamtapes) [S1].

**Data Structures:**
Each streamline is mapped to a 1D feature vector [S1]. The method uses two distinct feature layouts:
1. **Spatial Property Vector:** A 9-component vector formed by concatenating the start, middle, and end points of the streamline [S1].
2. **Shape Property Vector:** A 2-component vector containing Linear Entropy ($E_L$) and Angular Entropy ($E_A$) [S1].

**Feature Formulas:**
Shape properties quantify flow magnitude distribution and directional variation along the streamline [S1].

Linear Entropy ($E_L$) measures information in flow magnitude:
$$ E_L = - \frac{1}{\log_2(m + 1)} \sum_{j=1}^{m} \left( \frac{D_j}{L_S} \right) \log_2 \left( \frac{D_j}{L_S} \right) $$
Where:
* $m$ is the number of streamline segments [S1].
* $D_j$ is the length of the $j$-th segment [S1].
* $L_S = \sum_{j=1}^{m} D_j$ is the total streamline length [S1].

Angular Entropy ($E_A$) measures directional variation:
$$ E_A = - \frac{1}{\log_2(m)} \sum_{j=1}^{m-1} \left( \frac{A_j}{L_A} \right) \log_2 \left( \frac{A_j}{L_A} \right) $$
Where:
* $m$ is the number of streamline segments [S1].
* $A_j$ is the absolute angle at the $j$-th joint [S1].
* $L_A = \sum_{j=1}^{m-1} A_j$ is the total angular variation [S1].

**Algorithm Steps:**
1. **Feature Extraction:** Compute the 9D Spatial Property Vector and 2D Shape Property Vector for every input streamline [S1].
2. **Stage 1 Clustering:** Partition all streamlines into $p$ bundles using k-means with Lloyd’s algorithm on the Spatial Property Vectors [S1]. Each bundle contains streamlines with roughly coherent spatial layout [S1].
3. **Stage 2 Clustering:** For each bundle $i$ ($i = 1 \dots p$), partition it into $q_i$ sub-bundles using k-means on the Shape Property Vectors [S1]. The total cluster count is $c = \sum_{i=1}^{p} q_i$ [S1].
4. **Representative Selection:** Within each sub-bundle, compute the feature-space centroid, sort streamlines by Euclidean distance to the centroid, and select the closest streamline(s) as representatives [S1].
5. **Parameter Optimization (Optional):** Determine optimal $p$ and $q_i$ values using the elbow criterion, which identifies the cluster count where the percentage of variance explained (ratio of within-cluster variance to total variance) shows an abrupt change [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b): return sum(x * y for x, y in zip(a, b))
def norm(a): return math.sqrt(dot(a, a))
def dist(a, b): return norm([x - y for x, y in zip(a, b)])
def mean_vec(vecs): return [sum(v[i] for v in vecs) / len(vecs) for i in range(len(vecs[0]))]

def kmeans_lloyd(data, k, max_iter=100):
    if k > len(data): k = len(data)
    centroids = data[:k]
    labels = [0] * len(data)
    for _ in range(max_iter):
        new_labels = []
        for d in data:
            best_c = min(range(k), key=lambda c: dist(d, centroids[c]))
            new_labels.append(best_c)
        if new_labels == labels: break
        labels = new_labels
        for c in range(k):
            cluster_points = [data[i] for i in range(len(data)) if labels[i] == c]
            if cluster_points:
                centroids[c] = mean_vec(cluster_points)
    return labels

def compute_linear_entropy(segments):
    m = len(segments)
    if m == 0: return 0.0
    LS = sum(segments)
    if LS == 0: return 0.0
    entropy = sum((d / LS) * math.log2(d / LS) for d in segments if d > 0)
    return - (1.0 / math.log2(m + 1)) * entropy

def compute_angular_entropy(angles):
    m = len(angles)
    if m == 0: return 0.0
    LA = sum(angles)
    if LA == 0: return 0.0
    entropy = sum((a / LA) * math.log2(a / LA) for a in angles if a > 0)
    return - (1.0 / math.log2(m)) * entropy

def two_stage_kmeans_clustering(streamlines, p, q_list=None):
    spatial_feats = []
    shape_feats = []
    for sl in streamlines:
        spatial_feats.append(sl.start + sl.middle + sl.end)
        shape_feats.append((compute_linear_entropy(sl.segments),
                            compute_angular_entropy(sl.joint_angles)))

    stage1_labels = kmeans_lloyd(spatial_feats, p)
    bundles = [[] for _ in range(p)]
    for idx, label in enumerate(stage1_labels):
        bundles[label].append(idx)

    representatives = []
    for i in range(p):
        bundle_indices = bundles[i]
        bundle_shape = [shape_feats[idx] for idx in bundle_indices]
        q_i = q_list[i] if q_list else 2
        sub_labels = kmeans_lloyd(bundle_shape, q_i)

        for c in range(q_i):
            sub_indices = [bundle_indices[j] for j in range(len(bundle_indices)) if sub_labels[j] == c]
            if not sub_indices: continue
            centroid = mean_vec([shape_feats[idx] for idx in sub_indices])
            sub_indices.sort(key=lambda idx: dist(shape_feats[idx], centroid))
            representatives.append(streamlines[sub_indices[0]])

    return representatives
```
*Note: The sources do not provide the implementation details for the elbow criterion optimization [G].*

**Edge Cases and Pitfalls:**
* **Complexity:** Standard k-means on line data has high complexity because lines are represented as vectors of original points [S1]. This method avoids that by using 5-component feature vectors (3 spatial points + 2 shape metrics) [S1].
* **Degenerate Streamlines:** If a streamline has zero length or zero angular variation, the logarithmic terms in entropy calculations may be undefined [G]. Guard clauses (e.g., returning 0.0) are required [G].
* **Parameter Selection:** The choice of $p$ and $q_i$ significantly affects the abstraction level [S1]. The elbow criterion is suggested but requires computing variance ratios for multiple cluster counts [S1].

## Operating characteristics
* **Time Complexity:** The complexity is determined by the number of streamlines $N$ and the number of clusters [S1]. Since each streamline is reduced to a fixed-size feature vector (9 dimensions for Stage 1, 2 dimensions for Stage 2), the complexity is $O(N \cdot K \cdot I)$ where $K$ is the number of clusters and $I$ is the number of iterations, rather than $O(N \cdot L \cdot K \cdot I)$ where $L$ is the number of points per streamline [S1].
* **Memory Complexity:** Low, as it only stores the feature vectors and cluster assignments, not the full point clouds for all streamlines during clustering [S1].
* **Strengths:** Efficiently decomposes large streamline sets into coherent bundles; reduces visual clutter; facilitates illustrative rendering (e.g., streamtapes) [S1].
* **Limitations:** Relies on the quality of the feature vectors; spatial clustering may group streamlines with different flow shapes if they share start/end points; shape clustering ignores spatial position [S1].
* **When to use:** When visualizing complex 3D vector fields with many streamlines and needing a concise, illustrative representation [S1].

## Background
The method was introduced by Chen, Yan, Yu, Max, and Ma in 2011 in the paper "An Illustrative Visualization Framework for 3D Vector Fields" [S1]. It was developed to address the limitations of existing vector field visualization techniques, which often assume no particular structure or require topological connectivity [S1]. The method builds on the concept of using entropy metrics (Linear and Angular Entropy) originally proposed by Furuya and Itoh [S1]. It is used in the context of illustrative visualization, specifically rendering streamlines as "streamtapes" to enhance depth perception and depict vicinity flow [S1].

## Sources
[S1] An Illustrative Visualization Framework for 3D Vector Fields, http://vis.cs.ucdavis.edu/papers/pg2011paper.pdf, fetched 2026-10-29
