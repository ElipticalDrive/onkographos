# K-means-based hybrid segmentation

Also called: hybrid segmentation

## Overview
This method enhances ray casting object distinction by assigning object membership information (ID tags) to cubical vertices in the volume, generated via a hybrid segmentation algorithm [S1, unofficial]. It enables the utilization of binary (non-overlapping) transfer functions for color and opacity assignment [S1, unofficial]. It sets itself apart from sibling methods by combining unsupervised neural network initialization with K-means clustering and statistical optimization (Fisher discriminant ratio) to avoid local optima, rather than relying solely on geometric features or direct mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (represented as pixels/voxels with RGB color bands). Optional pre-processing filters.
*   **Output:** Object membership ID tags assigned to cubical vertices; preliminary cluster centers; final segmented groups [S1, unofficial].

**Data Structures:**
*   **Neural Map:** A 2D grid of nodes with dimensions `map_width` $\times$ `map_length`. Each node holds a weight vector $w_v$ (normalized to $[-1, 1]$) [S1, unofficial].
*   **Feature Vector:** For K-means, the feature vector for a pixel includes 3 color bands (R, G, B) and a Diagonal Busyness Factor (DF) [S1, unofficial].

**Steps:**

1.  **Pre-processing (Optional):** If input is low quality (noise/intensity imbalance), apply high pass/low pass filters and histogram equalization to improve feature resolution [S1, unofficial].

2.  **Neural Map Initialization:**
    *   Calculate total nodes: $N = \text{map\_width} \times \text{map\_length}$ [S1, unofficial].
    *   Find min and max RGB values across the dataset: $V_{min} = \min(R, G, B)$, $V_{max} = \max(R, G, B)$ [S1, unofficial].
    *   Calculate interval: $I = (V_{max} - V_{min}) / (N - 1)$ [S1, unofficial].
    *   Initialize nodes by traversing from $V_{min}$ to $V_{max}$ using interval $I$. Normalize feature vectors from $[0, 255]$ to $[-1, 1]$ [S1, unofficial].

3.  **Training Stage 1 (SOM):**
    *   Randomly select pixels from the input.
    *   Find the Best Matching Unit (BMU) $u$ for input vector $D(t)$.
    *   Adjust BMU and neighborhood weights using:
        $$w_v(s+1) = w_v(s) + \theta(u,v,s)\theta(s)(D(t) - w_v(s))$$
        Where:
        *   $w_v(s)$ is the weight vector of neuron $v$ at step $s$.
        *   $\theta(s)$ is a monotonically decreasing learning coefficient.
        *   $\theta(u,v,s)$ is the neighborhood function based on lattice distance between BMU $u$ and neuron $v$.
        *   $D(t)$ is the input vector [S1, unofficial].
    *   **Parameters:**
        *   Learning rate $\theta(s)$: Reduces from 0.9 to 0.1.
        *   Neighborhood size: Reduces from half map size to 1.
    *   **Training Stage 2 (Refinement):**
        *   Stabilize feature vectors.
        *   Learning rate fixed at 0.01.
        *   Neighborhood size fixed at 1 (only BMU adjusted) [S1, unofficial].

4.  **K-means Clustering (Phase 1.3):**
    *   Use centers from the trained Neural Map as initial seeds for K-means [S1, unofficial].
    *   Limit dimensions to 3 color bands and Diagonal Busyness Factor (DF) [S1, unofficial].
    *   Perform few rounds of K-means to identify preliminary groups [S1, unofficial].
    *   **DF Calculation:**
        $$DF = \sum_{i=1}^{4} |C - x_i|$$
        Where $C$ is the center pixel and $x_i$ are the four diagonal neighbors (top-left, top-right, bottom-left, bottom-right) [S1, unofficial].

5.  **Statistical Optimization:**
    *   Apply Fisher discriminant ratio to convert preliminary groups into optimum segments [S1, unofficial].
    *   Identify homogeneous clusters where a center possesses minimum distance (mean squared error) to data points compared to other centers [S1, unofficial].

