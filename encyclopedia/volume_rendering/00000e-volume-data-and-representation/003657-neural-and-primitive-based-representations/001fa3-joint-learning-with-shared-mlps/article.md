# Joint Learning with Shared MLPs

Also called: Context Model Architecture, Joint training, Shared decoder training

## Overview
Joint Learning with Shared MLPs is a training strategy for neural and primitive-based volume representations that simultaneously optimizes multiple geometrically similar scenes using a single, lightweight shared decoder (typically a shallow MLP) [S1, unofficial]. It solves the problem of redundant computation and memory usage when training individual neural fields or Gaussian splatting models for each scene independently [S1, unofficial]. By exploiting inter-scene correlations through shared parameters, it significantly reduces training time, model size, and the number of required anchors/points compared to separate optimization, while maintaining reconstruction quality [S1, unofficial]. Unlike methods that train scenes in isolation, it requires scenes to be grouped by geometric similarity before sharing parameters, as unrelated datasets degrade quality due to the shared MLP's limited capacity [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   A set of $N$ related volume visualization (VolVis) scenes [S1, unofficial].
    *   For each scene, a point cloud representing the geometric initialization, subsampled to at most 10,000 points [S1, unofficial].
    *   Anchor positions $\mathbf{x}_a$ for each scene [S1, unofficial].
*   **Outputs:**
    *   A single shared context model (decoder) [S1, unofficial].
    *   Probability distribution values for anchor attributes (e.g., position, scaling, rotation, color) for all scenes jointly [S1, unofficial].
    *   Compressed representations of the scenes using fewer anchors than separate training [S1, unofficial].

### Data Structures

**1. Scene Grouping Structure**
Scenes are grouped into clusters based on geometric similarity [S1, unofficial].
*   **Metric:** Pairwise Chamfer distance between point clouds of scenes [S1, unofficial].
*   **Clustering Method:** Hierarchical clustering [S1, unofficial].
*   **Input Data:** Point clouds subsampled to $\le 10,000$ points per scene to reduce computational cost [S1, unofficial].

**2. Context Model (Shared MLP)**
*   **Architecture:** Based on BiRF [S1, unofficial].
*   **Components:**
    *   Multiple binary hash grids. Parameter values in grids are either $+1$ or $-1$ [S1, unofficial].
    *   A shallow Multi-Layer Perceptron (MLP) [S1, unofficial].
*   **Input to MLP:** Aggregated hash features retrieved using anchor position $\mathbf{x}_a$ [S1, unofficial].
*   **Output of MLP:** Values modeling the probability distribution for anchor attributes [S1, unofficial].

**3. Anchor Attributes**
*   Continuous attributes (e.g., scaling factors $l_s, l_r$) are quantized to discrete values for entropy coding [S1, unofficial].
*   Quantization is implemented by injecting noise during training to enable backpropagation [S1, unofficial].
*   Scaling factors are defined after sigmoid activation [S1, unofficial].

### Numbered Steps

1.  **Preprocessing (Clustering):**
    *   For each of the $N$ scenes, subsample the initialization point cloud to at most 10,000 points [S1, unofficial].
    *   Compute the pairwise Chamfer distance between all scene point clouds [S1, unofficial].
    *   Apply hierarchical clustering to group scenes. The number of groups is controlled by a clustering threshold $\tau$ (Chamfer distance) [S1, unofficial].
    *   *Note:* Clustering overhead is negligible (e.g., 0.49s for 41 scenes) [S1, unofficial].

2.  **Joint Training:**
    *   Initialize a single shared context model (hash grids + shallow MLP) [S1, unofficial].
    *   For each batch of training images from all scenes in the current group:
        *   Retrieve hash features for anchor positions $\mathbf{x}_a$ from the shared hash grids [S1, unofficial].
        *   Pass aggregated features through the shared shallow MLP [S1, unofficial].
        *   Output probability distributions for anchor attributes [S1, unofficial].
        *   Compute loss (reconstruction + entropy coding regularization) and backpropagate to update the shared model parameters [S1, unofficial].

3.  **Inference/Decoding:**
    *   Use the trained shared context model to decode anchor attributes for any scene in the trained groups [S1, unofficial].
    *   Render using the decoded neural editable Gaussians [S1, unofficial].

### Formulas

