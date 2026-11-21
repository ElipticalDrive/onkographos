# Transfer Function Generation from Clustering

Also called: TF from Labels

## Overview
This method generates a transfer function (TF) by clustering material boundaries in the Local intensity-Histogram peak (LH) space and assigning visual properties to the resulting clusters [S1]. It solves the problem of manual TF design by automatically grouping similar voxel regions based on hierarchical clustering, allowing users to interact with cluster hierarchies to define color and opacity mappings [S1]. It differs from its parent method by explicitly defining the final mapping as a label-based assignment where clusters inherit visual properties upon merging, and from its sibling by focusing on the generation of the TF from the clustering result rather than just measuring similarity [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical clustering structure of material boundaries defined in the LH space, derived from the parent method [S1].
*   **Output:** A 2D transfer function mapping LH coordinates to color and opacity, or a labeled volume for 1D rendering [S1].

**Data Structures:**
*   **Cluster Hierarchy:** A tree structure where nodes represent clusters. Each node contains a set of LH coordinates (material boundaries) and a size metric (number of boundaries or voxels) [S1].
*   **Label Map:** A 3D array matching the volume dimensions, storing an integer label for each voxel [S1].
*   **Visual Properties:** A map associating each cluster ID with a color (RGB) and opacity (alpha) [S1].

**Steps:**
1.  **Initialize Clusters:** Assign a unique label to each initial cluster identified in the LH histogram [S1].
2.  **Assign Default Properties:** By default, assign each cluster a random color and full opacity (alpha = 1.0) [S1].
3.  **Process Hierarchy:** Traverse the clustering hierarchy from leaves to root (or as defined by the user's interaction level) [S1].
4.  **Merge Clusters:** When two clusters join (merge into a parent node):
    *   Identify the dominant cluster (the one with the larger size) [S1].
    *   Assign the smaller cluster the color and opacity of the dominant cluster [S1].
    *   Update the label map for all voxels belonging to the smaller cluster to match the dominant cluster's label [S1].
5.  **Finalize TF:** The resulting label map defines the 2D TF, where each unique label maps to a specific color and opacity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_tf_from_clustering(hierarchy, volume_labels, cluster_properties):
    """
    hierarchy: List of merge events, each containing (smaller_cluster_id, larger_cluster_id)
    volume_labels: 3D array of initial cluster labels
    cluster_properties: Dict mapping cluster_id -> {'color': (r,g,b), 'opacity': float}
    
    Returns:
    final_labels: 3D array of final cluster labels
    final_properties: Dict mapping final_cluster_id -> {'color': (r,g,b), 'opacity': float}
    """
    # Copy initial properties to avoid mutation issues
    current_properties = {cid: props.copy() for cid, props in cluster_properties.items()}
    
    # Process merges in order (assuming hierarchy is ordered by merge time/level)
    for smaller_id, larger_id in hierarchy:
        # Inherit properties from the dominant (larger) cluster
        inherited_color = current_properties[larger_id]['color']
        inherited_opacity = current_properties[larger_id]['opacity']
        
        # Update the smaller cluster's properties to match the larger one
        current_properties[smaller_id] = {
            'color': inherited_color,
            'opacity': inherited_opacity
        }
        
        # Update volume labels: voxels with smaller_id now have larger_id
        # Note: In a real implementation, this might be deferred or optimized
        # For the purpose of TF generation, we just need the final mapping
        pass 

    # The final TF is defined by current_properties for each unique label remaining
    # If rendering on hardware requiring 1D TF (e.g., VolumePro1000), 
    # the volume_labels must be updated to reflect the merged labels.
    return current_properties
```

**Parameters:**
*   **Dominance Criterion:** The cluster with the larger size (number of boundaries or voxels) is considered dominant [S1].
*   **Default Opacity:** Full opacity (1.0) [S1].
*   **Default Color:** Random [S1].

**Edge Cases and Pitfalls:**
*   **Hard Boundaries:** Using labels restricts the method to hard boundaries; fuzzy borders between clusters are not possible with this specific implementation [S1].
*   **Hardware Limitations:** If the rendering hardware only supports 1D TFs (e.g., VolumePro1000), the volume must be labeled once for the initial clusters, and a 1D TF is performed on the labels [S1].
*   **User Interaction:** There may be no single hierarchy level where clusters are grouped according to user expectations; constraints can be introduced into the hierarchy to handle this [S1].

## Subtopics

### [Non-Parametric Transfer Function Generation](6695-non-parametric-transfer-function-generation.md)
Non-Parametric Transfer Function Generation is a clustering method for 2D histograms (conventional or abstracted) that groups data based on neighborhood information using variable kernel density estimation. It provides users with a means to group data in the 2D histogram by areas of similar value.

### Multidimensional Transfer Function for Wing Dataset
Designed to capture details of green structures and inside vortices/spirals, which 1D scalar transfer functions fail to capture [S2, unofficial]. Color map control points: (35000, pink), (50000, cyan), (60000, green) [S2, unofficial]. Scalar opacity transfer function control points: (35000, 0.0), (50000, 0.1), (60000, 0.50) [S2, unofficial]. Gradient magnitude transfer function control points: (140, 0.0), (150, 1.0), (180, 0.0), (250, 0.0), (260, 1.0), (300, 0.0) [S2, unofficial]. Designing these functions is difficult because gradient values at different scalar values might be equal for different regions (e.g., different vortices) [S2, unofficial]. It is superior to 1D scalar transfer functions for this dataset as it captures vortices inside larger ones and significant spiral contours [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Demonstrated on CT data sets of the engine (256x256x110) and carp (256x256x512) [S1].
*   **Strengths:** Allows intuitive interaction with cluster hierarchies; can combine TF with segmentation [S1].
*   **Limitations:** Restricts to hard boundaries; fuzzy borders not possible in this implementation [S1].
*   **When to Use:** When automatic grouping of material boundaries is desired and user interaction with hierarchy is preferred [S1].
*   **When Not to Use:** When fuzzy borders are required and GPU-based rendering is not available [S1].

## Background
Originated in the paper "Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries" (EuroVis 2006) [S1]. The method builds on the concept of hierarchical clustering in LH space to automate TF design [S1]. It is used in volume rendering to simplify the process of defining color and opacity mappings for complex datasets [S1].

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
[S2] Volume Rendering, Maximum Intensity Projection and ..., https://ryanrossi.com/sv3.php, fetched 2026-10-28 (unofficial)
