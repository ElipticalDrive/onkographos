# 3D-to-4D Voxel Expansion Algorithm

## Overview
The 3D-to-4D Voxel Expansion Algorithm converts a static 3D occupancy grid into a 4D object-centric representation by clustering voxels based on geometry, color, and dynamics [S1, unofficial]. It solves the problem of separating mixed volumetric data into distinct object slots, assuming voxels belonging to the same object share similar motion and appearance features [S1, unofficial]. Unlike [Point Wisp Voxelization](10591-point-wisp-voxelization.md) which relies on point cloud structures, or [Virtual Smoke](17061-virtual-smoke.md) which models fluid dynamics explicitly, this method uses a graph-based connected components approach on pre-filtered voxel features to achieve unsupervised object-centric decomposition [S1, unofficial].

## How it works
**Inputs:**
*   $X = \{X_k\}_{k=1}^{N_s}$: Density values for all voxels.
*   $x = \{x_k\}_{k=1}^{N_s}$: 3D positions for all voxels.
*   $N_s = N_x \times N_y \times N_z$: Total number of voxels.
*   Hyperparameters: $\delta_{den}$ (density threshold), $\delta_{rgb}$ (color distance threshold), $\delta_{vel}$ (velocity distance threshold), $N_r$ (number of sampling rays), $N$ (assumed maximum number of objects/slots).
*   Models: $N_{\phi'}$ (neural network for emitted color), $f_{\xi'}$ (neural network for forward canonical-space transitions/dynamics).

**Outputs:**
*   $V_{t=1}^T$: 4D voxel grid where each voxel is assigned to a cluster (object slot).

**Data Structures:**
*   Valid voxel index set $\mathcal{N} = \{k \mid X_k > \delta_{den}, k \in [1, N_s]\}$.
*   Feature Graph $G = (\mathcal{N}, E)$, where nodes are valid voxels and edges $E$ represent similarity.
*   Cluster set $\{cl_k\}_{k=1}^M$ from connected components.

