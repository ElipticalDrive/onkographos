# K-means-based hybrid segmentation

Also called: hybrid segmentation

## Overview
This method enhances ray casting object distinction by assigning object membership information (ID tags) to cubical vertices in the volume, generated via a hybrid segmentation algorithm [S1, unofficial]. It differs from standard K-means by using a Self-Organizing Map (SOM) to initialize cluster centers, thereby avoiding local optima, and refining the result using a Diagonal Busyness Factor (DF) for texture and Fisher discriminant ratio for homogeneity [S1, unofficial].

## How it works
**Inputs:**
*   Volume data (treated as image pixels for segmentation).
*   Map dimensions: `map_width`, `map_length`.
*   Parameters: Initial learning rate $\alpha_0 = 0.9$, final learning rate $\alpha_{final} = 0.1$ (Stage 1), refinement learning rate $\alpha_{refine} = 0.01$ (Stage 2).

**Outputs:**
*   Segmented clusters (ID tags) assigned to volume vertices.
*   Binary transfer functions for color and opacity assignment [S1, unofficial].

**Data Structures:**
*   **Neural Map:** A 2D grid of neurons. Each neuron $v$ holds a weight vector $w_v$ (initially 3D for RGB, later extended).
*   **Input Vector $D(t)$:** For Stage 1, this is the normalized RGB color vector. For Stage 1.3 (K-means), this is a 4D vector: $[R, G, B, DF]$.

**Step 1: Optional Pre-processing**
If the input volume has low quality (noise or intensity imbalance), apply high-pass/low-pass filters and histogram equalization to improve feature resolution [S1, unofficial].

**Step 2: Neural Map Initialization (Phase 1.2)**
1.  Calculate total nodes: $N = \text{map\_width} \times \text{map\_length}$.
2.  Find global minimum and maximum RGB values across the volume: $V_{min} = \min(R, G, B)$, $V_{max} = \max(R, G, B)$.
3.  Calculate interval: $I = (V_{max} - V_{min}) / (N - 1)$.
4.  Initialize nodes by traversing from $V_{min}$ to $V_{max}$ using interval $I$.
5.  Normalize feature vectors from $[0, 255]$ to $[-1, 1]$ [S1, unofficial].

**Step 3: SOM Training (Phase 1.2)**
1.  **Stage 1 (Coarse):**
    *   Randomly select pixels from the volume.
    *   Find the Best Matching Unit (BMU) $u$ for input $D(t)$ based on Euclidean distance.
    *   Adjust BMU and its neighborhood.
    *   Learning rate $\theta(s)$ decreases from 0.9 to 0.1.
    *   Neighborhood size decreases from half the map size to 1.
2.  **Stage 2 (Refinement):**
    *   Stabilize feature vectors.
    *   Use learning rate 0.01.
    *   Neighborhood size remains 1 (only BMU adjusted).
3.  **Node Adjustment Formula:**
    $$w_v(s+1) = w_v(s) + \theta(u,v,s)\theta(s)(D(t) - w_v(s))$$
    Where:
    *   $w_v(s)$: Weight vector of neuron $v$ at step $s$.
    *   $\theta(s)$: Monotonically decreasing learning coefficient.
    *   $D(t)$: Input vector (training sample).
    *   $\theta(u,v,s)$: Neighborhood function based on lattice distance between BMU $u$ and neuron $v$ [S1, unofficial].

**Step 4: K-means Clustering (Phase 1.3)**
1.  Use the converged SOM weights as initial cluster centers for K-means.
2.  Limit dimensions to 3 color bands and Diagonal Busyness Factor (DF).
3.  Perform a few rounds of K-means to identify preliminary groups (elementary clusters) [S1, unofficial].
4.  **Diagonal Busyness Factor (DF):**
    $$DF = \sum_{i=1}^{4} |C - x_i|$$
    Where $C$ is the center pixel and $x_i$ are the four diagonal neighbors (top-left, top-right, bottom-left, bottom-right). This probes texture around each pixel [S1, unofficial].

**Step 5: Statistical Optimization**
1.  Apply Fisher discriminant ratio to convert preliminary groups into optimum segments [S1, unofficial].
2.  Use edge detection to exempt important boundary/pattern information from irrelevant recombination.
3.  Apply recombination and epsilon tolerance factor for spatial tuning [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kmeans_hybrid_segmentation(volume, map_width, map_length):
    # Phase 1.1: Pre-processing
    if needs_preprocessing(volume):
        volume = apply_filters(volume)
        volume = histogram_equalize(volume)

    # Phase 1.2: Neural Map Initialization
    num_nodes = map_width * map_length
    min_val = min(volume.flatten())
    max_val = max(volume.flatten())
    interval = (max_val - min_val) / (num_nodes - 1)
    
    # Initialize weights
    weights = []
    current_val = min_val
    for _ in range(num_nodes):
        weights.append(normalize(current_val)) # Normalize to [-1, 1]
        current_val += interval

    # Phase 1.2: Training Stage 1
    learning_rate = 0.9
    neighborhood_size = map_width / 2
    while learning_rate > 0.1:
        sample = random_pixel(volume)
        bmu = find_bmu(sample, weights)
        adjust_neighborhood(weights, bmu, sample, learning_rate, neighborhood_size)
        learning_rate = decay(learning_rate)
        neighborhood_size = max(1, neighborhood_size / 2)

    # Phase 1.2: Training Stage 2 (Refinement)
    learning_rate = 0.01
    neighborhood_size = 1
    while convergence_not_reached(weights):
        sample = random_pixel(volume)
        bmu = find_bmu(sample, weights)
        adjust_neighborhood(weights, bmu, sample, learning_rate, neighborhood_size)

    # Phase 1.3: K-means Clustering
    # Calculate DF for all voxels
    df_map = calculate_diagonal_busyness(volume)
    
    # Combine RGB and DF into 4D feature vectors
    features = combine_rgb_and_df(volume, df_map)
    
    # Use SOM weights (RGB part) as initial centers
    initial_centers = [w[:3] for w in weights] # Assuming weights are normalized
    
    # Run K-means
    clusters = kmeans(features, initial_centers, rounds=3)

    # Statistical Optimization
    optimized_clusters = apply_fisher_discriminant(clusters)
    optimized_clusters = apply_edge_detection(optimized_clusters, volume)
    optimized_clusters = spatial_tuning(optimized_clusters, epsilon=0.1)

    return optimized_clusters
```

## Operating characteristics
*   **Complexity:** Depends on map size and K-means iterations. SOM training is $O(N \cdot M \cdot I)$ where $N$ is pixels, $M$ is nodes, $I$ is iterations. K-means is $O(K \cdot N \cdot J)$ where $K$ is clusters, $J$ is iterations.
*   **Strengths:** Avoids local optima of standard K-means by using SOM initialization; incorporates texture (DF) for better material distinction [S1, unofficial].
*   **Limitations:** Requires careful tuning of map size and learning rates; computationally heavier than simple thresholding [S1, unofficial].

## Background
Proposed by Azimzadeh & Pourgholi in 2020 to enhance ray casting object distinction by assigning object membership information (ID tags) to cubical vertices [S1, unofficial]. It addresses the issue of K-means converging to local optima when initialized randomly [S1, unofficial].

## Sources
[S1] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
