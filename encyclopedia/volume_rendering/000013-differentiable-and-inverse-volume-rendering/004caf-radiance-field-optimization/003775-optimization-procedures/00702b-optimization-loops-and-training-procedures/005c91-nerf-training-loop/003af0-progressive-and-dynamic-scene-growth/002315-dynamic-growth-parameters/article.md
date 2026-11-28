# Dynamic Growth Parameters

Also called: NeRF Growth Steps, Recursive NeRF Growth

## Overview
Dynamic Growth Parameters configure the iterative expansion of a neural radiance field's network architecture during training [S1]. It sets apart from static NeRF variants by using k-means clustering of uncertain query points to spawn new network branches, and from other growth methods by strictly limiting the total number of growth iterations and the branching factor $k$ per step [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current network branch, training dataset, current iteration count, total allowed growth steps $N_{grow}$, branching factor $k$, iterations per stage $I_1$, and an uncertainty threshold [S1].
*   **Outputs:** Updated network architecture with new child branches and optimized parameters [S1].

**Data Structures:**
*   **Network Branch:** A multi-layer perceptron defined as $F_{\Phi_i} : (y_{p_i}, d) \rightarrow (c_i, \sigma_i, \delta_i, y_i)$, taking parent output $y_{p_i}$ and viewing direction $d$ as input, and predicting color $c_i$, density $\sigma_i$, uncertainty $\delta_i$, and latent vector $y_i$ [S1].
*   **Cluster Centers:** $k$ vectors representing the mean position/latent space of high-uncertainty points [S1].

**Numbered Steps:**
1.  **Training Phase:** Train the current network branch for a fixed number of iterations $I_1$ [S1].
2.  **Uncertainty Estimation:** Compute the uncertainty metric $\delta_i$ for all sampled query points in the scene [S1].
3.  **Clustering:** Identify points where $\delta_i$ exceeds a predefined threshold and cluster them using k-means with parameter $k \in [2, 4]$ [S1].
4.  **Branch Initialization:** Grow $k$ new child network branches initialized at the corresponding cluster centers [S1].
5.  **Growth Check:** If the total number of growth steps performed is less than $N_{grow}$, repeat from Step 1 with the expanded network [S1].

**Parameters:**
*   $N_{grow}$: Total number of growth iterations. Default value is 3 [S1].
*   $k$: Number of branches spawned per growth step. Default value is 2, with a range of $[2, 4]$ [S1].
*   $I_1$: Iterations per growth stage. Not given in the sources.
*   Threshold: Uncertainty threshold for clustering. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Early Termination:** The method uses early termination to finalize predictions when uncertainty is low enough, avoiding unnecessary calculations and speeding up rendering [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def dynamic_growth_parameters(network, dataset, N_grow=3, k=2, I_1=1000, threshold=0.0):
    current_network = network
    growth_step = 0
    while growth_step < N_grow:
        # Step 1: Train current network for I_1 iterations
        current_network = train_step(current_network, dataset, I_1)
        
        # Step 2: Compute uncertainty for sampled query points
        points = sample_points(dataset)
        uncertainties = compute_uncertainties(current_network, points)
        
        # Step 3: Cluster high uncertainty points
        high_unc_points = [p for p, u in zip(points, uncertainties) if u > threshold]
        if not high_unc_points:
            break
        centers = k_means(high_unc_points, k)
        
        # Step 4: Initialize and attach k new child branches
        for center in centers:
            child = init_child_branch(center)
            current_network.add_child(child)
            
        growth_step += 1
    return current_network

def train_step(net, data, iters):
    for _ in range(iters):
        batch = sample_batch(data)
        net.update_gradients(batch)
    return net

def sample_points(data):
    return data.points

def compute_uncertainties(net, points):
    return [net.predict(p).uncertainty for p in points]

def k_means(points, k):
    centers = random.sample(points, k)
    for _ in range(10):
        assignments = [min(range(k), key=lambda c: math.dist(p, centers[c])) for p in points]
        new_centers = [sum(points[i] for i, a in enumerate(assignments) if a == idx) / max(1, sum(1 for a in assignments if a == idx)) for idx in range(k)]
        if new_centers == centers: break
        centers = new_centers
    return centers

def init_child_branch(center):
    # Initialize new branch network parameters based on cluster center
    # The exact initialization logic is not specified in the sources [G]
    return NetworkBranch(initial_state=center)
```

## Subtopics

### Dynamic Growth
An adaptive strategy that clusters uncertain queries at the current stage and grows deeper networks according to the clustering result [S1]. After $I_1$ iterations, points with uncertainty higher than a threshold are clustered using k-means with $k \in [2, 4]$ [S1]. The network grows $k$ branches according to the cluster centers [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Increases linearly with the number of growth steps and $k$, as each step adds $k$ new network branches [S1].
*   **Time:** Training time increases due to iterative clustering and re-training phases, but is bounded by the default limit of 3 growth steps [S1].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   The method scales with scene complexity by allocating capacity to high-uncertainty regions, while early termination speeds up rendering for low-uncertainty points [S1].

**Strengths and Limitations:**
*   **Strengths:** Efficiently represents complex scenes by focusing capacity where needed and accelerates rendering via early termination [S1].
*   **Limitations:** Introduces overhead from iterative clustering and re-training phases [S1].

**When to use it and when not:**
*   Use when representing scenes with varying levels of detail and when iterative training is feasible [S1].

## Background
**Origin:**
*   Recursive NeRF was introduced in "Recursive-NeRF: An Efficient and Dynamically Growing NeRF" [S1].

**History and Development:**
*   The method was developed to address the limitations of single-branch NeRFs by introducing a recursive structure that grows dynamically based on uncertainty metrics [S1].

**Where it is used:**
*   It is used in neural radiance field optimization for view synthesis and 3D reconstruction [S1].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
