# Joint Learning with Shared MLPs

Also called: Context Model Architecture, Joint training, Shared decoder training

## Overview
Joint Learning with Shared MLPs is a strategy for simultaneously training multiple volumetric visualization (VolVis) scenes using a single, lightweight neural network to decode scene attributes from compact anchor points [S1, unofficial]. It solves the problem of redundant computation and storage when optimizing many similar scenes independently by exploiting inter-scene correlations through a shared context model [S1, unofficial]. Unlike separate optimization, which trains a distinct model for each scene, this method groups geometrically similar scenes and shares the decoding MLP across all members of a group, significantly reducing training time and model size while maintaining reconstruction quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of basic scenes, each represented by a point cloud of anchor positions $\mathbf{x}_a$.
*   **Output:** A set of trained models where a shared MLP decodes probability distributions for anchor attributes (e.g., position, scaling, color, opacity) for all scenes in a group.

**Data Structures:**
1.  **Anchor Points:** Compact representations of the scene geometry. Each anchor $\mathbf{x}_a$ is associated with a set of attributes modeled as a probability distribution.
2.  **Context Model:** A shared neural network consisting of multiple binary hash grids and a shallow Multi-Layer Perceptron (MLP) [S1, unofficial].
    *   **Hash Grids:** Binary grids storing parameter values of either $+1$ or $-1$ [S1, unofficial].
    *   **MLP:** A shallow network that takes aggregated hash features as input and outputs values modeling the probability distribution for anchor attributes [S1, unofficial].
3.  **Scene Groups:** Clusters of scenes determined by pairwise Chamfer distances between their point clouds using hierarchical clustering [S1, unofficial].

**Steps:**
1.  **Scene Grouping:**
    *   For each basic scene, subsample its point cloud to at most 10,000 points to reduce clustering cost [S1, unofficial].
    *   Compute pairwise Chamfer distances between the point clouds of all basic scenes [S1, unofficial].
    *   Apply hierarchical clustering based on these distances to group geometrically similar scenes. The clustering threshold (Chamfer distance) controls the granularity; tested values include 0, 0.005, 0.01, 0.02, and 0.04 [S1, unofficial].
2.  **Joint Training:**
    *   Initialize a shared Context Model (hash grids + shallow MLP) for each group [S1, unofficial].
    *   For each anchor position $\mathbf{x}_a$ in any scene within the group:
        *   Query the hash grids at $\mathbf{x}_a$ to retrieve binary features.
        *   Pass the aggregated features through the shared shallow MLP.
        *   The MLP outputs parameters for the probability distribution of the anchor's attributes [S1, unofficial].
    *   Quantize continuous attribute values to finite sets for entropy coding, injecting noise during quantization to enable backpropagation [S1, unofficial].
    *   Minimize a loss function between rendered observations and ground truth images across all scenes in the group simultaneously [S1, unofficial].
