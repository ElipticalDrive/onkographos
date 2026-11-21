# Unsupervised Asset Grouping

Also called: Asset Grouping via JSD Clustering

## Overview
Unsupervised Asset Grouping is a method for categorizing discovered objects in a scene by comparing the distributions of color and semantic features among image patches using Jensen-Shannon Divergence (JSD) and agglomerative clustering [S1]. It sets itself apart from sibling methods like [SemGauss-SLAM](9207-semgauss-slam.md) by operating on a discretized ground feature plane to group assets without requiring explicit object pose information or supervised labels, enabling the construction of a cross-scene asset library [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An inferred RGB-DINO ground feature plane $M_{\text{rgb-dino}}$, where each pixel $(x, y)$ is interpolated from discrete grids $M_{\text{rgb}}$ and $M_{\text{dino}}$ via decoders $\text{Dec}_{\text{rgb}}$ and $\text{Dec}_{\text{dino}}$ to form $m_{\text{rgb-dino}} = \text{Interp}(M_{\text{rgb-dino}}, (x, y))$ [S1].
*   **Output:** A set of labeled object feature patches $K$, grouped into clusters, with a selected template object per cluster and a BEV layout defined by rotation angles $\theta$ [S1].

**Data Structures:**
*   $M_{\text{rgb-dino}}$: A 2D grid of features combining color and semantic (DINO) embeddings.
*   $K$: A set of object feature patches, each enclosed in a bounding box.
*   $P_{\text{rgb-dino}}$: The set of all RGB-DINO feature patches in the scene.
*   $\hat{p}$: The template RGB-DINO feature patch selected from a cluster.
*   $H = (H_\sigma, H_{\text{rgb}})$: Shared vertical feature axes (used in cross-scene extension) [S1].

**Steps:**
1.  **Discretization:** Discretize the continuous feature plane $M_{\text{rgb-dino}}$ using a clustering algorithm (e.g., K-means) to identify distinct regions. This results in a set of labeled object feature patches $K$ [S1].
2.  **Similarity Measurement:** For any two object patches $k_i, k_j \in K$, compute the similarity between them using the Jensen-Shannon Divergence over the distribution of their labels, denoted as $\text{JSD}(k_i || k_j)$ [S1].
3.  **Clustering:** Perform agglomerative clustering on the patches in $K$ using the JSD values as the distance metric. The number of clusters is determined by inspecting training views and can be flexibly adjusted to control categorization granularity [S1].
4.  **Template Selection:** Select one template object from each resulting cluster. This selection can be random or user-defined [S1].
5.  **Layout Extraction:** For each template $\hat{p}$ and its copies $p \in P_{\text{rgb-dino}}$, extract the scene layout in Bird's Eye View (BEV) by optimizing a rotation angle $\theta$. The optimal angle $\theta^*$ minimizes the pixel-wise squared error between the template and the rotated copy:
    $$
    \theta^* = \arg\min_\theta \sum_i \|\hat{p}_i - R_\theta(p_i)\|_2^2
    $$
    where $R_\theta$ rotates the input feature patch $p$ by angle $\theta$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unsupervised_asset_grouping(M_rgb_dino):
    # Step 1: Discretize feature plane into labeled patches
    # K is a list of patches, each with a bounding box and label distribution
    K = k_means_clustering(M_rgb_dino)
    
    # Step 2 & 3: Compute JSD distances and perform agglomerative clustering
    # Distance matrix D where D[i,j] = JSD(K[i], K[j])
    D = compute_jsd_distance_matrix(K)
    
    # Perform agglomerative clustering with JSD metric
    # num_clusters is determined by inspecting training views
    clusters = agglomerative_clustering(K, D, num_clusters)
    
    # Step 4: Select templates
    templates = []
    for cluster in clusters:
        # Random or user-defined selection
        template = random.choice(cluster)
        templates.append(template)
        
    # Step 5: Extract BEV layout via rotation optimization
    layout = {}
    for template in templates:
        p_hat = template.patch
        # Find all copies of this asset in the scene
        copies = find_similar_patches(p_hat, M_rgb_dino)
        
        best_theta = 0
        min_loss = infinity
        
        # Optimize rotation angle theta
        for theta in range(0, 360, 1): # Discrete search or gradient descent
            # Rotate copy patches by theta
            rotated_copies = [rotate_patch(p, theta) for p in copies]
            
            # Compute pixel-wise loss
            loss = sum_squared_error(p_hat, rotated_copies)
            
            if loss < min_loss:
                min_loss = loss
                best_theta = theta
        
        layout[template.id] = best_theta
        
    return clusters, templates, layout

def compute_jsd_distance_matrix(patches):
    n = len(patches)
    D = zeros((n, n))
    for i in range(n):
        for j in range(i + 1, n):
            # JSD over label distributions of patches i and j
            D[i, j] = jensen_shannon_divergence(patches[i].labels, patches[j].labels)
            D[j, i] = D[i, j]
    return D
```

**Parameters:**
*   **Clustering Algorithm:** K-means is used for initial discretization [S1].
*   **Distance Metric:** Jensen-Shannon Divergence [S1].
*   **Number of Clusters:** Not fixed; set by inspecting training views [S1].

**Edge Cases and Pitfalls:**
*   **Absence of Pose Information:** Pixel-wise comparison is not ideal without pose; JSD on distributions is required [S1].
*   **Granularity:** The number of clusters must be manually adjusted based on user intent or scene complexity [S1].

## Subtopics

### Cross-scene Asset Library Construction
A framework that represents a scene using template feature patches, BEV layout, shared vertical feature axes, and MLP decoders, enabling cross-scene asset discovery by sharing vertical axes among different scenes [S1]. It constructs a library storing template feature patches by sharing vertical axes across scenes, allowing newly configured scenes to be rendered directly without retraining by using the shared axes and decoders with new template patches and layouts [S1]. This overcomes limitations of voxel-based representations that lack constraints for cross-scene analogue discovery [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of patches $N$ and the clustering algorithm. Agglomerative clustering is typically $O(N^3)$ or $O(N^2 \log N)$ depending on implementation. Optimization of $\theta$ for each template adds a cost proportional to the number of copies and the search space for $\theta$ [S1].
*   **Memory Complexity:** Requires storing the feature plane $M_{\text{rgb-dino}}$, the patches $K$, and the distance matrix $D$ ($O(N^2)$) [S1].
*   **Strengths:** Enables unsupervised categorization and cross-scene asset reuse without retraining [S1].
*   **Limitations:** Requires a synthetic dataset or pre-trained feature extractors (RGB-DINO); granularity is user-dependent [S1].
*   **When to Use:** When building a reusable asset library from multiple scenes or when unsupervised object categorization is needed in ground feature plane representations [S1].

## Background
Originated in the paper "AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation" (2023) [S1]. It addresses the lack of constraints in previous voxel-based neural representations that prevented cross-scene analogue discovery, leveraging the potential of neural representations to share vertical feature axes across scenes [S1].

## Sources
[S1] AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation, https://doi.org/10.1109/iccv51070.2023.00301, fetched 2026-10-29
