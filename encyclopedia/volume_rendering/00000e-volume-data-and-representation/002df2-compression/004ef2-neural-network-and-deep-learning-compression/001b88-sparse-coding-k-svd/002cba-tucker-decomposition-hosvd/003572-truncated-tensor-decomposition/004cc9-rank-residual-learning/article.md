# Rank-Residual Learning

Also called: Parallel Rank-Residual Learning, Parallel Rank-Residual Training, Progressive Rank Training

## Overview
Rank-Residual Learning is a training strategy for tensor-based volume compression that simulates the low-rank approximation property of Singular Value Decomposition (SVD) by training multiple truncation levels in parallel [S1]. Unlike sequential training methods that freeze earlier stages before training new ones, or independent parallel training that detaches gradients, this method simultaneously supervises sequentially accumulated outputs from all groups using a rank-residual loss [S1]. This approach allows each group of ranks to learn residual errors from previous groups, improving training speed while maintaining or improving reconstruction quality compared to baseline truncation methods [S1]. Independent parallel training with detached outputs yields worse final performance due to optimizing later stages with not fully converged earlier stages [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A target total rank $R$ for a tensor decomposition (e.g., CP decomposition of a NeRF representation) [S1].
*   **Output:** A trained decomposition model with factor matrices that support dynamic rank adjustment at inference time without re-optimization [S1].

**Data Structures:**
The method divides the total $R$ ranks into $M$ non-empty groups [S1].
*   Let $R_m$ be the accumulated rank for group $m$, where $m \in \{1, ..., M\}$ [S1].
*   The decomposition is represented by factor matrices (e.g., $U_x, U_y, U_z$ for CP decomposition) [S1].
*   The model computes RGB values $\hat{C}_m(r)$ from the truncated decomposition using only the first $R_m$ rank components [S1].

**Parameters:**
*   $R$: Total target rank.
*   $M$: Number of training groups. Typically $M \ll R$ [S1].
*   $R_m$: Accumulated ranks defining the boundaries of the groups.

**Algorithm Steps:**

1.  **Group Division:** Divide the total $R$ ranks into $M$ non-empty groups with accumulated ranks $R_1, R_2, ..., R_M$ [S1].
2.  **Parallel Training:** Train all $M$ groups simultaneously without detaching outputs from previous stages [S1]. For each group $m$, the loss function applies to all its previous rank groups [S1].
3.  **Loss Calculation:** Compute the rank-residual loss $\mathcal{L}_{residual}$ by supervising the sequentially accumulated outputs from all $M$ groups [S1]:
    $$
    \mathcal{L}_{residual} = \sum_{m=1}^{M} \sum_{r \in \mathcal{R}} ||C(r) - \hat{C}_m(r)||_2^2
    $$
    where $C(r)$ is the ground truth RGB value at rank configuration $r$, and $\hat{C}_m(r)$ is the predicted RGB value from the truncated decomposition keeping only the first $R_m$ rank components [S1].
4.  **Residual Learning:** During training, each group learns the residual error from previous groups [S1]. Ideally, choosing $M=R$ assures the low-rank approximation property holds at any targeted rank, but this is computationally unaffordable for large $R$ [S1]. In practice, $M \ll R$, so the property holds at the dividing ranks $\{R_m\}$ [S1].
5.  **Inference Truncation (Empirical Sort-and-Truncate):** For any target rank $R'$ such that $R_m < R' < R_{m+1}$:
    *   Keep the fully covered $R_m$ rank components from the previous groups [S1].
    *   Select the remaining $R' - R_m$ ranks from the last group (indices $R_m + 1$ to $R_{m+1}$) based on the largest average importance [S1].
    *   Rank importance is determined by sorting ranks column-wise based on their average importance metric [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_rank_residual_learning(target_rank_R, num_groups_M, data_loader, learning_rate, max_epochs):
    # 1. Define group boundaries
    accumulated_ranks = [target_rank_R * (m + 1) // num_groups_M for m in range(num_groups_M)]
    
    # 2. Initialize factor matrices for the full rank R
    U_x = [[0.0] * target_rank_R for _ in range(target_rank_R)]
    U_y = [[0.0] * target_rank_R for _ in range(target_rank_R)]
    U_z = [[0.0] * target_rank_R for _ in range(target_rank_R)]
    
    # 3. Parallel Training Loop
    for epoch in range(max_epochs):
        for batch in data_loader:
            ground_truth_rgb = batch.rgb
            total_loss = 0.0
            
            # 4. Compute loss for each group simultaneously
            for m in range(num_groups_M):
                R_m = accumulated_ranks[m]
                
                # Truncate decomposition to first R_m ranks
                U_x_trunc = [row[:R_m] for row in U_x]
                U_y_trunc = [row[:R_m] for row in U_y]
                U_z_trunc = [row[:R_m] for row in U_z]
                
                # Calculate predicted RGB from truncated CP decomposition
                predicted_rgb = reconstruct_cp(U_x_trunc, U_y_trunc, U_z_trunc)
                
                # Compute L2 loss for this truncation level
                loss_m = sum((gt - pred) ** 2 for gt, pred in zip(ground_truth_rgb, predicted_rgb))
                total_loss += loss_m
            
            # 5. Backpropagate and update parameters
            # Gradient computation and parameter update are not specified in the sources [G]
            # Standard gradient descent would update U_x, U_y, U_z using total_loss
            
    # 6. Compute Rank Importance for Inference
    rank_importance = compute_average_rank_importance(U_x, U_y, U_z)
    
    return U_x, U_y, U_z, rank_importance

def reconstruct_cp(U_x, U_y, U_z):
    # CP decomposition reconstruction: sum over ranks of outer products
    return [sum(U_x[k][i] * U_y[k][j] * U_z[k][l] for k in range(len(U_x))) 
            for i in range(len(U_x[0])) for j in range(len(U_y[0])) for l in range(len(U_z[0]))]

def compute_average_rank_importance(U_x, U_y, U_z):
    # Average importance per rank across dimensions
    return [sum(abs(U_x[k][i]) + abs(U_y[k][i]) + abs(U_z[k][i]) for i in range(len(U_x[0]))) 
            for k in range(len(U_x))]

def infer_with_dynamic_rank(U_x, U_y, U_z, rank_importance, target_rank_R_prime, accumulated_ranks):
    selected_ranks = []
    
    # Keep fully covered groups
    for m in range(len(accumulated_ranks)):
        R_m = accumulated_ranks[m]
        if target_rank_R_prime >= R_m:
            start_idx = selected_ranks[-1] if selected_ranks else 0
            end_idx = R_m
            selected_ranks.extend(range(start_idx, end_idx))
        else:
            break
            
    # Handle the remaining ranks from the last partially covered group
    remaining_ranks_needed = target_rank_R_prime - len(selected_ranks)
    if remaining_ranks_needed > 0:
        last_group_start = selected_ranks[-1] if selected_ranks else 0
        last_group_end = len(U_x)
        
        # Get importance scores for ranks in the last group
        last_group_importance = rank_importance[last_group_start:last_group_end]
        
        # Select top 'remaining_ranks_needed' ranks based on importance
        sorted_indices = sorted(range(len(last_group_importance)), key=lambda i: last_group_importance[i], reverse=True)
        top_indices = sorted_indices[:remaining_ranks_needed]
        
        # Map back to global indices
        global_top_indices = [last_group_start + idx for idx in top_indices]
        selected_ranks.extend(global_top_indices)
        
    # Reconstruct using selected ranks
    U_x_sel = [row[selected_ranks] for row in U_x]
    U_y_sel = [row[selected_ranks] for row in U_y]
    U_z_sel = [row[selected_ranks] for row in U_z]
    
    return reconstruct_cp(U_x_sel, U_y_sel, U_z_sel)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The empirical sort-and-truncate strategy relies on accurate rank importance metrics. If importance is not concentrated, compression quality drops sharply [S1].
*   **Boundary Conditions:** The method assumes $R_m$ boundaries are chosen such that $M \ll R$. If $M=R$, the low-rank approximation property holds exactly, but training is inefficient [S1].

## Subtopics

### Incremental Compression
Uses projection matrices $P$ to handle truncation problems. The structure involves projecting factor matrices $U$ using $P$ (e.g., $P_1, P_2, P_3$) to update the decomposition [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Parallel training is significantly faster than sequential training, which requires each stage to fully converge before increasing ranks [S1]. Independent parallel training with detached outputs yields worse final performance due to optimizing later stages with not fully converged earlier stages [S1].
*   **Memory Complexity:** Requires storing factor matrices for the full rank $R$ during training, but allows dynamic rank adjustment at inference without extra optimization [S1].
*   **Performance:** Achieves near-optimal compression at any rank when combined with empirical sort-and-truncate, closing the performance gap of baseline truncation methods [S1].
*   **Strengths:** Enables dynamic adjustment of model size and rendering quality (e.g., for texture mipmaps) with no extra optimization [S1]. Eases training of earlier stages by letting later stages model complex details [S1].
*   **Limitations:** Requires careful selection of $M$ and rank importance metrics. Not given in the sources for specific hardware performance metrics.

## Background
Rank-Residual Learning was proposed in the context of Compressible-Composable NeRF to address the sharp drop in rendering quality when truncating models based on rank importance [S1]. It builds on the principles of tensor decomposition and low-rank approximation, aiming to simulate SVD's properties in neural network representations [S1]. The method was introduced in "Compressible-Composable NeRF Via Rank-Residual Decomposition" [S1].

## Sources
[S1] Compressible-Composable NeRF Via Rank-Residual Decomposition, https://doi.org/10.52202/068431-1076, fetched 2026-10-26
[S2] PDF Analysis of Tensor Approximation for Compression-Domain Volume ..., https://www.ifi.uzh.ch/vmml/publications/analysistensorapproximationtalk/tensorapproximation.pdf, fetched 2026-10-29 (unofficial)
