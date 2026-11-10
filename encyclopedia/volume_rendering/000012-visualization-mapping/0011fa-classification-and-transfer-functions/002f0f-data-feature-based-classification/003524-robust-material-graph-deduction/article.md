# Robust Material Graph Deduction

Also called: Graph-based TF Creation, Material Graph Generation

## Overview
Robust Material Graph Deduction automatically discovers material interiors and their spatial interconnectivity within a volume, solving the problem of manual transfer function creation by generating a graph $G(M, I)$ where $M$ is the set of materials and $I$ is the set of interfaces [S1, unofficial]. It sets itself apart from sibling methods like [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) or [Feature Graph](8595-feature-graph.md) by explicitly modeling materials as Gaussian intensity distributions with occlusion values and deducing connectivity through a greedy merging process optimized by separation energy, rather than relying solely on geometric curvature or static feature clustering [S1, unofficial].

## How it works
**Inputs:** A 3D volume dataset $V$.
**Outputs:** A material graph $G(M, I)$, where $M$ is the set of discovered materials and $I$ is the set of interfaces between adjacent materials. Each material $M_i \in M$ is defined by an intensity distribution $V_i$ (Gaussian with mean $\mu_i$, standard deviation $\sigma_i$) and a mean occlusion value $\eta_i$ [S1, unofficial].

**Data Structures:**
*   **Initial Edge Set $\hat{I}$:** A set of tuples $(L, H)$, where $L$ and $H$ are intensity values sampled from adjacent distributions at an interface.
*   **Distance Matrix $D$:** A symmetric matrix where $D_{ij}$ is the Bhattacharyya distance between material distributions $V_i$ and $V_j$.
*   **Material Graph $G(M, I)$:** An adjacency structure where nodes are materials and edges are interfaces.

**Steps:**

1.  **Compute Initial Edge Set $\hat{I}$:**
    For each edge pixel in the volume, perform a bidirectional traversal along the local gradient to search for a low value $L$ and a high value $H$ in a small neighborhood [S1, unofficial]. These values are samples from adjacent intensity distributions $V_{i,1}$ and $V_{i,2}$ [S1, unofficial].

2.  **Cluster Interfaces:**
    Subsample the interface set from their distribution to keep the algorithm computationally tractable for large volumes [S1, unofficial]. Cluster the $(L, H)$ samples from $\hat{I}$ using the HDBSCAN algorithm to define initial material interfaces [S1, unofficial]. This yields a set of initial material pairs and their associated intensity samples.

3.  **Compute Base Distance Threshold $\theta_{base}$:**
    Calculate the minimum Bhattacharyya distance between the two materials in any initial interface:
    $$ \theta_{base} = \min_{i \in \{1, \dots, k\}} d_B(V_{i,1}, V_{i,2}) $$
    where $d_B$ is the Bhattacharyya distance between Gaussian distributions [S1, unofficial].

4.  **Define Search Range:**
    Choose two multipliers $\epsilon \ll 1$ and $\zeta \gg 1$. The search threshold $\theta$ is chosen from the range $[\epsilon \theta_{base}, \zeta \theta_{base}]$ [S1, unofficial].

5.  **Greedy Merging for a Given $\theta$:**
    For a specific threshold $\theta$, deduce a feasible graph in two steps:
    *   **Find Materials to Merge:** Walk through the distance matrix $D$ to find similar materials. Create sets of materials that are similar to each other. A set may include a material from an interface, but not both materials of that interface (to avoid collapsing an interface to a single material) [S1, unofficial]. Reduce each set to a single material using a parent-child hierarchy, merging all children with their respective parents [S1, unofficial]. This results in the set of materials $M$.
    *   **Reconstruct Graph:** Merge nodes of similar materials together and merge repeating interface edges into one interface edge [S1, unofficial]. Since each material intensity is a distribution function, the resulting material intensities for merged materials are computed by a weighted average of the constituent material distributions (means and standard deviations) [S1, unofficial].

