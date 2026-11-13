# Two-stage k-means Streamline Clustering

Also called: Two-stage k-means Clustering

## Overview
Two-stage k-means streamline clustering is a method for abstracting 3D vector fields by decomposing a large set of streamlines into coherent bundles and extracting representative streamlines for visualization [S1]. It solves the problem of visual clutter and high computational complexity associated with rendering raw streamline data by reducing the dataset to a concise set of representative paths [S1]. Unlike standard k-means which operates on high-dimensional point vectors, this method uses low-dimensional feature vectors (spatial coordinates and entropy metrics) to ensure efficiency and coherence in flow structure representation [S1].

## How it works
**Inputs:** A set of generated streamlines from a 3D vector field. Each streamline is defined by a sequence of points.
**Outputs:** A set of representative streamlines, grouped into sub-bundles, suitable for illustrative rendering (e.g., as streamtapes) [S1].

**Data Structures:**
Each streamline is represented by a 1D feature vector. The method distinguishes between two types of feature vectors used in different stages:
1.  **Spatial Property Vector:** A 9-component vector (or 3 points) comprising the start point, middle point, and end point of the streamline. Let $\mathbf{s}_i$ be the spatial vector for streamline $i$.
2.  **Shape Property Vector:** A 2-component vector comprising Linear Entropy ($E_L$) and Angular Entropy ($E_A$). Let $\mathbf{sh}_i$ be the shape vector for streamline $i$.

**Feature Formulas:**
The shape properties are calculated based on the streamline's geometry and segment lengths [S1].

Linear Entropy ($E_L$) quantifies the information in flow magnitude along the streamline:
$$ E_L = - \frac{1}{\log_2(m + 1)} \sum_{j=0}^{D} \left( \frac{D_j}{L_S} \right) \log_2 \left( \frac{D_j}{L_S} \right) $$
Where:
*   $m$ is the number of streamline segments.
*   $D_j$ is the length of the $j$-th segment.
*   $L_S$ is the total length of the streamline ($\sum D_j$).
*   Note: The summation index in the source text [S1] is written as $j=0$ to $D$, but the context implies summing over all segments. Given $m$ segments, the sum typically runs over the $m$ segments. The source formula uses $D$ in the limit but $m$ in the denominator log. We adhere to the source's notation: sum over segments $j$, where $D_j$ is segment length.

Angular Entropy ($E_A$) quantifies the angular variation:
$$ E_A = - \frac{1}{\log_2(m)} \sum_{j=0}^{m-1} \left( \frac{A_j}{L_A} \right) \log_2 \left( \frac{A_j}{L_A} \right) $$
Where:
*   $m$ is the number of streamline segments.
*   $A_j$ is the absolute value of the angle at the $j$-th streamline joint.
*   $L_A$ is the total angular variation along the streamline ($\sum A_j$).

**Algorithm Steps:**

1.  **Feature Extraction:** For each streamline in the input set, compute the Spatial Property Vector (start, middle, end points) and the Shape Property Vector ($E_L, E_A$) [S1].

2.  **Stage 1 Clustering (Spatial Partitioning):**
    *   Input: All streamlines, Spatial Property Vectors.
    *   Parameter: $p$, the number of initial bundles.
    *   Action: Perform k-means clustering with Lloyd’s algorithm using Euclidean distance on the Spatial Property Vectors.
    *   Output: $p$ bundles, where each bundle contains streamlines with roughly coherent spatial structure [S1].

3.  **Stage 2 Clustering (Shape Refinement):**
    *   For each bundle $i$ (where $i = 1 \dots p$) obtained from Stage 1:
        *   Input: Streamlines in bundle $i$, Shape Property Vectors.
        *   Parameter: $q_i$, the number of sub-bundles for bundle $i$.
        *   Action: Perform k-means clustering with Lloyd’s algorithm using Euclidean distance on the Shape Property Vectors.
        *   Output: $q_i$ sub-bundles within bundle $i$.
    *   Total clusters: $c = \sum_{i=1}^{p} q_i$ [S1].

4.  **Selection of Representative Streamlines:**
    *   For each sub-bundle:
        *   Calculate the centroid of the sub-bundle in the shape feature space (or spatial space, the source implies sorting by distance to centroid, typically in the feature space used for clustering or the original geometry; the source says "sort all streamlines within each sub-bundle based on their distances to the centroid").
        *   Sort streamlines by their distance to the centroid.
        *   Select one or multiple streamlines closest to the centroid as the representative streamlines for that sub-bundle [S1].