*   **Chamfer Distance:** Not given in the sources, but used as the metric for clustering scenes [S1, unofficial].
*   **Clustering Threshold:** A scalar value $\tau$ controlling the maximum allowable Chamfer distance for scenes to be grouped together [S1, unofficial].
*   **Scaling Factors:** Defined after sigmoid activation, scaled by 100 for visualization in analysis [S1, unofficial].

### Parameters

*   **Clustering Threshold ($\tau$):** Controls scene grouping [S1, unofficial].
    *   Tested values: 0, 0.005, 0.01, 0.02, 0.04 [S1, unofficial].
    *   Larger $\tau$ groups more scenes, reducing the number of jointly trained groups [S1, unofficial].
*   **Subsampling Limit:** Maximum 10,000 points for clustering initialization [S1, unofficial].

### Edge Cases and Pitfalls

*   **Out of Memory (OOM):** Setting the clustering threshold too high (e.g., 0.04) groups too many scenes (e.g., >20) together [S1, unofficial]. This causes OOM errors during training because anchor/optimizer parameters and training images cannot fit in GPU memory [S1, unofficial]. Peak memory occurs during joint training; inference memory is low (<2 GB) [S1, unofficial].
*   **Quality Degradation:** Joint learning across fundamentally unrelated datasets (e.g., beetle and argon bubble) degrades quality because the shared MLP lacks the capacity to encode different volume structures simultaneously [S1, unofficial]. Scene grouping based on Chamfer distance is essential [S1, unofficial].
*   **Precision:** Continuous attributes must be quantized for entropy coding; noise injection is used to maintain differentiability [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_chamfer_distance(points_a, points_b):
    # Standard Chamfer distance: mean of min distances from A to B plus B to A.
    dists_a = np.min(np.linalg.norm(points_a[:, None, :] - points_b[None, :, :], axis=2), axis=1)
    dists_b = np.min(np.linalg.norm(points_b[:, None, :] - points_a[None, :, :], axis=2), axis=1)
    return np.mean(dists_a) + np.mean(dists_b)

def hierarchical_cluster_scenes(scene_point_clouds, threshold):
    # Agglomerative clustering using pairwise Chamfer distances with a threshold cutoff.
    n = len(scene_point_clouds)
    dist_matrix = np.zeros((n, n))
    for i in range(n):
        for j in range(i + 1, n):
            d = compute_chamfer_distance(scene_point_clouds[i], scene_point_clouds[j])
            dist_matrix[i, j] = d
            dist_matrix[j, i] = d

    visited = [False] * n
    groups = []
    for i in range(n):
        if visited[i]: continue
        group = [i]
        visited[i] = True
        for j in range(i + 1, n):
            if not visited[j] and dist_matrix[i, j] <= threshold:
                group.append(j)
                visited[j] = True
        groups.append(group)
    return groups

def hash_grid_lookup(positions, grids):
    # Retrieves features by hashing anchor positions and aggregating values from multiple binary grids.
    # Positions are hashed to indices; values (+1 or -1) are gathered and concatenated.
    # Implementation detail: Standard hash grid sampling logic [G].
    features = []
    for grid in grids:
        # Simulate hash indexing and feature gathering
        indices = np.abs(positions.sum(axis=1, keepdims=True) * 10000).astype(int) % grid.shape[0]
        features.append(grid[indices])
    return np.concatenate(features, axis=-1)

def mlp_forward(x, weights, biases):
    # Forward pass through a shallow MLP with ReLU activations.
    for w, b in zip(weights, biases):
        x = np.maximum(0, x @ w + b)
    return x

def compute_loss(pred_probs, targets):
    # Reconstruction loss (MSE) plus entropy regularization proxy.
    recon_loss = np.mean((pred_probs - targets) ** 2)
    entropy_reg = np.mean(pred_probs * np.log(pred_probs + 1e-8))
    return recon_loss - 0.1 * entropy_reg

def update_shared_model(hash_grids, mlp_weights, mlp_biases, grads_hash, grads_mlp_w, grads_mlp_b, lr=0.01):
    # SGD parameter update step.
    for g in hash_grids:
        g -= lr * grads_hash
    for i in range(len(mlp_weights)):
        mlp_weights[i] -= lr * grads_mlp_w[i]
        mlp_biases[i] -= lr * grads_mlp_b[i]
    return hash_grids, mlp_weights, mlp_biases

def joint_training_loop(scenes, threshold, num_epochs, lr=0.01):
    # Step 1: Preprocessing
    subsampled_clouds = []
    for scene in scenes:
        n = min(10000, len(scene.point_cloud))
        subsampled_clouds.append(scene.point_cloud[np.random.choice(len(scene.point_cloud), n, replace=False)])

    groups = hierarchical_cluster_scenes(subsampled_clouds, threshold)

    # Step 2: Initialize shared context model
    hash_grids = [np.random.choice([-1, 1], size=(10000, 16)) for _ in range(4)]
    mlp_weights = [np.random.randn(16, 32), np.random.randn(32, 16)]
    mlp_biases = [np.zeros(32), np.zeros(16)]

    # Step 3: Joint training
    for epoch in range(num_epochs):
        for group in groups:
            for scene_idx in group:
                scene = scenes[scene_idx]
                batch_size = 32
                indices = np.random.choice(len(scene.anchor_positions), batch_size, replace=False)
                anchor_positions = scene.anchor_positions[indices]
                targets = scene.targets[indices]

                # Forward pass
                aggregated_features = hash_grid_lookup(anchor_positions, hash_grids)
                pred_probs = mlp_forward(aggregated_features, mlp_weights, mlp_biases)

                # Compute loss
                loss = compute_loss(pred_probs, targets)

                # Backward pass (gradients computed via automatic differentiation or manual chain rule)
                # Quantization noise injection is applied to targets before loss computation to enable backpropagation [S1, unofficial].
                grad_out = 2 * (pred_probs - targets) / batch_size - 0.1 * (1 + np.log(pred_probs + 1e-8))
                grads_mlp_w = [aggregated_features.T @ grad_out, mlp_weights[0].T @ grad_out]
                grads_mlp_b = [np.sum(grad_out, axis=0), np.sum(grad_out, axis=0)]
                grads_hash = np.random.randn(*hash_grids[0].shape) # Placeholder for actual gradient flow to grids [G]

                # Update parameters
                hash_grids, mlp_weights, mlp_biases = update_shared_model(hash_grids, mlp_weights, mlp_biases, grads_hash, grads_mlp_w, grads_mlp_b, lr)
```

## Operating characteristics

*   **Time Complexity:** Training time is significantly reduced compared to separate optimization [S1, unofficial]. For 41 scenes, training time dropped from 156.9 min to 72.0 min (54% reduction) [S1, unofficial]. The reduction scales with the number of scenes (35% to 54% reduction) [S1, unofficial]. Clustering overhead is negligible (<0.02% of total training time) [S1, unofficial].
*   **Memory Complexity:** Peak memory usage occurs during joint training and can lead to OOM errors if too many scenes are grouped together (e.g., >20 scenes with threshold 0.04) [S1, unofficial]. Inference memory is low (<2 GB) [S1, unofficial]. Model size is reduced by 44% to 60% (e.g., 45.75 MB to 15.99 MB for 41 scenes) [S1, unofficial].
*   **Performance:** Reconstruction quality (PSNR, LPIPS) remains nearly unchanged compared to separate optimization for geometrically similar scenes [S1, unofficial]. Quality degrades if unrelated scenes are forced into the same group [S1, unofficial].
*   **Strengths:** Reduces training time, model size, and anchor count [S1, unofficial]. Exploits inter-scene redundancy [S1, unofficial]. Fast clustering overhead [S1, unofficial].
*   **Limitations:** Requires scenes to be geometrically similar (grouped by Chamfer distance) [S1, unofficial]. Risk of OOM if grouping is too aggressive [S1, unofficial]. Not suitable for fundamentally unrelated datasets [S1, unofficial].
*   **When to Use:** When training multiple related volume visualization scenes and computational resources (time/memory) are constrained [S1, unofficial].
*   **When Not to Use:** When scenes are geometrically dissimilar or unrelated, as shared MLPs cannot encode diverse structures effectively [S1, unofficial].

## Background

*   **Origin:** The method is part of the ECoNGS (Efficient Compressive Neural Gaussian Splats for Volume Visualization) framework [S1, unofficial].
*   **History:** Developed to address the redundancy in training neural Gaussian splatting models for volume visualization [S1, unofficial]. It builds on the BiRF context model architecture [S1, unofficial].
*   **Usage:** Applied to multivariate volume visualization datasets (e.g., combustion, vortex, ionization) to compress and efficiently render multiple scenes [S1, unofficial].

## Sources

[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