**Steps:**
1.  **Filter Voxels:** Identify valid voxels $\mathcal{N}$ by comparing density $X_k$ against $\delta_{den}$ [S1, unofficial].
2.  **Estimate Color:** For each valid voxel $j \in \mathcal{N}$, sample $N_r$ random rays with directions $d_i$. Calculate emitted color $c_{i,j} = N_{\phi'}(x_j, d_i)$ for each ray. Compute the averaged color $\bar{c}_j = \frac{1}{N_r} \sum_{i=1}^{N_r} c_{i,j}$ [S1, unofficial].
3.  **Estimate Velocity:** For each time step $t$ from $1$ to $T-1$, calculate the velocity vector $v_{t,j} = f_{\xi'}(x_j, t+1) - f_{\xi'}(x_j, t)$ for each valid voxel $j \in \mathcal{N}$ [S1, unofficial].
4.  **Build Feature Graph:** For every pair of valid voxels $(p, q)$:
    *   Compute color distance: $D_{rgb}(p, q) = \| \bar{c}_p - \bar{c}_q \|_2$.
    *   Compute velocity distance: $D_{vel}(p, q) = \max_{t \in [1, T-1]} \| v_{t,p} - v_{t,q} \|_2$.
    *   If $p$ and $q$ are spatial neighbors AND $D_{rgb}(p, q) < \delta_{rgb}$ AND $D_{vel}(p, q) < \delta_{vel}$, set edge $E(p, q) = 1$. Otherwise, $E(p, q) = 0$ [S1, unofficial].
5.  **Cluster:** Run Connected Components on graph $G$ to obtain initial cluster set $\{cl_k\}_{k=1}^M$ [S1, unofficial].
6.  **Merge Clusters:** Calculate the number of smallest clusters to merge: $N_l = M - N + 1$. Sort clusters by size and merge the $N_l$ smallest clusters to ensure the total number of clusters does not exceed $N$ [S1, unofficial].
7.  **Assign Slots:** For each final cluster $k$ and each voxel $j$ in that cluster, assign the voxel to slot $k$ in the 4D grid $V$ using the density value $X_j$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def expand_3d_to_4d(X, x, Ns, delta_den, delta_rgb, delta_vel, Nr, N, N_phi_prime, f_xi_prime, T):
    # 1. Filter valid voxels
    valid_indices = [k for k in range(1, Ns + 1) if X[k] > delta_den]
    
    # 2. Estimate averaged color for valid voxels
    avg_colors = {}
    for j in valid_indices:
        color_sum = [0.0, 0.0, 0.0]
        for i in range(1, Nr + 1):
            direction = sample_random_direction() # [G] Implementation detail for random ray
            emitted_color = N_phi_prime(x[j], direction)
            color_sum = color_sum + emitted_color
        avg_colors[j] = color_sum / Nr
        
    # 3. Estimate velocity for valid voxels
    velocities = {} # velocities[j][t]
    for j in valid_indices:
        velocities[j] = []
        for t in range(1, T):
            pos_next = f_xi_prime(x[j], t + 1)
            pos_curr = f_xi_prime(x[j], t)
            vel = pos_next - pos_curr
            velocities[j].append(vel)
            
    # 4. Build graph
    edges = {}
    for p in valid_indices:
        for q in valid_indices:
            if p == q:
                continue
            # Check spatial neighbor status [G] Implementation detail for neighbor check
            if not is_neighbor(p, q):
                continue
                
            d_rgb = l2_norm(avg_colors[p] - avg_colors[q])
            
            # Max velocity distance over time
            max_d_vel = 0.0
            for t in range(len(velocities[p])):
                d_vel = l2_norm(velocities[p][t] - velocities[q][t])
                if d_vel > max_d_vel:
                    max_d_vel = d_vel
            
            if d_rgb < delta_rgb and max_d_vel < delta_vel:
                edges[(p, q)] = 1
            else:
                edges[(p, q)] = 0
                
    # 5. Connected Components
    clusters = connected_components(valid_indices, edges)
    M = len(clusters)
    
    # 6. Merge small clusters if M > N
    if M > N:
        N_l = M - N + 1
        sorted_clusters = sort_clusters_by_size(clusters)
        # Merge the N_l smallest clusters into the largest remaining ones
        # [G] Specific merge strategy (e.g., union-find) is not detailed in source
        final_clusters = merge_smallest_clusters(sorted_clusters, N_l)
    else:
        final_clusters = clusters
        
    # 7. Assign to 4D grid
    V = {}
    for k, cluster in enumerate(final_clusters):
        for j in cluster:
            V[(j, k)] = X[j]
            
    return V
```

**Parameters:**
*   $N_s$: Total voxels ($N_x \times N_y \times N_z$). Typical grid size $110^3$ [S1, unofficial].
*   $\delta_{den}$: Density threshold to filter invalid voxels. Value not given in the sources.
*   $\delta_{rgb}$: RGB distance threshold. Value not given in the sources.
*   $\delta_{vel}$: Velocity distance threshold. Value not given in the sources.
*   $N_r$: Number of sampling rays. Value not given in the sources.
*   $N$: Assumed maximum number of objects. Set to 10 in experiments [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If the number of connected components $M$ is less than $N$, no merging is performed, leaving empty slots [S1, unofficial].
*   Precision in $f_{\xi'}$ and $N_{\phi'}$ affects the accuracy of velocity and color features, directly impacting clustering quality [S1, unofficial].
*   Spatial neighbor definition must be consistent (e.g., 6-connectivity or 26-connectivity) to ensure graph connectivity reflects physical adjacency [G].

## Subtopics

### 4D Voxel Optimization
A subsequent stage that finetunes the dynamics module, volume slot attention, and compositional renderer to refine the object-centric voxel grids obtained from expansion [S1, unofficial].

### Object-Centric Voxelization
The broader goal of decomposing dynamic scenes into independent object representations, enabling unsupervised learning of scene dynamics [S1, unofficial].

## Operating characteristics
*   **Complexity:** The graph construction step involves comparing all pairs of valid voxels, resulting in $O(|\mathcal{N}|^2)$ pairwise comparisons, where $|\mathcal{N}|$ is the number of valid voxels. Connected components is nearly linear in the number of edges and nodes.
*   **Memory:** Requires storing features (color, velocity) for all valid voxels and the adjacency graph. The graph size can be large for dense grids.
*   **Performance:** The method relies on pre-trained neural networks ($N_{\phi'}, f_{\xi'}$) for feature extraction, which must be available before expansion [S1, unofficial].
*   **Strengths:** Unsupervised; separates objects based on appearance and motion; handles complex textures [S1, unofficial].
*   **Limitations:** Requires careful tuning of thresholds ($\delta_{rgb}, \delta_{vel}$); assumes voxels of the same object have similar features; sensitive to the accuracy of the underlying dynamic model [S1, unofficial].

## Background
The algorithm was introduced in "DynaVol: Unsupervised Learning for Dynamic Scenes through Object-Centric Voxelization" by Liu et al. (2024) [S1, unofficial]. It is part of a pipeline that includes a warmup stage and a 4D voxel optimization stage, designed for real-world dynamic scenes with complex textures [S1, unofficial]. The method builds on prior literature for hyperparameter adoption (Liu et al., 2022) [S1, unofficial].

## Sources
[S1] DynaVol: Unsupervised Learning for Dynamic Scenes through Object-Centric Voxelization, https://doi.org/10.48550/arxiv.2305.00393, fetched 2026-10-26 (unofficial)
