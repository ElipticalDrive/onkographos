# Robust Material Graph Deduction

Also called: Graph-based TF Creation, Material Graph Generation

## Overview
Robust Material Graph Deduction automatically discovers the set of materials and their spatial interconnectivity within a volume to generate a transfer function, solving the problem of manual classification by mapping material interiors rather than just boundaries [S1, unofficial]. It sets itself apart from sibling methods like [Feature Graph](8595-feature-graph.md) by explicitly modeling materials as Gaussian intensity distributions with occlusion values and using a greedy merging strategy based on Bhattacharyya distance to optimize interface separation, whereas siblings often rely on static feature clustering or heuristic ranking [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar volume dataset $V$.
*   **Output:** A material graph $G(M, I)$, where $M$ is the set of deduced materials and $I$ is the set of interfaces between adjacent materials. Each material $M_i \in M$ is defined by a Gaussian intensity distribution $V_i$ with mean $\mu_i$, standard deviation $\sigma_i$, and a mean occlusion value $\eta_i$. Each interface $I_j \in I$ connects two materials.

**Data Structures**
*   **Material:** A tuple $(\mu, \sigma, \eta)$.
*   **Interface:** A pair of material indices $(m_a, m_b)$ and the set of edge pixels belonging to it.
*   **Distance Matrix $D$:** A symmetric matrix where $D_{ij}$ is the Bhattacharyya distance between material $i$ and material $j$.
*   **Edge Set $\hat{I}$:** The initial set of interface edges detected from the volume.

**Algorithm Steps**

1.  **Initial Edge Detection:**
    Compute the initial edge set $\hat{I}$ using the LH histogram method [S1, unofficial]. For each edge pixel, perform a bidirectional traversal along the local gradient to find a low intensity value $L$ and a high intensity value $H$ in a small neighborhood. These values $(L, H)$ are samples from the adjacent intensity distributions $V_{i,1}$ and $V_{i,2}$ of the interface $I_i$ [S1, unofficial].

2.  **Subsampling and Clustering:**
    To ensure computational tractability for large volumes, subsample the interface set from the distribution of $\hat{I}$ [S1, unofficial]. Cluster the $(L, H)$ samples using the HDBSCAN algorithm to define initial material interfaces [S1, unofficial]. This step groups edge pixels into distinct interfaces between potential material pairs.

3.  **Compute Base Threshold:**
    Calculate the base distance threshold $\theta_{base}$ as the minimum Bhattacharyya distance between the two materials in any initial interface:
    $$ \theta_{base} = \min_{i \in \{1, \dots, k\}} d_B(V_{i,1}, V_{i,2}) $$
    where $k$ is the number of initial interfaces and $d_B$ is the Bhattacharyya distance between two Gaussian distributions [S1, unofficial].

4.  **Define Search Range:**
    Choose two multipliers $\epsilon \ll 1$ and $\zeta \gg 1$. The search threshold $\theta$ is chosen from the range $[\epsilon \theta_{base}, \zeta \theta_{base}]$ [S1, unofficial].

5.  **Greedy Merging (for a given $\theta$):**
    Perform greedy merging in two steps to deduce the graph $G(M, I)$:
    *   **Find Materials to Merge:** Walk through the distance matrix $D$ to find sets of similar materials. Create sets of materials that are similar to each other, ensuring that a set does not include both materials of a single interface (to avoid collapsing an interface into a single material). Reduce each set to a single material using a parent-child hierarchy, merging all children with their respective parents [S1, unofficial].
    *   **Reconstruct Graph:** Merge nodes of similar materials together. Merge repeating interface edges (edges between the same pair of materials) into a single interface edge. The resulting material intensity distributions are computed by a weighted average of the constituent material distributions (means and standard deviations) [S1, unofficial].

6.  **Optimal Graph Search:**
    Search for the optimal threshold $\theta_{opt}$ that maximizes the separation energy $E_{sep}$:
    $$ E_{sep}(G(M, I)) = \frac{1}{|I|} \sum_{j=1}^{|I|} d_B(V_{j,1}, V_{j,2}) $$
    Maximizing $E_{sep}$ results in a graph with fewer but well-separated interface edges [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def robust_material_graph_deduction(volume):
    # Step 1: Initial Edge Detection
    # Compute LH histogram to find edge pixels
    # For each edge pixel, traverse gradient to find L and H
    initial_edges = []
    for pixel in volume.edge_pixels():
        L, H = traverse_gradient(pixel, volume)
        initial_edges.append((L, H, pixel))

    # Step 2: Subsampling and Clustering
    # Subsample to keep algorithm tractable
    subsampled_edges = subsample(initial_edges)
    # Cluster using HDBSCAN to define initial interfaces
    # Each cluster represents an interface between two intensity distributions
    interfaces = hdbscan_cluster(subsampled_edges)

    # Step 3: Compute Base Threshold
    # Calculate Bhattacharyya distance for each interface
    distances = []
    for interface in interfaces:
        # interface contains samples from two distributions V1 and V2
        V1, V2 = fit_gaussians(interface.samples)
        dist = bhattacharyya_distance(V1, V2)
        distances.append(dist)
    
    theta_base = min(distances)

    # Step 4: Define Search Range
    epsilon = 0.1  # Example value << 1
    zeta = 10.0    # Example value >> 1
    theta_range = [epsilon * theta_base, zeta * theta_base]

    # Step 5 & 6: Optimal Graph Search
    best_theta = None
    max_energy = -infinity
    best_graph = None

    for theta in search_grid(theta_range):
        # Greedy Merging
        materials, graph = greedy_merge(interfaces, theta)
        
        # Calculate Separation Energy
        if len(graph.interfaces) == 0:
            continue
            
        energy = 0.0
        for interface in graph.interfaces:
            V1, V2 = interface.materials
            energy += bhattacharyya_distance(V1, V2)
        
        energy /= len(graph.interfaces)
        
        if energy > max_energy:
            max_energy = energy
            best_theta = theta
            best_graph = graph

    return best_graph, best_theta

def traverse_gradient(pixel, volume):
    # Bidirectional traversal along local gradient
    # Search for low (L) and high (H) values in small neighborhood
    gradient = compute_gradient(pixel, volume)
    pos = pixel + gradient
    neg = pixel - gradient
    L = find_min_in_neighborhood(neg, volume)
    H = find_max_in_neighborhood(pos, volume)
    return L, H

def fit_gaussians(samples):
    # Fit two Gaussian distributions to the samples
    # Returns (mean1, std1), (mean2, std2)
    # Implementation details omitted [G]
    pass

def bhattacharyya_distance(V1, V2):
    # Calculate Bhattacharyya distance between two Gaussians
    # Formula: 0.25 * ln(0.5 * (sigma1^2 + sigma2^2) / (sigma1 * sigma2)) 
    #         + 0.25 * (mu1 - mu2)^2 / (sigma1^2 + sigma2^2)
    # Note: Exact formula may vary by implementation [G]
    pass

def greedy_merge(interfaces, theta):
    # Create distance matrix D from interface distributions
    D = compute_distance_matrix(interfaces)
    
    # Find sets of similar materials
    # Walk through D, create sets excluding both materials of an interface
    material_sets = find_similar_sets(D, theta)
    
    # Reduce sets to single materials using parent-child hierarchy
    merged_materials = reduce_sets(material_sets)
    
    # Reconstruct graph: merge nodes and repeating edges
    graph = reconstruct_graph(merged_materials, interfaces)
    
    return merged_materials, graph
```

**Parameters**
*   $\epsilon$: Lower bound multiplier for threshold search. Must be much less than 1. Not given in the sources.
*   $\zeta$: Upper bound multiplier for threshold search. Must be much greater than 1. Not given in the sources.
*   Subsampling rate: To keep the algorithm computationally tractable. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Noise:** Noise in volume data and subtle intensity variations can result in repeating boundaries for the same interface. The clustering and merging steps help mitigate this, but excessive noise may lead to over-segmentation before merging [S1, unofficial].
*   **Connectivity:** The algorithm assumes that clustering provides valid material-material interfaces. It does not explicitly model connectivity beyond adjacency, relying on the merging process to infer topology [S1, unofficial].
*   **Background:** The background material is typically set to invisible (white node) to hide it from rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the initial edge detection and the clustering step. The greedy merging is linear in the number of materials/interfaces for a fixed threshold, but the search for optimal $\theta$ requires evaluating multiple thresholds. Not given in the sources.
*   **Memory Complexity:** Depends on the number of edge pixels and the size of the distance matrix $D$. Subsampling is used to manage memory for large volumes [S1, unofficial].
*   **Performance:** In a real-time GPU volume renderer (C++ with Python backend), the graph deduction step took approximately 2-4 seconds on the tested datasets, with voxel classification taking 4-6 seconds (Table 1 in [S1, unofficial]).
*   **Strengths:** Automatically discovers material interiors and spatial interconnectivity; provides fine control over material color and opacities; achieves automatic background removal [S1, unofficial].
*   **Limitations:** Relies on the quality of the LH histogram and HDBSCAN clustering; sensitive to the choice of $\epsilon$ and $\zeta$; assumes materials can be modeled as Gaussian distributions [S1, unofficial].

## Background
The method was proposed in 2018 by authors associated with Eurographics Proceedings [S1, unofficial]. It addresses the limitation of previous methods that focused only on material boundaries, neglecting the interior of materials for visualization. The approach is motivated by the need for robust, automatic transfer function creation that maps both interiors and interfaces. It builds on the LH histogram proposed by Šereda et al. [S1, unofficial].

## Sources
[S1] Robust Material Graphs for Volume Rendering, https://doi.org/10.2312/pg20181282, fetched 2026-11-04 (unofficial)
