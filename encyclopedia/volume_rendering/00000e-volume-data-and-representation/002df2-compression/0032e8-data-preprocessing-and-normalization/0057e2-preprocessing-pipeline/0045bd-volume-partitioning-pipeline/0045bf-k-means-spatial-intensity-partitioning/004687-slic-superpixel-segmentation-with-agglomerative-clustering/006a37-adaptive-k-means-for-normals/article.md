# Adaptive K-means for Normals

Also called: Adaptive K-means Clustering for Normals

## Overview
Adaptive K-means for Normals is a method to determine the optimal number of clusters $k$ for surface normals within a superpixel, up to a maximum limit $k_{max}$ [S1]. It solves the problem of photometric homogeneity in textureless areas, where standard superpixel segmentation fails to capture geometric boundaries (e.g., corners) because the visual appearance is uniform [S1]. Unlike fixed-$k$ clustering, this method adaptively increases cluster count until the internal smoothness of all resulting clusters exceeds a threshold $\tau_n$, ensuring that distinct geometric planes are separated [S1]. It builds on [SLIC Superpixel Segmentation with Agglomerative Clustering](11782-slic-superpixel-segmentation-with-agglomerative-clustering.md) by processing the normals generated from its superpixels.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of normals $\{n\}$ belonging to a single superpixel $A$.
*   **Parameters:** Smoothness threshold $\tau_n = 0.9$, maximum clustering limit $k_{max} = 3$ [S1].
*   **Output:** A list of principal normals, one for each identified cluster.

**Data Structures:**
*   $\{n\}$: A list or array of 3D normal vectors.
*   $\{n\}_j$: The subset of normals assigned to cluster $j$.
*   $\delta(\{n\}_j)$: A scalar smoothness metric for cluster $j$. The source defines the condition $\delta(\{n\}_j) > \tau_n$ but does not provide the explicit formula for $\delta$. [G] In the context of normal clustering, $\delta$ typically represents the mean cosine similarity or the inverse of the variance of the normals within the cluster.
*   $n_{principal}$: The centroid normal of a cluster.

**Steps:**
1.  Initialize $k = 1$ and an empty output list `output`.
2.  While $k \le k_{max}$:
    a.  Perform K-means clustering on $\{n\}$ with $k$ clusters to obtain clusters $\{\{n\}_j\}_{j=1}^k$.
    b.  Check if the smoothness condition holds for all clusters: if $\forall j \in \{1, \dots, k\}, \delta(\{n\}_j) > \tau_n$:
        i.  For each cluster $\{n\}_j$:
            1.  Calculate the principal normal: $n_{principal} = \frac{1}{|\{n\}_j|} \sum_{n \in \{n\}_j} n$.
            2.  Append $n_{principal}$ to `output`.
        ii. Break the loop.
    c.  Else: Increment $k = k + 1$.
3.  Return `output`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_kmeans_normals(normals, tau_n=0.9, k_max=3):
    """
    Adaptive K-means clustering for normals.
    
    Args:
        normals: List of 3D normal vectors (numpy arrays or lists).
        tau_n: Smoothness threshold.
        k_max: Maximum number of clusters to try.
        
    Returns:
        List of principal normal vectors.
    """
    output = []
    k = 1
    
    while k <= k_max:
        # Perform K-means clustering
        # Note: Standard K-means implementation required. 
        # Returns a list of lists: clusters[j] contains normals for cluster j.
        clusters = kmeans(normals, k)
        
        # Check smoothness condition for all clusters
        all_smooth = True
        for cluster_normals in clusters:
            # Calculate smoothness metric delta
            # [G] delta is not explicitly defined in sources. 
            # Assuming delta is mean cosine similarity or 1 - variance.
            # Implementation detail: compute delta(cluster_normals)
            delta = compute_smoothness(cluster_normals)
            if delta <= tau_n:
                all_smooth = False
                break
        
        if all_smooth:
            # Extract principal normals
            for cluster_normals in clusters:
                # Calculate centroid
                n_principal = sum(cluster_normals) / len(cluster_normals)
                # Normalize to ensure unit length
                n_principal = n_principal / norm(n_principal)
                output.append(n_principal)
            break
        else:
            k += 1
            
    return output

def compute_smoothness(normals):
    """
    Placeholder for the undefined smoothness metric delta.
    The source [S1] uses delta({n}_j) > tau_n but does not define delta.
    Common implementations use mean cosine similarity or inverse variance.
    """
    # [G] Generic implementation: Mean cosine similarity to centroid
    if not normals:
        return 0.0
    centroid = sum(normals) / len(normals)
    centroid = centroid / norm(centroid)
    similarities = [dot(n, centroid) for n in normals]
    return sum(similarities) / len(similarities)
```

**Edge Cases and Pitfalls:**
*   If $k$ reaches $k_{max}$ and the smoothness condition is still not met, the loop terminates without adding the current clusters to the output (or potentially returns the last failed state depending on implementation, though the pseudocode implies an empty return if the break is never hit). The source does not specify behavior if the condition is never met within $k_{max}$.
*   Precision issues in normal normalization should be handled.

## Subtopics

### Textureless Surface Area Handling
A scheme to handle textureless surface areas by leveraging superpixel homogeneity and adaptive K-means clustering to extract principal normals [S1]. It identifies textureless superpixels as those with a smoothness score over 0.9 (from Algorithm 1) [S1]. It conducts adaptive K-means clustering on these superpixels to extract principal normals, assigns internal pixels with these normals to obtain a clustered normal map, and conducts mesh-guided consistency among multi-view images on these maps [S1]. The resulting smooth normal maps regularize the learning of the neural implicit surface in HelixSurf [S1]. For textureless areas totally not covered by MVS predictions, it initializes with normals generated using the Manhattan assumption [S1]. A pitfall is that photometric homogeneity inside superpixels may not support correct geometric partition of textureless areas, such as confusing corners [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The method involves iterative K-means runs up to $k_{max}$ (typically 3), which is computationally light relative to the overall HelixSurf pipeline.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Adaptively handles textureless regions where standard segmentation fails, improving geometric accuracy for neural implicit surface learning [S1].
*   **Limitations:** Relies on the assumption that textureless areas have low smoothness scores in the superpixel metric; may confuse geometric features like corners if photometric homogeneity is misleading [S1].

**When to Use:**
Use when processing multivariate volumes or images with significant textureless regions (e.g., indoor scenes with white walls) where standard superpixel segmentation fails to capture geometric boundaries [S1].

## Background
Originated in the HelixSurf paper by [S1] (2023). It is part of the HelixSurf framework for robust and efficient neural implicit surface learning of indoor scenes. The method is specifically designed to address the limitations of Multi-View Stereo (MVS) in textureless areas by using superpixel-based adaptive clustering to regularize normal maps.

## Sources
[S1] HelixSurf: A Robust and Efficient Neural Implicit Surface Learning of Indoor Scenes with Iterative Intertwined Regularization, https://arxiv.org/abs/2302.14340v2, fetched 2026-10-26
