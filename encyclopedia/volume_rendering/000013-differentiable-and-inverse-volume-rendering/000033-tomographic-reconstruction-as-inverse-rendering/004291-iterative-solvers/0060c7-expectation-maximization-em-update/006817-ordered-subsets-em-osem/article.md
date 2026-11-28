# Ordered Subsets EM (OSEM)
Also called: MAP-OSEM, OSEM (Ordered Subset Expectation Maximisation), Ordered Subset Expectation Maximization (OSEM)

## Overview
Ordered Subsets EM (OSEM) is an accelerated variant of the Maximum Likelihood Expectation Maximization (MLEM) algorithm used for tomographic reconstruction [S1, S2, unofficial]. It reduces reconstruction time by decomposing each full iteration into $B$ sub-iterations, where each sub-iteration uses only a subset of the available projection data [S1, S2, unofficial]. Unlike standard EM, which processes all projections simultaneously, OSEM updates the image estimate sequentially for each subset, significantly speeding up convergence, though this acceleration can lead to earlier divergence, often necessitating regularization (MAP-OSEM) [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Initial image estimate $f^{(0)}$ (non-negative), projection data $p_i$, system matrix elements $R_{ij}$ (probability that photon from voxel $j$ is detected in bin $i$), and number of subsets $M$ [S2, unofficial].
- Output: Reconstructed image $f$ after convergence [S2, unofficial].

**Data Structures:**
- $f_j$: Value of voxel $j$.
- $p_i$: Measured projection value for bin $i$.
- $R_{ij}$: System matrix element.
- $s_m$: Set of projection indices belonging to subset $m$, where $m \in \{1, \dots, M\}$ [S2, unofficial].

**Parameters:**
- $M$: Number of subsets. Must be a positive even integer less than or equal to the total number of projections [S2, unofficial].
- $B$: Number of subsamples per step (equivalent to $M$ in the notation of [S2, unofficial]) [S1, unofficial].

**Algorithm Steps:**
1. Initialize $f_j^{(0)} > 0$ for all voxels $j$ [S2, unofficial].
2. For iteration $n = 0, 1, 2, \dots$:
   3. For each subset $m = 1, \dots, M$:
      4. Compute the forward projection of the current image estimate using only projections in subset $s_m$:
         $$ \hat{p}_i^{(n, m-1)} = \sum_{j} R_{ij} f_j^{(n, m-1)} \quad \forall i \in s_m $$
         [S2, unofficial]
      5. Compute the ratio of measured to estimated projections for the subset:
         $$ \text{ratio}_i = \frac{p_i}{\hat{p}_i^{(n, m-1)}} \quad \forall i \in s_m $$
         [S2, unofficial]
      6. Compute the backprojection of the ratio, normalized by the sum of system matrix elements for the subset:
         $$ \text{backproj}_j = \sum_{i \in s_m} R_{ij} \cdot \text{ratio}_i $$
         $$ \text{norm}_j = \sum_{i \in s_m} R_{ij} $$
         [S2, unofficial]
      7. Update the image estimate for the current sub-iteration:
         $$ f_j^{(n, m)} = f_j^{(n, m-1)} \cdot \frac{\text{backproj}_j}{\text{norm}_j} $$
         [S2, unofficial]
   8. Set the starting point for the next iteration:
      $$ f_j^{(n+1)} = f_j^{(n, M)} $$
      [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def osem_reconstruct(projections, system_matrix, num_subsets, max_iterations):
    """
    Perform Ordered Subsets EM reconstruction.
    
    Args:
        projections: List of measured projection values p_i.
        system_matrix: 2D array R_ij where i is projection bin and j is voxel.
        num_subsets: Integer M, number of subsets (positive even integer <= len(projections)).
        max_iterations: Integer, maximum number of full iterations.
        
    Returns:
        Reconstructed image array f.
    """
    num_projections = len(projections)
    num_voxels = len(system_matrix[0])
    
    # Initialize image estimate to positive values [G]
    # Common initialization is uniform positive value or backprojection of data
    image = [1.0] * num_voxels
    
    # Define subsets s_m
    # Source [S2, unofficial] states subsets should be ordered to minimize redundancy (e.g., separated by 90 degrees).
    # A simple round-robin assignment is a valid implementation strategy if not specified otherwise.
    subsets = [[] for _ in range(num_subsets)]
    for i in range(num_projections):
        subsets[i % num_subsets].append(i)
        
    for n in range(max_iterations):
        # Current image state for the iteration
        current_image = list(image)
        
        for m in range(num_subsets):
            subset_indices = subsets[m]
            
            # Step 4: Forward projection for subset
            estimated_projections = [0.0] * num_projections
            for i in subset_indices:
                proj_sum = 0.0
                for j in range(num_voxels):
                    proj_sum += system_matrix[i][j] * current_image[j]
                estimated_projections[i] = proj_sum
                
            # Step 5 & 6 & 7: Ratio, Backprojection, and Update
            # Precompute normalization factors for the subset
            norm_factors = [0.0] * num_voxels
            for j in range(num_voxels):
                norm_sum = 0.0
                for i in subset_indices:
                    norm_sum += system_matrix[i][j]
                norm_factors[j] = norm_sum
                
            # Update image
            new_image = [0.0] * num_voxels
            for j in range(num_voxels):
                backproj_sum = 0.0
                for i in subset_indices:
                    # Avoid division by zero if estimated projection is 0 [G]
                    if estimated_projections[i] > 0:
                        ratio = projections[i] / estimated_projections[i]
                    else:
                        ratio = 0.0
                    backproj_sum += system_matrix[i][j] * ratio
                
                if norm_factors[j] > 0:
                    new_image[j] = current_image[j] * (backproj_sum / norm_factors[j])
                else:
                    new_image[j] = current_image[j]
            
            current_image = new_image
            
        image = current_image
        
    return image
```

**Edge Cases and Pitfalls:**
- **Divergence:** The accelerated convergence of OSEM can lead to divergence problems occurring earlier than in standard MLEM [S2, unofficial]. This often requires the use of regularization, resulting in MAP-OSEM [S2, unofficial].
- **Subset Selection:** Projections should be grouped and ordered to minimize redundancy within subsets (e.g., separated by large angles like 90 degrees) for optimal efficiency [S2, unofficial].
- **Subsample Types:** Hudson and Larkin distinguish between non-overlapping subsamples, nested subsamples, and the standard sample (which is equivalent to standard EM) [S1, unofficial].
- **Single Subset:** If $B=1$ (or $M=1$), OSEM is identical to standard EM [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity:** Each full iteration consists of $M$ sub-iterations. Each sub-iteration processes only $1/M$ of the projections. Thus, one full iteration of OSEM has roughly the same computational cost as one iteration of MLEM, but converges faster in terms of iterations [S2, unofficial].
- **Memory Complexity:** Requires storage for the image, projection data, and system matrix (or access to it). No significant additional memory overhead compared to MLEM [S2, unofficial].
- **Strengths:** Significantly reduced reconstruction time compared to MLEM [S1, S2, unofficial].
- **Limitations:** Prone to divergence earlier than MLEM due to acceleration [S2, unofficial]. Requires careful selection of subsets [S2, unofficial].
- **When to use:** When faster convergence is needed and computational resources per iteration are not the primary bottleneck, or when combined with regularization (MAP-OSEM) to handle divergence [S2, unofficial].
- **When not to use:** When strict monotonic convergence is required without regularization, or when the number of projections is very small (making subset selection difficult) [S2, unofficial].

## Background
- **Origin:** Introduced by Hudson and Larkin in 1994 [S1, S2, unofficial].
- **History:** Developed as an acceleration technique for the EM algorithm in tomographic reconstruction [S1, unofficial]. The method decomposes each iteration into sub-iterations using subsets of projections [S2, unofficial].
- **Usage:** Widely used in medical imaging (e.g., PET, SPECT) and other tomographic applications where fast reconstruction is critical [S2, unofficial].

## Sources
[S1] Reconstruction Tomographique Mojette, https://theses.hal.science/tel-00426920v1, fetched 2026-10-26 (unofficial)
[S2] Élaboration d'une méthode tomographique de reconstruction 3D en vélocimétrie par image de particules basée sur les processus ponctuels marqués, https://hal.science/tel-01292656v3, fetched 2026-10-26 (unofficial)
