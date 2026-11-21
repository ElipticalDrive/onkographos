# Dynamic Growth Parameters

Also called: NeRF Growth Steps, Recursive NeRF Growth

## Overview
Dynamic Growth Parameters define the configuration for the iterative expansion of a neural radiance field's network architecture during training. This method solves the problem of representing complex scenes with a single static network by dynamically allocating capacity where reconstruction uncertainty is high. It sets apart from static NeRF variants by using k-means clustering of uncertain query points to spawn new network branches, and from other growth methods by strictly limiting the total number of growth iterations and the branching factor $k$ per step [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current network parameters $\Phi$, training dataset, current iteration count $I$, total allowed growth steps $N_{grow}$, and branching factor $k$.
*   **Outputs:** Updated network architecture with new child branches, updated parameters.

**Data Structures:**
*   **Network Branch:** A multi-layer perceptron (MLP) taking parent output $y_{p_i}$ and viewing direction $d$ as input, predicting color $c_i$, density $\sigma_i$, uncertainty $\delta_i$, and latent vector $y_i$ [S1].
*   **Cluster Centers:** $k$ vectors representing the mean latent/position space of high-uncertainty points.

**Numbered Steps:**
1.  **Training Phase:** Train the current network branch for a fixed number of iterations $I_1$ [S1].
2.  **Uncertainty Estimation:** Compute the uncertainty metric $\delta_i$ for all sampled query points in the scene [S1].
3.  **Clustering:** Identify points where $\delta_i$ exceeds a predefined threshold. Apply k-means clustering with parameter $k$ to these high-uncertainty points to determine $k$ cluster centers [S1].
4.  **Branch Initialization:** Initialize $k$ new child network branches. Each branch is initialized based on the corresponding cluster center from the previous step [S1].
5.  **Growth Check:** If the total number of growth steps performed is less than $N_{grow}$, repeat from Step 1 with the expanded network. Otherwise, stop growth [S1].

**Parameters:**
*   $N_{grow}$: Total number of growth iterations. Default value is 3 [S1].
*   $k$: Number of branches spawned per growth step. Default value is 2 [S1]. Range is $[2, 4]$ [S1].
*   $I_1$: Iterations per growth stage. Not given in the sources.
*   Threshold: Uncertainty threshold for clustering. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Clustering relies on the stability of the uncertainty metric $\delta_i$. If $\delta_i$ is noisy, clustering may produce unstable branch centers [S1].
*   **Boundaries:** The method assumes the scene can be effectively segmented by k-means in the latent/position space. Degenerate cases where uncertainty is uniformly low may result in no growth, while uniformly high uncertainty may lead to excessive branching if $k$ is not capped [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_growth_parameters(network, dataset, N_grow=3, k=2, I_1=1000):
    """
    Implements the dynamic growth parameter logic for Recursive NeRF.
    """
    current_network = network
    growth_step = 0
    
    while growth_step < N_grow:
        # Step 1: Train current network for I_1 iterations
        current_network = train_network(current_network, dataset, iterations=I_1)
        
        # Step 2: Compute uncertainty for all query points
        # Assume get_uncertainties returns a list of (point, delta) tuples
        points_and_uncertainties = get_uncertainties(current_network, dataset)
        
        # Step 3: Cluster high uncertainty points
        # Filter points above threshold (threshold not specified in sources, assume fixed or adaptive)
        high_uncertainty_points = [p for p, delta in points_and_uncertainties if delta > THRESHOLD]
        
        if len(high_uncertainty_points) == 0:
            break
            
        # Perform k-means clustering on the points
        # Returns k cluster centers
        cluster_centers = k_means(high_uncertainty_points, k=k)
        
        # Step 4: Initialize k new child branches
        new_branches = []
        for center in cluster_centers:
            # Initialize new branch based on cluster center
            # The exact initialization logic (e.g., copying weights or zero-init) is not specified
            new_branch = initialize_branch(center)
            new_branches.append(new_branch)
            
        # Attach new branches to the current network structure
        current_network.add_children(new_branches)
        
        growth_step += 1
        
    return current_network
```

## Subtopics

### Dynamic Growth
An adaptive strategy that clusters uncertain queries at the current stage and grows deeper networks according to the clustering result. After $I_1$ iterations, points with uncertainty higher than a threshold are clustered using k-means with $k \in [2, 4]$. The network grows $k$ branches according to the cluster centers [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Increases linearly with the number of growth steps and $k$. Each growth step adds $k$ new network branches [S1].
*   **Time:** Training time increases due to the iterative clustering and re-training phases. The total number of growth steps is limited to 3 by default, bounding the complexity [S1].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   The method scales with scene complexity by allocating more network capacity to high-uncertainty regions. The fixed limit of 3 growth steps prevents unbounded growth [S1].

**Strengths and Limitations:**
*   **Strengths:** Efficiently represents complex scenes by focusing capacity where needed. Early termination allows speeding up rendering for low-uncertainty points [S1].
*   **Limitations:** Requires careful tuning of $k$ and the uncertainty threshold. The recursive nature may complicate implementation compared to single-branch NeRFs [S1].

**When to use it and when not:**
*   Use when representing scenes with varying levels of detail and when computational resources allow for iterative training. Not suitable for real-time applications requiring a fixed, pre-trained model without growth overhead [S1].

## Background
**Origin:**
*   Recursive NeRF was introduced in the paper "Recursive-NeRF: An Efficient and Dynamically Growing NeRF" [S1].

**History and Development:**
*   The method was developed to address the limitations of single-branch NeRFs in representing complex scenes. It introduces a recursive structure where the network grows dynamically based on uncertainty metrics [S1].

**Where it is used:**
*   Used in neural radiance field optimization for view synthesis and 3D reconstruction [S1].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