5.  **Optional Parameter Optimization:**
    *   Determine optimal $p$ and $q_i$ using the elbow criterion [KS96].
    *   The elbow criterion finds the number of clusters with an abrupt change in the percentage of variance explained.
    *   Percentage of variance explained = (Within-cluster variance) / (Total variance) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_kmeans_clustering(streamlines, p, q_list=None):
    """
    streamlines: List of Streamline objects.
    p: Number of initial spatial bundles.
    q_list: List of integers specifying sub-bundle counts for each of the p bundles.
            If None, optimal q_i must be determined via elbow criterion (not implemented here).
    Returns: List of representative streamlines.
    """
    
    # Step 1: Extract Features
    spatial_features = []
    shape_features = []
    
    for sl in streamlines:
        # Spatial: Start, Middle, End points (3 points * 3 coords = 9 dims)
        # Assuming sl.start, sl.middle, sl.end are (x,y,z) tuples
        s_vec = sl.start + sl.middle + sl.end 
        spatial_features.append(s_vec)
        
        # Shape: Linear Entropy (EL) and Angular Entropy (EA)
        el = compute_linear_entropy(sl)
        ea = compute_angular_entropy(sl)
        shape_features.append((el, ea))
        
    spatial_features = np.array(spatial_features)
    shape_features = np.array(shape_features)
    
    # Step 2: Stage 1 - Spatial Clustering
    # Cluster spatial features into p bundles
    stage1_labels = kmeans(spatial_features, n_clusters=p)
    
    bundles = [[] for _ in range(p)]
    for idx, label in enumerate(stage1_labels):
        bundles[label].append(idx)
        
    representatives = []
    
    # Step 3: Stage 2 - Shape Clustering
    for i in range(p):
        bundle_indices = bundles[i]
        bundle_shape_features = shape_features[bundle_indices]
        
        # Determine q_i
        if q_list is not None:
            q_i = q_list[i]
        else:
            # Not given in sources: Implementation of elbow criterion for q_i
            # Placeholder for optimization logic
            q_i = 2 
            
        # Cluster shape features within this bundle
        # Note: kmeans usually takes all data. Here we cluster only the subset.
        # We need to map local indices back to global indices.
        local_labels = kmeans(bundle_shape_features, n_clusters=q_i)
        
        sub_bundles = [[] for _ in range(q_i)]
        for local_idx, label in enumerate(local_labels):
            global_idx = bundle_indices[local_idx]
            sub_bundles[label].append(global_idx)
            
        # Step 4: Select Representatives
        for sub_bundle_indices in sub_bundles:
            if not sub_bundle_indices:
                continue
                
            # Calculate centroid of the sub-bundle in shape space
            sub_shape = shape_features[sub_bundle_indices]
            centroid = np.mean(sub_shape, axis=0)
            
            # Calculate distances to centroid
            distances = np.linalg.norm(shape_features[sub_bundle_indices] - centroid, axis=1)
            
            # Sort by distance (ascending)
            sorted_order = np.argsort(distances)
            
            # Select closest (e.g., the first one)
            best_local_idx = sorted_order[0]
            best_global_idx = sub_bundle_indices[best_local_idx]
            
            representatives.append(streamlines[best_global_idx])
            
    return representatives

def compute_linear_entropy(streamline):
    """
    EL = - (1 / log2(m + 1)) * sum( (Dj / LS) * log2(Dj / LS) )
    """
    segments = streamline.segments # List of segment lengths
    m = len(segments)
    if m == 0:
        return 0.0
    LS = sum(segments)
    if LS == 0:
        return 0.0
        
    entropy = 0.0
    for Dj in segments:
        ratio = Dj / LS
        if ratio > 0:
            entropy += ratio * np.log2(ratio)
            
    return - (1.0 / np.log2(m + 1)) * entropy

def compute_angular_entropy(streamline):
    """
    EA = - (1 / log2(m)) * sum( (Aj / LA) * log2(Aj / LA) )
    """
    angles = streamline.joint_angles # List of absolute angles at joints
    m = len(angles)
    if m == 0:
        return 0.0
    LA = sum(angles)
    if LA == 0:
        return 0.0
        
    entropy = 0.0
    for Aj in angles:
        ratio = Aj / LA
        if ratio > 0:
            entropy += ratio * np.log2(ratio)
            
    return - (1.0 / np.log2(m)) * entropy
```

**Edge Cases and Pitfalls:**
*   **Complexity:** Standard k-means on line data has high complexity because lines are represented as vectors of original points. This method avoids that by using 5-component feature vectors (3 spatial points + 2 shape metrics) [S1].
*   **Degenerate Streamlines:** If a streamline has zero length or zero angular variation, the logarithmic terms in entropy calculations may be undefined. Guard clauses (e.g., returning 0.0) are required [G].
*   **Parameter Selection:** The choice of $p$ and $q_i$ significantly affects the abstraction level. The elbow criterion is suggested but requires computing variance ratios for multiple cluster counts [S1].

## Operating characteristics
*   **Time Complexity:** The complexity is determined by the number of streamlines $N$ and the number of clusters. Since each streamline is reduced to a fixed-size feature vector (9 dims for Stage 1, 2 dims for Stage 2), the complexity is $O(N \cdot K \cdot I)$ where $K$ is the number of clusters and $I$ is the number of iterations, rather than $O(N \cdot L \cdot K \cdot I)$ where $L$ is the number of points per streamline. This makes it efficient for large datasets [S1].
*   **Memory Complexity:** Low, as it only stores the feature vectors and cluster assignments, not the full point clouds for all streamlines during clustering [S1].
*   **Strengths:** Efficiently decomposes large streamline sets into coherent bundles; reduces visual clutter; facilitates illustrative rendering (e.g., streamtapes) [S1].
*   **Limitations:** Relies on the quality of the feature vectors; spatial clustering may group streamlines with different flow shapes if they share start/end points; shape clustering ignores spatial position [S1].
*   **When to use:** When visualizing complex 3D vector fields with many streamlines and needing a concise, illustrative representation [S1].
*   **When not to use:** When precise geometric fidelity of every streamline is required, or when the flow field has no coherent spatial structure [S1].

## Background
The method was introduced by Chen, Yan, Yu, Max, and Ma in 2011 in the paper "An Illustrative Visualization Framework for 3D Vector Fields" [S1]. It was developed to address the limitations of existing vector field visualization techniques, which often assume no particular structure or require topological connectivity. The method builds on the concept of using entropy metrics (Linear and Angular Entropy) originally proposed by Furuya and Itoh [FI08] and Marchesi et al. [S1]. It is used in the context of illustrative visualization, specifically rendering streamlines as "streamtapes" to enhance depth perception and depict vicinity flow [S1].

## Sources
[S1] An Illustrative Visualization Framework for 3D Vector Fields, http://vis.cs.ucdavis.edu/papers/pg2011paper.pdf, fetched 2026-10-29
