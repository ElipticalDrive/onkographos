# K-means-based hybrid segmentation
Also called: hybrid segmentation

## Overview
This method enhances ray casting object distinction in volume rendering by assigning object membership information (ID tags) to cubical vertices via a hybrid segmentation algorithm [S1, unofficial]. It differs from standard clustering by combining unsupervised neural network initialization with K-means clustering and statistical optimization (Fisher discriminant ratio) to produce binary, non-overlapping transfer functions for color and opacity assignment [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (typically RGB color bands; grayscale is supported) [S1, unofficial].
*   **Output:** Object membership ID tags assigned to volume vertices, enabling intra-object buffer coordination [S1, unofficial].

**Data Structures:**
*   **Neural Map:** A 2D lattice of nodes with dimensions `map_width` $\times$ `map_length`. Each node stores a weight vector $w_v$ [S1, unofficial].
*   **Feature Vector:** For K-means, features are limited to three color bands and a Diagonal Busyness Factor (DF) [S1, unofficial].

**Steps:**

1.  **Pre-processing (Optional):** Apply high-pass/low-pass filters and histogram equalization to remove noise and rectify intensity imbalance [S1, unofficial].
2.  **Neural Map Initialization:**
    *   Calculate total nodes: $N = \text{map\_width} \times \text{map\_length}$.
    *   Find global min and max RGB values: $V_{min}, V_{max}$ [S1, unofficial].
    *   Calculate interval: $I = (V_{max} - V_{min}) / (N - 1)$.
    *   Initialize nodes by traversing from $V_{min}$ to $V_{max}$ using step $I$ [S1, unofficial].
3.  **Self-Organized Map (SOM) Training:**
    *   Normalize input feature vectors from $[0, 255]$ to $[-1, 1]$ [S1, unofficial].
    *   Randomly select pixels and find the Best Matching Unit (BMU).
    *   Adjust BMU and neighborhood weights using the update rule:
        $$w_v(s+1) = w_v(s) + \theta(u,v,s)\theta(s)(D(t) - w_v(s))$$
        where $w_v$ is the weight vector, $s$ is the step index, $u$ is the BMU index, $v$ is the neuron, $t$ is the training sample index, $\theta(s)$ is the monotonically decreasing learning coefficient, $D(t)$ is the input vector, and $\theta(u,v,s)$ is the neighborhood function based on lattice distance [S1, unofficial].
    *   **Stage 1:** Learning rate $\theta(s)$ reduces from 0.9 to 0.1. Neighborhood size reduces from half map size to 1 [S1, unofficial].
    *   **Stage 2:** Refinement with learning rate 0.01. Neighborhood size remains 1 (only BMU adjusted) [S1, unofficial].
4.  **K-means Clustering:**
    *   Use SOM output as initial cluster centers.
    *   Limit dimensions to 3 color bands and Diagonal Busyness Factor (DF) [S1, unofficial].
    *   Perform few rounds of K-means to identify preliminary groups [S1, unofficial].
5.  **Statistical Optimization:**
    *   Apply Fisher discriminant ratio to convert preliminary groups into optimum segments [S1, unofficial].
6.  **Post-processing:**
    *   Use edge detection to exempt important boundary/pattern information from irrelevant recombination [S1, unofficial].
    *   Apply recombination and epsilon tolerance factor for spatial tuning [S1, unofficial].

**Diagonal Busyness Factor (DF):**
Used to probe texture around each pixel. Calculated as:
$$DF = \sum_{i=1}^{4} |C - x_i|$$
where $C$ is the center pixel value and $x_i$ are the four diagonal neighbors (top-left, top-right, bottom-left, bottom-right) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kmeans_hybrid_segmentation(volume_data, map_width, map_length):
    # Phase 1.1: Optional Pre-processing
    # Apply filters and histogram equalization if needed [S1, unofficial]
    processed_data = volume_data
    
    # Phase 1.2: Neural Map Initialization
    num_nodes = map_width * map_length
    min_val = min(processed_data.flatten())
    max_val = max(processed_data.flatten())
    interval = (max_val - min_val) / (num_nodes - 1)
    
    # Initialize weights linearly
    weights = [min_val + i * interval for i in range(num_nodes)]
    
    # Phase 1.2: Training Stage 1
    # Normalize inputs to [-1, 1]
    normalized_data = normalize_to_neg1_pos1(processed_data)
    
    learning_rate = 0.9
    neighborhood_size = map_width / 2
    
    # Iterate through training samples
    for sample in normalized_data:
        # Find Best Matching Unit (BMU)
        bmu_index = find_bmu(sample, weights)
        
        # Update weights
        for i in range(num_nodes):
            dist = lattice_distance(i, bmu_index, map_width)
            theta_uv = neighborhood_function(dist, neighborhood_size)
            weights[i] = weights[i] + theta_uv * learning_rate * (sample - weights[i])
            
        # Decay parameters
        learning_rate = decay(learning_rate, start=0.9, end=0.1)
        neighborhood_size = max(1, neighborhood_size / 2)
        
    # Phase 1.2: Training Stage 2 (Refinement)
    learning_rate = 0.01
    neighborhood_size = 1
    
    for sample in normalized_data:
        bmu_index = find_bmu(sample, weights)
        weights[bmu_index] = weights[bmu_index] + learning_rate * (sample - weights[bmu_index])
        
    # Phase 1.3: K-means Clustering
    # Extract features: 3 color bands + DF
    features = extract_features(processed_data)
    initial_centers = weights[:num_clusters] # Assuming num_clusters derived from map size or specified
    
    # Perform few rounds of K-means
    clusters = kmeans(features, initial_centers, rounds=3)
    
    # Statistical Optimization
    # Apply Fisher discriminant ratio to refine clusters
    optimized_clusters = fisher_discriminant_optimization(clusters)
    
    # Post-processing
    # Edge detection and recombination
    id_tags = assign_id_tags(optimized_clusters, processed_data)
    
    return id_tags

def extract_features(volume):
    # Calculate DF for each pixel
    # DF = sum(|C - x_i|) for 4 diagonal neighbors
    features = []
    for pixel in volume:
        c = pixel.value
        neighbors = get_diagonal_neighbors(pixel)
        df = sum(abs(c - n) for n in neighbors)
        features.append([pixel.r, pixel.g, pixel.b, df])
    return features

def lattice_distance(node_idx, bmu_idx, width):
    # Calculate Manhattan or Euclidean distance on 2D grid
    r1, c1 = divmod(node_idx, width)
    r2, c2 = divmod(bmu_idx, width)
    return abs(r1 - r2) + abs(c1 - c2)

def neighborhood_function(dist, size):
    # Binary or Gaussian neighborhood function
    return 1 if dist <= size else 0
```

## Operating characteristics
*   **Time Complexity:** Depends on map size, number of training samples, and K-means iterations. The SOM initialization is $O(N \cdot T)$ where $N$ is nodes and $T$ is samples. K-means is $O(K \cdot I \cdot D)$ where $K$ is clusters, $I$ is iterations, $D$ is dimensions (4 in this method) [S1, unofficial].
*   **Memory Complexity:** Proportional to the neural map size and volume data size [S1, unofficial].
*   **Strengths:** Avoids local optima common in random K-means initialization by using SOM; incorporates texture (DF) for better material separation; produces binary transfer functions [S1, unofficial].
*   **Limitations:** Requires tuning of map dimensions and learning rates; pre-processing is optional but recommended for low-quality data [S1, unofficial].

## Background
Originated in research by Azimzadeh & Pourgholi (2020) to enhance ray casting object distinction by integrating segmentation into the rendering pipeline [S1, unofficial]. It addresses the issue of K-means converging to local optima when initialized randomly [S1, unofficial].

## Sources
[S1] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