6.  **Search for Optimal $\theta$:**
    Search for the optimal $\theta$ that maximizes the separation energy $E_{sep}$:
    $$ E_{sep}(G(M, I)) = \frac{1}{|I|} \sum_{j=1}^{|I|} d_B(V_{j,1}, V_{j,2}) $$
    Maximizing $E_{sep}$ results in a graph with fewer but well-separated interface edges [S1, unofficial]. The optimal $\theta$ is the one yielding the maximum $E_{sep}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from hdbscan import HDBSCAN

def bhattacharyya_distance(mean1, std1, mean2, std2):
    """
    Computes Bhattacharyya distance between two 1D Gaussian distributions.
    """
    # Formula for Bhattacharyya distance between two Gaussians
    # D_B = 0.25 * ln(0.5 * (std1^2/std2^2 + std2^2/std1^2 + 2)) 
    #       + 0.25 * ((mean1 - mean2)^2 / (std1^2 + std2^2))
    var1 = std1 ** 2
    var2 = std2 ** 2
    term1 = 0.25 * np.log(0.5 * (var1 / var2 + var2 / var1 + 2))
    term2 = 0.25 * ((mean1 - mean2) ** 2) / (var1 + var2)
    return term1 + term2

def weighted_average_gaussian(dist1, dist2, weight1, weight2):
    """
    Computes the weighted average of two Gaussian distributions.
    dist = (mean, std)
    """
    mean_new = (weight1 * dist1[0] + weight2 * dist2[0]) / (weight1 + weight2)
    # Variance of sum of independent variables is sum of variances weighted by squared weights?
    # Actually, for mixture or merging, the source says "weighted average of constituent material distributions".
    # Assuming simple weighted average of parameters for the graph node representation:
    std_new = (weight1 * dist1[1] + weight2 * dist2[1]) / (weight1 + weight2)
    return (mean_new, std_new)

def robust_material_graph_deduction(volume_data):
    # Step 1: Compute Initial Edge Set I_hat
    # For each edge pixel, traverse along gradient to find L and H
    # This requires gradient computation and neighborhood search
    # [G] Assuming edge pixels are identified by high gradient magnitude
    edges = []
    # Placeholder for actual edge detection and L/H sampling
    # L, H = sample_LH(volume_data, position, gradient)
    # edges.append((L, H))
    
    # Step 2: Cluster Interfaces
    # Subsample to keep tractable
    subsampled_edges = subsample(edges, rate=0.1) # [G] Arbitrary subsampling rate
    # Cluster using HDBSCAN
    # HDBSCAN clusters the (L, H) pairs
    clusterer = HDBSCAN(min_cluster_size=10)
    labels = clusterer.fit_predict(subsampled_edges)
    
    # Extract initial material pairs from clusters
    # Each cluster represents an interface between two materials
    initial_interfaces = []
    for label in np.unique(labels):
        cluster_samples = subsampled_edges[labels == label]
        # Assume each cluster defines a pair of materials based on L and H values
        # [G] Logic to group L and H into distinct material distributions per cluster
        # This is complex; simplified: assume cluster defines one interface type
        initial_interfaces.append(cluster_samples)
        
    # Step 3: Compute Base Distance Threshold
    # Calculate Bhattacharyya distance for each initial interface
    distances = []
    for interface_samples in initial_interfaces:
        # Split samples into L and H groups
        L_vals = interface_samples[:, 0]
        H_vals = interface_samples[:, 1]
        
        # Fit Gaussians to L and H values
        mu_L = np.mean(L_vals)
        sigma_L = np.std(L_vals)
        mu_H = np.mean(H_vals)
        sigma_H = np.std(H_vals)
        
        dist = bhattacharyya_distance(mu_L, sigma_L, mu_H, sigma_H)
        distances.append(dist)
        
    theta_base = np.min(distances)
    
    # Step 4: Define Search Range
    epsilon = 0.1 # << 1
    zeta = 10.0   # >> 1
    theta_range = np.logspace(np.log10(epsilon * theta_base), np.log10(zeta * theta_base), 50)
    
    best_theta = None
    max_e_sep = -1
    best_graph = None
    
    # Step 5 & 6: Search for Optimal Theta
    for theta in theta_range:
        # Greedy Merging
        # 1. Find materials to merge
        # Create distance matrix D between all initial material distributions
        # [G] Construct D from initial_interfaces
        D = np.zeros((len(initial_interfaces), len(initial_interfaces)))
        for i in range(len(initial_interfaces)):
            for j in range(i + 1, len(initial_interfaces)):
                # Compute distance between interface i and interface j?
                # No, D is between materials.
                # Initial interfaces define pairs of materials.
                # We need to identify unique materials first.
                pass
        
        # Simplified Merging Logic based on source description:
        # Walk through D to find similar materials (distance < theta)
        # Create sets of similar materials, ensuring no set contains both materials of a single interface
        # Reduce sets to single materials via parent-child hierarchy
        
        # Reconstruct Graph
        # Merge nodes and repeating edges
        # Compute resulting material distributions via weighted average
        
        # Calculate Separation Energy
        # E_sep = (1/|I|) * sum(d_B(V_j1, V_j2)) for all interfaces j in G
        # [G] Calculate E_sep for the current graph configuration
        e_sep = 0.0 # Placeholder
        if e_sep > max_e_sep:
            max_e_sep = e_sep
            best_theta = theta
            best_graph = current_graph
            
    return best_graph, best_theta
