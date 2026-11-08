# L1 Sparsity Regularization

Also called: L1 Density Sparsity, L1 Norm Regularization for Density

## Overview
L1 Sparsity Regularization applies an L1 norm penalty to the density field (or saliency grid) of a neural radiance field to enforce sparsity, reducing model complexity and eliminating unnecessary features [S1, S2]. It differs from L1 Norm Regularization for Transfer Functions, which targets opacity and color transfer functions to avoid unnecessary complexity in the rendering pipeline, by directly constraining the volumetric density or feature saliency values [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A neural radiance field parameterized by $\Theta$ (or $\mathcal{W}, \mathcal{H}, \mathcal{G}$), containing a density field $\sigma_\Theta(x)$ or a saliency grid $\mathcal{G}$.
*   **Outputs:** A regularized loss term $R_{l1}$ or an augmented loss function used during optimization.

**Data Structures:**
*   **Density Field $\sigma_\Theta(x)$:** A continuous function mapping 3D coordinates to density. In discrete implementations, it is sampled on a grid.
*   **Saliency Grid $\mathcal{G}$:** A 3D tensor of size $K \times K \times K$ (or mapped to a hashgrid) storing saliency values $g$ for each voxel [S2].
*   **Sigmoid Operator $S_{sig}(\cdot)$:** Applied element-wise to $\mathcal{G}$ to bound saliency values, as the forward pipeline uses sigmoided saliency to weight feature vectors [S2].

**Formulas:**
1.  **Basic L1 Regularization (Density):**
    $$ R_{l1}(\Theta) = \int_{[-1,1]^3} |\sigma_\Theta(x)| dx \approx \sum_{i=1}^N \sum_{j=1}^N \sum_{k=1}^N |\sigma_\Theta(\delta i, \delta j, \delta k)| \delta^3 $$
    where $\delta$ is the grid spacing and $N$ is the number of samples per dimension [S1].

2.  **Basic L1 Regularization (Saliency Grid):**
    $$ \min_{\mathcal{W}, \mathcal{H}, \mathcal{G}} \left( L + \lambda \|S_{sig}(\mathcal{G})\|_1 \right) $$
    where $L$ is the primary training loss (e.g., MSE), $\lambda$ is a constant weight controlling sparsity, and $\| \cdot \|_1$ denotes the L1 norm [S2].

3.  **ADMM Pruner (Constrained Optimization):**
    To overcome the sensitivity of $\lambda$, the problem is formulated as a constrained optimization:
    $$ \min_{\mathcal{W}, \mathcal{H}, \mathcal{G}} L(\mathcal{W}, \mathcal{H}, \mathcal{G}) \quad \text{s.t.} \quad \|S_{sig}(\mathcal{G})\|_1 < C $$
    where $C$ is the sparsity constraint [S2].

4.  **Augmented Lagrangian (Unconstrained Minimax):**
    The constrained problem is transformed into:
    $$ \min_{\mathcal{W}, \mathcal{H}, \mathcal{G}} \max_{\gamma \ge 0} L(\mathcal{W}, \mathcal{H}, \mathcal{G}) + \frac{\rho_\gamma}{2} \left[ \|S_{sig}(\mathcal{G})\|_1 - C \right]_+^2 + \gamma \left( \|S_{sig}(\mathcal{G})\|_1 - C \right) $$
    where $\gamma$ is the trainable dual variable (Lagrange multiplier), $\rho_\gamma$ is its learning rate, and $[x]_+ = \max(0, x)$ [S2].

**Parameters:**
*   $\lambda$: Constant weight for basic L1 regularization. Not given in the sources.
*   $C$: Sparsity constraint threshold for ADMM. Not given in the sources.
*   $\gamma$: Trainable dual variable. Updated via gradient ascent: $\gamma^{(t+1)} = \gamma^{(t)} + \rho_\gamma (\|S_{sig}(\mathcal{G})\|_1 - C)$ [S2].
*   $\rho_\gamma$: Learning rate for $\gamma$. Not given in the sources.

**Steps:**
1.  **Basic Method:**
    1.  Compute the L1 norm of the density field $\sigma_\Theta$ or sigmoided saliency grid $S_{sig}(\mathcal{G})$.
    2.  Multiply by $\lambda$.
    3.  Add to the primary loss $L$.
    4.  Backpropagate to update $\Theta$ (or $\mathcal{W}, \mathcal{H}, \mathcal{G}$) [S1, S2].

2.  **ADMM Pruner Method:**
    1.  Initialize $\gamma \ge 0$.
    2.  **Minimization Step:** Update $\mathcal{W}, \mathcal{H}, \mathcal{G}$ to minimize the augmented Lagrangian loss using gradient descent, treating $\gamma$ as constant [S2].
    3.  **Dual Update Step:** Update $\gamma$ using gradient ascent: $\gamma^{(t+1)} = \gamma^{(t)} + \rho_\gamma (\|S_{sig}(\mathcal{G})\|_1 - C)$ [S2].
    4.  Repeat until convergence [S2].

**Edge Cases and Pitfalls:**
*   **Over-pruning:** If $\lambda$ is too high or training runs too long, visible features may be pruned, degrading performance [S2].
*   **Gradient Persistence:** Even when the grid is sparse, the L1 term may provide gradients that further prune, requiring careful tuning of $\lambda$ [S2].
*   **Hash Collisions:** In hashgrid-based methods (e.g., Instant-NGP), sparsity enforcement helps mitigate the impact of hash collisions by reducing the number of active features, but improper pruning can still affect accuracy [S2].

## Subtopics

### ADMM Pruner
A pruning method using Alternating Direction Method of Multipliers (ADMM) to enforce sparsity in the saliency grid $\mathcal{G}$ by treating sparsity as a constrained optimization problem [S2]. It sets apart from basic L1 regularization by using a trainable dual variable $\gamma$ instead of a fixed $\lambda$, adapting the sparsity constraint dynamically during training [S2]. Choose this when precise control over the sparsity level $C$ is required and fixed $\lambda$ tuning is difficult [S2]. Link: [HollowNeRF](https://doi.org/10.1109/iccv51070.2023.00322).

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead to the forward pass (L1 norm computation). The ADMM variant adds a dual variable update step, which is negligible compared to the main optimization step [S2].
*   **Memory Complexity:** No additional memory required for the basic method. The ADMM method requires storing the dual variable $\gamma$ [S2].
*   **Measured Performance:** HollowNeRF with ADMM pruner reduces parameters significantly (e.g., from 0.50M to 0.76M for $2^{14}$ hashgrid size in Table 1 of [S2]) while maintaining or improving PSNR compared to Instant-NGP [S2].
*   **Strengths:** Reduces model size and complexity; mitigates hash collision effects in hashgrid-based NeRFs; ADMM provides adaptive sparsity control [S2].
*   **Limitations:** Basic L1 requires careful tuning of $\lambda$; over-pruning can degrade quality [S2].
*   **When to Use:** When model size is a constraint or when using hashgrid-based NeRFs where sparsity can mitigate collisions [S2].
*   **When Not to Use:** When high fidelity is required and sparsity might remove fine details [S2].

## Background
L1 regularization is a standard technique in optimization for promoting sparsity. In the context of neural radiance fields, it has been applied to density fields to encourage sparse representations [S1]. The ADMM pruner was introduced in HollowNeRF to address the limitations of simple L1 regularization in hashgrid-based NeRFs, specifically to handle hash collisions and enable adaptive sparsity control [S2].

## Sources
[S1] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
[S2] HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation, https://doi.org/10.1109/iccv51070.2023.00322, fetched 2026-10-26
