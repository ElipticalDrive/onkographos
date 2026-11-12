# Dynamic Growth Parameters
Also called: NeRF Growth Steps, Recursive NeRF Growth

## Overview
Dynamic Growth Parameters define the configuration for the iterative expansion of a neural radiance field's network architecture during training. This method solves the problem of representing complex scenes with a single static network by dynamically adding capacity where the current model exhibits high uncertainty. It sets apart from static NeRF variants by coupling the network topology directly to the training error, growing $k$ new branches per step based on k-means clustering of high-uncertainty query points [S1].

## How it works
**Inputs and Outputs:**
- Input: Current trained network state, set of sampled 3D points with associated uncertainty values $\delta_i$, and viewing directions $d$.
- Output: Updated network topology with new child branches and their initial parameters.

**Data Structures:**
- **Network Branch:** A neural field $F_{\Phi}$ taking parent output $y_{p}$ and direction $d$ to predict color $c$, density $\sigma$, uncertainty $\delta$, and latent vector $y$.
- **Cluster Centers:** $k$ centroids derived from k-means clustering of high-uncertainty points.

**Parameters:**
- $k$: Number of branches to grow per step. Default value is 2 [S1]. Range is $[2, 4]$ [S1].
- $I_1$: Iteration count after which the first growth evaluation occurs [S1].
- $N_{steps}$: Total number of growth steps. Default is 3 [S1].

**Steps:**
1. **Train:** Train the current network for $I_1$ iterations.
2. **Evaluate Uncertainty:** Compute uncertainty $\delta_i$ for sampled points.
3. **Cluster:** Identify points with $\delta_i$ above a threshold. Apply k-means clustering with parameter $k$ to these points to determine $k$ cluster centers [S1].
4. **Grow:** Create $k$ new child branches. Initialize their parameters based on the cluster centers [S1].
5. **Repeat:** If the current step count $< N_{steps}$, return to Step 1 with the expanded network. Otherwise, terminate [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_growth_parameters(network, points, directions, k=2, max_steps=3):
    """
    Implements the dynamic growth parameter configuration for Recursive NeRF.
    """
    current_network = network
    for step in range(max_steps):
        # Train current network for a fixed number of iterations
        train(current_network, points, directions, iterations=I_1)
        
        # Evaluate uncertainty for all points
        uncertainties = compute_uncertainty(current_network, points, directions)
        
        # Filter high uncertainty points
        high_uncertainty_points = filter_high_uncertainty(points, uncertainties, threshold=THRESHOLD)
        
        # Check if growth is needed
        if len(high_uncertainty_points) == 0:
            break
            
        # Cluster high uncertainty points to find branch centers
        cluster_centers = kmeans(high_uncertainty_points, k=k)
        
        # Grow new branches based on cluster centers
        new_branches = grow_branches(current_network, cluster_centers, k=k)
        
        # Update network with new branches
        current_network.add_branches(new_branches)
        
    return current_network
```

**Edge Cases:**
- If no points exceed the uncertainty threshold, growth terminates early [S1].
- Precision issues in clustering may arise if high-uncertainty points are sparse [G].

## Subtopics

### Dynamic Growth
An adaptive strategy that clusters uncertain queries at the current stage and grows deeper networks according to the clustering result. After $I_1$ iterations, points with uncertainty higher than a threshold are clustered using k-means with $k \in [2, 4]$. The network grows $k$ branches according to the cluster centers [S1]. This variant emphasizes the recursive nature of the field, where each branch refines the prediction for a specific spatial region identified by the clustering [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Time: Increases with each growth step due to additional training iterations and the evaluation of uncertainty across the scene. The total training time is bounded by $N_{steps}$ [S1].
- Memory: Scales linearly with the number of branches. Each growth step adds $k$ new network branches [S1].

**Measured Performance:**
- Recursive NeRF grows 3 times in total [S1].
- Default $k=2$ for each growth step [S1].
- Early termination allows the network to finalize predictions when uncertainty is low, speeding up rendering [S1].

**Strengths and Limitations:**
- Strengths: Efficiently allocates network capacity to complex regions; avoids unnecessary calculations in low-uncertainty areas [S1].
- Limitations: Requires careful tuning of $k$ and the uncertainty threshold; training time increases with the number of growth steps [S1].

## Background
Originated in Recursive-NeRF [S1]. The method builds on the principle that a single static network may not capture all scene details, necessitating a dynamic, recursive structure that grows based on training error [S1].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