6.  **Post-processing:**
    *   Use edge detection to exempt important boundary/pattern information from irrelevant recombination [S1, unofficial].
    *   Use recombination and epsilon tolerance factor for spatial tuning [S1, unofficial].
    *   Assign object membership ID tags to cubical vertices [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kmeans_hybrid_segmentation(volume_data, map_width, map_length):
    # Phase 1.1: Pre-processing (Optional)
    # Apply high/low pass filters and histogram equalization if needed
    data = volume_data
    
    # Phase 1.2: Neural Map Initialization
    num_nodes = map_width * map_length
    min_val = min(data.flatten())
    max_val = max(data.flatten())
    interval = (max_val - min_val) / (num_nodes - 1)
    
    # Initialize weights
    weights = []
    current_val = min_val
    for _ in range(num_nodes):
        # Normalize to [-1, 1]
        normalized = 2 * (current_val - min_val) / (max_val - min_val) - 1
        weights.append([normalized, normalized, normalized]) # 3D vector
        current_val += interval
        
    # Phase 1.2: Training Stage 1
    learning_rate = 0.9
    neighborhood_size = map_width / 2
    for epoch in range(num_epochs):
        # Randomly select pixels
        sample = random_pixel(data)
        # Find BMU
        bmu_idx = argmin([euclidean_distance(sample, w) for w in weights])
        # Adjust weights
        for i, w in enumerate(weights):
            dist = lattice_distance(i, bmu_idx, map_width)
            if dist <= neighborhood_size:
                theta_u_v = neighborhood_function(dist, neighborhood_size)
                theta_s = learning_rate
                weights[i] = w + theta_u_v * theta_s * (sample - w)
        
        # Decay parameters
        learning_rate = decay(0.9, 0.1, epoch)
        neighborhood_size = decay(map_width/2, 1, epoch)
        
    # Phase 1.2: Training Stage 2 (Refinement)
    learning_rate = 0.01
    neighborhood_size = 1
    for epoch in range(refinement_epochs):
        sample = random_pixel(data)
        bmu_idx = argmin([euclidean_distance(sample, w) for w in weights])
        weights[bmu_idx] = weights[bmu_idx] + learning_rate * (sample - weights[bmu_idx])
        
    # Phase 1.3: K-means Clustering
    # Calculate DF for each pixel
    df_values = calculate_diagonal_busyness(data)
    features = combine_rgb_and_df(data, df_values)
    
    # Use SOM weights as initial centers
    centers = weights[:k] # Assuming k centers derived from map
    labels = kmeans_cluster(features, centers, max_iters=5)
    
    # Statistical Optimization
    final_segments = optimize_with_fisher_discriminant(labels, features)
    
    # Post-processing
    edges = detect_edges(data)
    final_ids = assign_ids(final_segments, edges)
    
    return final_ids
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of map nodes, training epochs, and K-means iterations. The SOM training is $O(N \cdot E \cdot D)$ where $N$ is nodes, $E$ is epochs, $D$ is dimensionality. K-means is $O(I \cdot M \cdot K \cdot D)$ where $I$ is iterations, $M$ is samples.
*   **Memory Complexity:** $O(N \cdot D)$ for the neural map weights.
*   **Strengths:** Avoids local optima common in random K-means initialization; incorporates texture (DF) and color; produces binary non-overlapping transfer functions [S1, unofficial].
*   **Limitations:** Requires careful tuning of map size and learning rates; pre-processing is optional but recommended for noisy data; DF calculation adds computational overhead [S1, unofficial].
*   **When to use:** When object distinction in ray casting is poor due to similar colors or lack of clear boundaries; when binary transfer functions are desired [S1, unofficial].

## Background
Originated in a 2020 paper by Azimzadeh & Pourgholi, published in the *Journal of AI and Data Mining* [S1, unofficial]. The method was proposed to enhance ray casting technique object distinction by modifying the architecture to accommodate object membership information generated by this hybrid segmentation algorithm [S1, unofficial]. It addresses the issue of K-means converging to local optima when initialized randomly by using a self-organized neural network for initialization [S1, unofficial].

## Sources
[S1] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
