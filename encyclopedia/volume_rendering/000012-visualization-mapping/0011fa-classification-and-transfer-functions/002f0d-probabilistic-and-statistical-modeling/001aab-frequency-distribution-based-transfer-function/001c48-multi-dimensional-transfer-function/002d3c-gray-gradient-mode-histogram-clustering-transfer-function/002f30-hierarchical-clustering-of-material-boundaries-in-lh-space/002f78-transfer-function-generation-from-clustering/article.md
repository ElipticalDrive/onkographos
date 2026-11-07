# Transfer Function Generation from Clustering

Also called: TF from Labels

## Overview
This method generates a 2D transfer function by assigning color and opacity values to clusters identified in the Local intensity-Histogram peak (LH) space. It solves the problem of mapping volumetric data to visual properties by leveraging the hierarchical structure produced by parent clustering methods, allowing users to select specific hierarchy levels to define material boundaries. It differs from its parent method by formalizing the visual mapping from cluster labels to rendering parameters, and from its sibling method by focusing on the label-based assignment of visual attributes rather than just boundary similarity metrics.

## How it works
**Inputs and Outputs**
*   **Input:** A set of clusters derived from the LH histogram (typically via the parent hierarchical clustering method). Each cluster contains a set of LH coordinates.
*   **Output:** A 2D transfer function defined by a label map. Each unique cluster label is associated with a specific color and opacity value.

**Data Structures**
*   **Label Map:** A 2D grid corresponding to the LH histogram bins. Each cell stores an integer label ID. If a bin is not part of any cluster, it may be assigned a background label (e.g., 0).
*   **Cluster Properties:** A mapping from Label ID to $(Color, Opacity)$.

**Steps**
1.  **Initialize Clusters:** Perform the hierarchical clustering on the LH histogram to obtain a hierarchy of clusters. Select a specific level in the hierarchy to define the final set of disjoint clusters.
2.  **Assign Labels:** Iterate through the LH histogram bins. For each bin $(L, H)$, determine which cluster it belongs to at the selected hierarchy level. Assign a unique integer label $k$ to each cluster. Store this label in the Label Map at position $(L, H)$.
3.  **Initialize Visual Properties:** By default, assign a random color $C_k$ and full opacity $\alpha_k = 1.0$ to each cluster label $k$.
4.  **Handle Hierarchy Merges (Optional/Default Behavior):** If the implementation supports dynamic hierarchy traversal where clusters merge, enforce consistency: when two clusters $A$ and $B$ merge into a parent cluster, the smaller cluster (by number of voxels or histogram bin count) inherits the color and opacity of the larger (dominant) cluster. This ensures visual continuity as the hierarchy is traversed.
5.  **User Interaction:** Allow the user to manually override the color and opacity for any label $k$.
6.  **Rendering Preparation:**
    *   **For 2D-capable renderers:** Use the Label Map directly as the 2D transfer function. The renderer samples the LH value of each voxel, looks up the label, and applies the associated color and opacity.
    *   **For 1D-capable renderers (e.g., VolumePro1000):** Perform a one-time labeling of the volume data. Replace each voxel's intensity with its cluster label. Apply a standard 1D transfer function on these labels. This step is performed once for the initial clusters; subsequent hierarchy changes only require updating the 1D TF mapping, not re-labeling the volume.

**Formulas**
Let $C_k$ be the color and $\alpha_k$ be the opacity for label $k$.
If cluster $A$ merges with cluster $B$, and $|A| < |B|$ (where $|\cdot|$ denotes cluster size):
$$ C_A \leftarrow C_B, \quad \alpha_A \leftarrow \alpha_B $$

**Parameters**
*   **Hierarchy Level:** The depth in the clustering tree where clusters are finalized. Not given in the sources.
*   **Color Assignment:** Random initialization. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Hard Boundaries:** Using labels restricts the method to hard boundaries between clusters. Fuzzy borders between clusters are not possible with this specific label-based implementation [S1].
*   **Hardware Limitations:** On hardware that only supports 1D transfer functions (like VolumePro1000), the volume must be pre-labeled. This limits the interactivity speed if the hierarchy changes frequently, although the sources note that labeling is only needed once for initial clusters, preserving interaction speed for hierarchy traversal [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_tf_from_clustering(clusters, lh_histogram_shape):
    """
    clusters: list of cluster objects, each containing a set of (L, H) coordinates.
    lh_histogram_shape: tuple (num_L, num_H) defining the 2D histogram dimensions.
    Returns:
        label_map: 2D array of integers, shape (num_L, num_H).
        tf_properties: dict mapping label_id -> (color, opacity).
    """
    num_L, num_H = lh_histogram_shape
    label_map = np.zeros(lh_histogram_shape, dtype=int)
    tf_properties = {}
    
    current_label_id = 1
    
    # Step 1 & 2: Assign labels to histogram bins
    for cluster in clusters:
        color = random_color() # [G] Placeholder for random color generation
        opacity = 1.0
        
        for (L, H) in cluster.coordinates:
            label_map[L, H] = current_label_id
            tf_properties[current_label_id] = (color, opacity)
            current_label_id += 1
            
    # Step 3 & 4: Handle merges if hierarchy is dynamic
    # Note: This part is conceptual based on the description of inheritance.
    # In a static final level, this step is skipped.
    # If dynamic:
    # for merge_event in hierarchy_merges:
    #     small_cluster = merge_event.small_cluster
    #     large_cluster = merge_event.large_cluster
    #     small_label = small_cluster.label_id
    #     large_label = large_cluster.label_id
    #     tf_properties[small_label] = tf_properties[large_label]
        
    return label_map, tf_properties
```

## Subtopics

### [Non-Parametric Transfer Function Generation](6695-non-parametric-transfer-function-generation.md)
Non-Parametric Transfer Function Generation is a clustering method for the 2D histogram (conventional or abstracted) using variable kernel density estimation to group data based on neighborhood information. It sets itself apart from fixed-bandwidth density methods by adapting the kernel width to local data density, ensuring accurate estimation in sparse regions without assuming a specific number of clusters.

## Operating characteristics
*   **Time Complexity:** Labeling the volume is a one-time cost $O(N)$ where $N$ is the number of voxels. Subsequent hierarchy interactions are $O(1)$ for 1D TF updates if the volume is pre-labeled [S1].
*   **Memory Complexity:** Requires storage for the label map ($O(N)$) and the TF properties ($O(K)$ where $K$ is the number of clusters).
*   **Strengths:** Simple to implement; allows combination with segmentation [S1]; fast interaction with hierarchy levels for 1D-capable hardware [S1].
*   **Limitations:** Cannot represent fuzzy boundaries between clusters [S1]; requires pre-labeling for 1D hardware, which may limit flexibility in some interactive scenarios.
*   **When to use:** When using hardware with 1D TF limitations or when hard boundaries between materials are desired.
*   **When not to use:** When smooth transitions or fuzzy boundaries between material clusters are required.

## Background
The method was introduced in the paper "Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries" (EuroVis 2006) [S1]. It was demonstrated on CT datasets of an engine and a carp. The approach was designed to work with the VolumePro1000 rendering board, which imposes 1D TF constraints, necessitating the label-based approach [S1].

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