```

**Parameters:**
*   $\epsilon \ll 1$: Lower bound multiplier for threshold search (e.g., 0.1) [S1, unofficial].
*   $\zeta \gg 1$: Upper bound multiplier for threshold search (e.g., 10.0) [S1, unofficial].
*   Subsampling rate: To keep algorithm tractable for large volumes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise:** Material boundaries detected by clustering may result in repeating boundaries of the same interface due to noise [S1, unofficial]. The greedy merging and optimal $\theta$ search help mitigate this by merging similar materials and interfaces.
*   **Connectivity:** A given interface only conveys adjacency; explicit connectivity is not known initially [S1, unofficial]. The graph deduction algorithm infers connectivity.
*   **Background:** The background material is set to invisible (white node) to hide it from rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm involves clustering (HDBSCAN) and a greedy merging process over a distance matrix. The search for optimal $\theta$ iterates over a range of values, recomputing the graph and separation energy. The source provides running times: Interface clustering takes ~2 seconds, Graph deduction takes ~4 seconds, Total ~6 seconds for the tested datasets [S1, unofficial].
*   **Memory Complexity:** Depends on the size of the volume and the number of initial edges/interfaces. Subsampling is used to keep it tractable [S1, unofficial].
*   **Strengths:** Automatically discovers material interiors and interconnectivity; provides fine control over material color and opacities; achieves automatic background removal [S1, unofficial].
*   **Limitations:** Relies on the quality of the initial edge set and clustering; performance may degrade with high noise or complex material distributions not well-separated by intensity [S1, unofficial].
*   **When to use:** When automatic transfer function generation is needed for volumes with distinct intensity-based materials and spatial interconnectivity is important [S1, unofficial].
*   **When not to use:** When materials have overlapping intensity distributions that cannot be separated by the Bhattacharyya distance metric, or when real-time performance is critical and the preprocessing time is prohibitive [S1, unofficial].

## Background
The method was proposed in 2018 by the authors of "Robust Material Graphs for Volume Rendering" [S1, unofficial]. It addresses the limitation of previous methods that focused only on material boundaries, neglecting the interior of materials [S1, unofficial]. The approach is implemented in a real-time GPU volume renderer written in C++ with a Python backend, using datasets from The Volume Library [S1, unofficial].

## Sources
[S1] Robust Material Graphs for Volume Rendering, https://doi.org/10.2312/pg20181282, fetched 2026-11-04 (unofficial)
