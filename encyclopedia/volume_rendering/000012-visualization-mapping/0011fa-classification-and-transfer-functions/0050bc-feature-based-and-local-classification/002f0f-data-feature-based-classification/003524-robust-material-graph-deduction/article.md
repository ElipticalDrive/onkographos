# Robust Material Graph Deduction

Also called: Graph-based TF Creation, Material Graph Generation

## Overview
Robust Material Graph Deduction is an automated method for constructing transfer functions by discovering the topological structure of materials within a volume [S1, unofficial]. It solves the problem of manual transfer function creation by automatically identifying material interiors and their spatial interconnectivity, producing a graph where nodes represent materials and edges represent interfaces [S1, unofficial]. Unlike sibling methods that focus on local feature extraction or high-dimensional clustering, this approach explicitly models the adjacency relationships between materials to enable automatic background removal and fine control over opacity based on depth [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset $V$ with scalar intensity values.
*   **Output:** A material graph $G(M, I)$, where $M$ is the set of discovered materials and $I$ is the set of interfaces between adjacent materials. Each material $M_k$ is defined by a Gaussian intensity distribution $V_k \sim \mathcal{N}(\mu_k, \sigma_k)$ and a mean occlusion value $\eta_k$ [S1, unofficial].

**Data Structures**
*   **Initial Edge Set $\hat{I}$:** A collection of interface samples. Each sample is a pair of intensity values $(L, H)$ representing the low and high sides of an interface.
*   **Distance Matrix $D$:** A symmetric matrix where $D_{ij}$ is the Bhattacharyya distance between material distributions $V_i$ and $V_j$.
*   **Material Graph $G$:** A graph structure with nodes $M$ and edges $I$.

**Algorithm Steps**

1.  **Compute Initial Edge Set $\hat{I}$:**
    For each edge pixel in the volume, perform a bidirectional traversal along the local gradient to find a low value $L$ and a high value $H$ in a small neighborhood [S1, unofficial]. These values are samples from adjacent intensity distributions $V_{i,1}$ and $V_{i,2}$ [S1, unofficial].

2.  **Cluster Interfaces:**
    Cluster the $(L, H)$ samples from $\hat{I}$ using the HDBSCAN algorithm to define distinct material interfaces [S1, unofficial]. Subsample the interface set from their distribution to maintain computational tractability for large volumes [S1, unofficial].

3.  **Define Base Threshold $\theta_{base}$:**
    Calculate the minimum Bhattacharyya distance between any two materials in the initial interfaces:
    $$ \theta_{base} = \min_{i \in \{1, \dots, k\}} d_B(V_{i,1}, V_{i,2}) $$
    where $d_B$ is the Bhattacharyya distance [S1, unofficial].

4.  **Search for Optimal Threshold $\theta$:**
    Define a search range $[\epsilon \theta_{base}, \zeta \theta_{base}]$ where $\epsilon \ll 1$ and $\zeta \gg 1$ [S1, unofficial]. For each candidate $\theta$, perform greedy merging to construct a graph and calculate the separation energy $E_{sep}$:
    $$ E_{sep}(G(M, I)) = \frac{1}{|I|} \sum_{j=1}^{|I|} d_B(V_{j,1}, V_{j,2}) $$
    Select the $\theta$ that maximizes $E_{sep}$ [S1, unofficial]. Maximizing this energy results in a graph with fewer, well-separated interface edges [S1, unofficial].

5.  **Greedy Merging (for a fixed $\theta$):**
    *   **Find Materials to Merge:** Walk through the distance matrix $D$ to create sets of similar materials. A set may include a material from an interface, but not both materials of that interface (to avoid collapsing an interface into a single material) [S1, unofficial]. Reduce each set to a single material using a parent-child hierarchy, merging all children with their respective parents [S1, unofficial].
    *   **Reconstruct Graph:** Merge nodes of similar materials and merge repeating interface edges into a single interface edge [S1, unofficial].
    *   **Update Distributions:** Compute the resulting material intensity distributions as a weighted average of the constituent material distributions (means and standard deviations) [S1, unofficial].

6.  **Assign Occlusion:**
    Calculate the mean occlusion value $\eta$ for each material, indicating its depth inside the volume, which is used to assign transparency values [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def robust_material_graph_deduction(volume):
    # Step 1: Compute initial edge set
    initial_edges = []
    for pixel in volume.edge_pixels():
        L, H = bidirectional_gradient_traversal(pixel, volume)
        initial_edges.append((L, H))
    
    # Step 2: Cluster interfaces
    # Subsample for tractability
    sampled_edges = subsample(initial_edges)
    clusters = hdbscan_cluster(sampled_edges)
    
    # Initialize materials from clusters
    materials = [create_material(cluster) for cluster in clusters]
    
    # Step 3: Compute base threshold
    distances = compute_bhattacharyya_matrix(materials)
    theta_base = min(distances[i][j] for i, j in interface_pairs(materials))
    
    # Step 4: Search for optimal theta
    epsilon = 0.1  # Example value << 1
    zeta = 10.0    # Example value >> 1
    best_theta = theta_base
    max_energy = -infinity
    
    for theta in log_space(epsilon * theta_base, zeta * theta_base, steps=50):
        # Step 5: Greedy merging
        merged_materials, merged_interfaces = greedy_merge(materials, distances, theta)
        
        # Calculate separation energy
        if len(merged_interfaces) == 0:
            continue
        energy = sum(bhattacharyya(m1, m2) for m1, m2 in merged_interfaces) / len(merged_interfaces)
        
        if energy > max_energy:
            max_energy = energy
            best_theta = theta
            best_graph = (merged_materials, merged_interfaces)
            
    # Step 6: Finalize graph
    final_materials, final_interfaces = greedy_merge(materials, distances, best_theta)
    
    # Assign occlusion values
    for material in final_materials:
        material.occlusion = compute_mean_occlusion(material, volume)
        
    return Graph(final_materials, final_interfaces)

def bidirectional_gradient_traversal(pixel, volume):
    # Traverse along gradient to find local min (L) and max (H)
    grad = volume.gradient(pixel)
    low_val = volume[pixel]
    high_val = volume[pixel]
    
    # Move in negative gradient direction
    curr = pixel
    for _ in range(num_steps):
        curr = curr - step_size * grad
        low_val = min(low_val, volume[curr])
        
    # Move in positive gradient direction
    curr = pixel
    for _ in range(num_steps):
        curr = curr + step_size * grad
        high_val = max(high_val, volume[curr])
        
    return low_val, high_val

def bhattacharyya(v1, v2):
    # Bhattacharyya distance between two Gaussians N(mu1, sigma1) and N(mu2, sigma2)
    mu1, sigma1 = v1.mean, v1.std
    mu2, sigma2 = v2.mean, v2.std
    avg_sigma = (sigma1 + sigma2) / 2
    avg_mu = (mu1 + mu2) / 2
    dist_sq = (mu1 - mu2)**2 / (2 * avg_sigma**2)
    coeff = 0.25 * log(avg_sigma**2 / (sigma1 * sigma2))
    return dist_sq + coeff

def greedy_merge(materials, distances, theta):
    # Create parent-child hierarchy
    parent = {m: m for m in materials}
    
    # Merge similar materials
    for i in range(len(materials)):
        for j in range(i + 1, len(materials)):
            if distances[i][j] < theta:
                # Check if merging would collapse an interface
                if not interface_collapses(materials[i], materials[j], materials):
                    parent[materials[j]] = materials[i]
                    
    # Reduce sets to single materials
    unique_materials = set()
    for m in materials:
        root = find_root(m, parent)
        unique_materials.add(root)
        
    # Reconstruct interfaces
    interfaces = []
    for m1 in unique_materials:
        for m2 in unique_materials:
            if m1 != m2 and are_adjacent(m1, m2, materials):
                interfaces.append((m1, m2))
                
    return list(unique_materials), interfaces

def find_root(node, parent):
    while parent[node] != node:
        node = parent[node]
    return node
```

**Parameters**
*   $\epsilon$: Lower bound multiplier for threshold search. Must be much less than 1 ($\epsilon \ll 1$) [S1, unofficial].
*   $\zeta$: Upper bound multiplier for threshold search. Must be much greater than 1 ($\zeta \gg 1$) [S1, unofficial].
*   $\theta$: Distance threshold for merging materials. Searched in $[\epsilon \theta_{base}, \zeta \theta_{base}]$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Noise:** Noise in volume data can result in repeating boundaries for the same interface. The clustering and merging steps help mitigate this, but subsampling is critical for performance [S1, unofficial].
*   **Disconnected Components:** Materials may have multiple disconnected components. The graph deduction assumes valid material-material interfaces from clustering [S1, unofficial].
*   **Background:** The background material is typically set to invisible (white node) to hide it from rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the initial edge detection and the HDBSCAN clustering. The greedy merging is efficient, but the search for optimal $\theta$ requires evaluating multiple graph configurations [S1, unofficial].
*   **Memory Complexity:** Requires storing the initial edge set, distance matrix, and the final graph. Subsampling helps manage memory for large volumes [S1, unofficial].
*   **Performance:** Implemented in a real-time GPU volume renderer with a Python backend. Running times for graph deduction are reported in seconds for typical datasets (e.g., 2-4 seconds for graph deduction on standard datasets) [S1, unofficial].
*   **Strengths:** Automatic discovery of materials and interfaces; fine control over color and opacity; automatic background removal; robust to noise via clustering [S1, unofficial].
*   **Limitations:** Relies on the quality of the initial edge detection; assumes materials can be modeled by Gaussian distributions; may struggle with highly complex or overlapping intensity distributions [S1, unofficial].

## Background
The method was proposed to address the limitation of existing techniques that focus on material boundaries without adequately representing material interiors [S1, unofficial]. It builds on the LH histogram approach for edge detection [S1, unofficial]. The algorithm was implemented and tested on datasets from The Volume Library [S1, unofficial].

## Sources
[S1] Robust Material Graphs for Volume Rendering, https://doi.org/10.2312/pg20181282, fetched 2026-11-04 (unofficial)