3.  **Inference:**
    *   Use the trained shared Context Model to decode anchor attributes for any scene in the group [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_chamfer_distance(points_a, points_b):
    """
    Compute pairwise Chamfer distance between two point clouds.
    points_a: (N, 3) array
    points_b: (M, 3) array
    Returns: scalar float
    """
    # Compute all pairwise distances
    # dists[i, j] = ||points_a[i] - points_b[j]||^2
    dists = np.sum((points_a[:, np.newaxis, :] - points_b[np.newaxis, :, :]) ** 2, axis=2)
    
    # Chamfer distance is mean of min distances in both directions
    min_dist_a_to_b = np.min(dists, axis=1)
    min_dist_b_to_a = np.min(dists.T, axis=1)
    
    chamfer = np.mean(min_dist_a_to_b) + np.mean(min_dist_b_to_a)
    return chamfer

def hierarchical_clustering(point_clouds, threshold):
    """
    Group scenes based on pairwise Chamfer distances.
    point_clouds: list of (N, 3) arrays
    threshold: float, max Chamfer distance to group scenes
    Returns: list of lists, where each inner list contains indices of scenes in a group
    """
    n = len(point_clouds)
    # Compute pairwise distance matrix
    dist_matrix = np.zeros((n, n))
    for i in range(n):
        for j in range(i + 1, n):
            d = compute_chamfer_distance(point_clouds[i], point_clouds[j])
            dist_matrix[i, j] = d
            dist_matrix[j, i] = d
            
    # Simple agglomerative clustering based on threshold
    # Initialize each scene as its own cluster
    clusters = {i: [i] for i in range(n)}
    used = set()
    
    # Merge clusters if distance < threshold
    # Note: This is a simplified implementation. Real implementation uses 
    # standard hierarchical clustering algorithms (e.g., scipy.cluster.hierarchy)
    # on the distance matrix.
    for i in range(n):
        if i in used:
            continue
        current_cluster = clusters[i]
        for j in range(i + 1, n):
            if j in used:
                continue
            if dist_matrix[i, j] < threshold:
                # Merge j into i's cluster
                current_cluster.extend(clusters[j])
                used.add(j)
        used.add(i)
        clusters[i] = current_cluster
        
    # Return only non-empty clusters
    result = [clusters[i] for i in range(n) if i not in used or clusters[i] == [i]]
    # Re-collect all clusters properly
    final_clusters = []
    visited = set()
    for i in range(n):
        if i not in visited:
            cluster = clusters[i]
            final_clusters.append(cluster)
            visited.update(cluster)
    return final_clusters

class SharedContextModel:
    def __init__(self, num_hash_grids, grid_size, mlp_hidden_dim):
        """
        Initialize the shared context model.
        num_hash_hash_grids: int
        grid_size: int, size of each hash grid
        mlp_hidden_dim: int
        """
        # Initialize binary hash grids with +1 or -1
        self.hash_grids = np.random.choice([1, -1], size=(num_hash_grids, grid_size))
        # Initialize shallow MLP weights
        self.mlp_weights = np.random.randn(mlp_hidden_dim, mlp_hidden_dim)
        self.mlp_bias = np.zeros(mlp_hidden_dim)
        
    def query(self, anchor_position):
        """
        Query the context model for an anchor position.
        anchor_position: (3,) array
        Returns: output of the MLP modeling probability distribution parameters
        """
        # Hash the position to get indices for each grid
        # This is a placeholder for the actual hashing function (e.g., fast hash)
        indices = self._hash_position(anchor_position)
        
        # Retrieve features from hash grids
        features = np.array([self.hash_grids[i][indices[i]] for i in range(len(indices))])
        
        # Pass through MLP
        # Assuming a simple single-layer MLP for demonstration
        hidden = np.tanh(np.dot(features, self.mlp_weights) + self.mlp_bias)
        return hidden
    
    def _hash_position(self, position):
        """
        Hash a 3D position to indices for each hash grid.
        """
        # Placeholder for actual hashing logic
        return [int(p * 1000) % self.hash_grids.shape[1] for p in position]

def joint_learning(scenes, threshold=0.02):
    """
    Perform joint learning across multiple scenes.
    scenes: list of scene objects, each having a point cloud
    threshold: float, clustering threshold
    """
    # Step 1: Subsample point clouds
    subsampled_clouds = []
    for scene in scenes:
        cloud = scene.point_cloud
        if len(cloud) > 10000:
            indices = np.random.choice(len(cloud), 10000, replace=False)
            subsampled_clouds.append(cloud[indices])
        else:
            subsampled_clouds.append(cloud)
            
    # Step 2: Cluster scenes
    groups = hierarchical_clustering(subsampled_clouds, threshold)
    
    # Step 3: Train shared model for each group
    models = []
    for group_indices in groups:
        # Initialize shared model
        model = SharedContextModel(num_hash_grids=10, grid_size=1024, mlp_hidden_dim=64)
        
        # Joint training loop
        # For each anchor in each scene in the group:
        #   - Query model
        #   - Decode attributes
        #   - Render
        #   - Compute loss
        #   - Backpropagate
        # This step is simplified here as the actual training loop is complex
        # and depends on the specific rendering engine and loss function.
        print(f"Training group with {len(group_indices)} scenes")
        # model.train(scenes[group_indices])
        models.append(model)
        
    return models

```

**Parameters:**
*   **Clustering Threshold:** Controls the grouping of scenes. Values tested: 0, 0.005, 0.01, 0.02, 0.04 [S1, unofficial]. Larger thresholds group more scenes, reducing the number of groups and shared models.
*   **Subsampling Limit:** Point clouds are subsampled to at most 10,000 points for clustering [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **OOM Errors:** Setting the clustering threshold too high (e.g., 0.04) can group more than 20 scenes together, causing Out-Of-Memory (OOM) errors during training because the anchor/optimizer parameters and training images cannot fit in GPU memory [S1, unofficial]. Peak memory usage occurs during the joint training phase [S1, unofficial].
*   **Quality Degradation:** Joint learning across fundamentally unrelated datasets (e.g., beetle and argon bubble) degrades reconstruction quality because the shared MLP lacks the capacity to encode different volume structures simultaneously [S1, unofficial]. Scene grouping based on Chamfer distance is essential to avoid this [S1, unofficial].
*   **Inference Memory:** While training memory can be high, inference memory is significantly lower (<2 GB) as only the trained model needs to be loaded [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Clustering Overhead:** Quadratic in the number of basic scenes due to pairwise Chamfer distance computation. However, subsampling to 10K points keeps this negligible (e.g., 0.49s for 41 scenes, <0.02% of total training time) [S1, unofficial].
*   **Training Time:** Joint learning reduces training time by 35% to 54% compared to separate optimization [S1, unofficial].
*   **Model Size:** Reduces model size by 44% to 60% [S1, unofficial].
*   **Anchor Count:** Reduces the total number of anchors needed [S1, unofficial].

**Measured Performance:**
*   On the multivariate combustion dataset (41 scenes):
    *   Training time: 156.9 min (separate) vs. 72.0 min (joint) [S1, unofficial].
    *   Model size: 45.75 MB (separate) vs. 15.99 MB (joint) [S1, unofficial].
    *   Anchors: 1,049,074 (separate) vs. 415,275 (joint) [S1, unofficial].
*   Savings grow monotonically with the number of basic scenes [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant reductions in training time, model size, and anchor count. Preserves fast training and rendering performance. Effective for geometrically similar scenes [S1, unofficial].
*   **Limitations:** Requires careful grouping of scenes; ineffective or harmful for unrelated scenes. Memory-intensive during joint training if groups are too large [S1, unofficial].

**When to Use:**
*   When optimizing many volumetric scenes that are geometrically similar.
*   When computational resources (time, memory, storage) are limited.
*   When scene grouping can be effectively determined by Chamfer distance [S1, unofficial].

**When Not to Use:**
*   When scenes are fundamentally unrelated, as quality will degrade [S1, unofficial].
*   When GPU memory is extremely limited and the number of similar scenes is large, risking OOM errors [S1, unofficial].

## Background
**Origin:**
The method is presented in the paper "ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization" [S1, unofficial]. It leverages a context model architecture (based on BiRF [42]) that uses multiple binary hash grids and a shallow MLP to decode neural editable Gaussians from compact anchor points [S1, unofficial].

**History and Development:**
The approach is part of the ECoNGS framework, which aims to provide efficient compressive neural Gaussian splatting for volume visualization. It combines joint learning with neural entropy coding to achieve superior reconstruction quality, smaller model size, and faster convergence compared to existing VolVis scene representation methods [S1, unofficial].

**Usage:**
Used in volume visualization (VolVis) for efficiently representing and rendering multiple volumetric scenes, particularly in scientific visualization contexts where many similar simulations or datasets need to be visualized [S1, unofficial].

## Sources
[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
