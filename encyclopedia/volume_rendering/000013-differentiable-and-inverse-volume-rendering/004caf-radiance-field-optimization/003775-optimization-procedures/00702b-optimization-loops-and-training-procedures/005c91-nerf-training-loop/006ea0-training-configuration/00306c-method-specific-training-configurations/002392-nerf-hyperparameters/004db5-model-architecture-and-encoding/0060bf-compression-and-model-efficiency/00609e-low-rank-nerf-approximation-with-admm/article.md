# Low Rank NeRF Approximation with ADMM

Also called: TT Decompression via ADMM, Tensor Train NeRF Compression

## Overview
This method reduces the model size of a trained Neural Radiance Field (NeRF) by approximating its fully connected layer weights with a Tensor Train (TT) format [S1, unofficial]. It solves the resulting rank-constrained optimization problem using the Alternating Direction Method of Multipliers (ADMM) to minimize reconstruction error while enforcing low-rank structure [S1, unofficial]. It differs from sibling methods like [CCNeRF](15008-ccnerf.md) by explicitly using ADMM to iteratively refine the weight matrices into TT cores, rather than relying solely on static decomposition or pruning [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained NeRF model with fully connected layer weight matrices $\mathbf{W}_i \in \mathbb{R}^{n_1 \times n_2}$ and bias vectors $\mathbf{b}_i$.
- **Output:** Low-rank Tensor Train cores $\mathbf{Q}_{1i}, \mathbf{Q}_{2i}$ and unchanged bias vectors $\mathbf{b}_i$.

**Data Structures:**
- **Weight Matrix $\mathbf{W}_i$:** Original full-rank matrix of size $n_1 \times n_2$.
- **TT Cores:**
  - $\mathbf{Q}_{1i}$: Tensor of shape $1 \times n_1 \times r$. In implementation, the first dimension is squeezed, resulting in a matrix of size $n_1 \times r$.
  - $\mathbf{Q}_{2i}$: Tensor of shape $r \times n_2 \times 1$. In implementation, the last dimension is squeezed, resulting in a matrix of size $r \times n_2$.
  - $r$: Target rank (integer).
- **Dual Multiplier $\mathbf{U}_i$:** Matrix of size $n_1 \times n_2$, initialized to zero or previous iteration values.
- **Auxiliary Variable $\mathbf{Z}_i$:** Matrix of size $n_1 \times n_2$, used in ADMM updates.

**Parameters:**
- **Rank $r$:** Controls compression ratio. Typical experimental values: 40, 50, 70, 90 [S1, unofficial].
- **Penalty Parameter $\rho$:** Positive scalar for the augmented Lagrangian [S1, unofficial]. Value not given in the sources.
- **Learning Rate:** For gradient descent step [S1, unofficial]. Value not given in the sources.

**Algorithm Steps:**
The optimization minimizes the Lagrangian $\mathcal{L}$ defined below. The process iterates until convergence.

1. **Initialize:** Set $\mathbf{W}_i^{(0)}$ to the trained weights, $\mathbf{U}_i^{(0)} = \mathbf{0}$, and $\mathbf{Z}_i^{(0)} = \mathbf{0}$. Set iteration counter $t=0$.
2. **W-Update:** Update $\mathbf{W}_i$ by minimizing the Lagrangian with respect to $\mathbf{W}_i$ using gradient descent.
   $$ \mathbf{W}_i^{(t+1)} = \arg\min_{\mathbf{W}_i} \mathcal{L}(\mathbf{W}_i, \mathbf{Z}_i^{(t)}, \mathbf{U}_i^{(t)}) $$
   [S1, unofficial]
3. **Z-Update:** Update the auxiliary variable $\mathbf{Z}_i$ by projecting the sum of the updated weight and dual multiplier onto the set of rank-$r$ matrices (TT format).
   $$ \mathbf{Z}_i^{(t+1)} = \Pi_r(\mathbf{W}_i^{(t+1)} + \mathbf{U}_i^{(t)}) $$
   where $\Pi_r$ denotes the operation that decomposes the matrix into the TT format truncated to rank $r$ [S1, unofficial].
4. **U-Update:** Update the dual multiplier.
   $$ \mathbf{U}_i^{(t+1)} = \mathbf{U}_i^{(t)} + \mathbf{W}_i^{(t+1)} - \mathbf{Z}_i^{(t+1)} $$
   [S1, unofficial]
5. **Repeat:** Increment $t$ and repeat steps 2-4 until convergence.
6. **Final Decomposition:** After convergence, the optimized weights $\mathbf{W}_i$ are decomposed into $\mathbf{Q}_{1i}$ and $\mathbf{Q}_{2i}$ using the same TT decomposition logic as $\Pi_r$. The bias vectors $\mathbf{b}_i$ remain unchanged [S1, unofficial].

**Formulas:**
- **Augmented Lagrangian (Scaled Dual Form):**
  $$ \mathcal{L}(\mathbf{W}_i, \mathbf{Z}_i, \mathbf{U}_i) = l(\mathbf{W}_i) + g(\mathbf{Z}_i) + \frac{\rho}{2}||\mathbf{W}_i - \mathbf{Z}_i + \mathbf{U}_i||_F^2 + \frac{\rho}{2}||\mathbf{U}_i||_F^2 $$
  where $l(\mathbf{W}_i) = ||\mathbf{c}' - \mathbf{c}_{gt}||_2^2$ is the reconstruction error (rendered pixel value $\mathbf{c}'$ vs ground truth $\mathbf{c}_{gt}$), $g(\mathbf{Z}_i)$ is a regularization term enforcing the TT structure (implicit in the projection $\Pi_r$), $\rho$ is the penalty parameter, and $\mathbf{U}_i$ is the dual multiplier [S1, unofficial].
- **TT Decomposition:**
  $$ \mathbf{W}_i = \mathbf{Q}_{1i} \cdot \mathbf{Q}_{2i} $$
  where $\mathbf{Q}_{1i} \in \mathbb{R}^{1 \times n_1 \times r}$ and $\mathbf{Q}_{2i} \in \mathbb{R}^{r \times n_2 \times 1}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def tt_decomposition(matrix, rank):
    """
    Decomposes a matrix into Tensor Train format (rank r).
    Returns Q1 (n1 x r) and Q2 (r x n2).
    Implementation detail: Uses SVD or similar low-rank approximation.
    Note: The source [S1, unofficial] cites [36] for details on Pi_r.
    We assume a standard low-rank approximation here.
    """
    n1, n2 = matrix.shape
    # Truncated SVD to get rank r approximation
    U, S, Vt = np.linalg.svd(matrix, full_matrices=False)
    # Take top r components
    U_r = U[:, :rank]
    S_r = np.diag(S[:rank])
    Vt_r = Vt[:rank, :]
    
    # TT Cores: Q1 = U_r * sqrt(S_r), Q2 = sqrt(S_r) * Vt_r
    # This ensures Q1 @ Q2 approximates the original matrix
    sqrt_S_r = np.sqrt(S_r)
    Q1 = U_r @ sqrt_S_r
    Q2 = sqrt_S_r @ Vt_r
    
    return Q1, Q2

def low_rank_nerf_admm(W_initial, rank, rho, num_iterations, learning_rate):
    """
    Performs ADMM optimization to compress NeRF weights into TT format.
    
    Args:
        W_initial: Initial weight matrix (n1, n2)
        rank: Target rank r
        rho: Penalty parameter
        num_iterations: Number of ADMM steps
        learning_rate: Step size for W-update
        
    Returns:
        Q1, Q2: Tensor Train cores
    """
    n1, n2 = W_initial.shape
    
    # Initialize variables
    W = W_initial.copy()
    U = np.zeros_like(W)
    Z = np.zeros_like(W)
    
    for t in range(num_iterations):
        # Step 1: W-update
        # Minimize L(W, Z, U) w.r.t W using gradient descent.
        # L = l(W) + rho/2 * ||W - Z + U||^2 + const
        # Gradient of l(W) is the gradient of the rendering loss.
        # Gradient of the quadratic term is rho * (W - Z + U).
        # Note: The source [S1, unofficial] states W-update uses gradient descent.
        # We need the gradient of the loss l(W). This is typically computed via backprop.
        # For this self-contained pseudocode, we assume a function `compute_loss_grad` exists.
        grad_l = compute_loss_grad(W) # [G] Placeholder for actual gradient computation from renderer
        
        # Gradient of the augmented Lagrangian term w.r.t W:
        # d/dW [ rho/2 * ||W - Z + U||_F^2 ] = rho * (W - Z + U)
        grad_aug = rho * (W - Z + U)
        
        total_grad = grad_l + grad_aug
        
        # Update W
        W = W - learning_rate * total_grad
        
        # Step 2: Z-update
        # Z = Pi_r(W + U)
        Z = tt_decomposition(W + U, rank)[0] # [G] tt_decomposition returns (Q1, Q2). 
                                              # The source [S1, unofficial] implies Z is the 
                                              # rank-r approximation. We can store the 
                                              # approximation directly or the cores.
                                              # Eq 13 says Z = Pi_r(...). 
                                              # Let's assume Z is the matrix approximation.
        # Re-compute Z as the matrix approximation for the next step's U update
        # Actually, Eq 13 defines Z as the result of Pi_r. 
        # Let's return the matrix approximation from tt_decomposition for Z.
        # But we need Q1, Q2 for the final output.
        # Let's adjust: Z is the projected matrix.
        U, S, Vt = np.linalg.svd(W + U, full_matrices=False)
        S_r = np.diag(S[:rank])
        Z = U[:, :rank] @ S_r @ Vt[:rank, :]
        
        # Step 3: U-update
        U = U + W - Z
        
    # Final decomposition for storage
    Q1, Q2 = tt_decomposition(W, rank)
    
    return Q1, Q2
```

**Edge Cases and Pitfalls:**
- **Bias Parameters:** Bias vectors $\mathbf{b}_i$ are rank 1 and are not optimized via ADMM; they remain unchanged [S1, unofficial].
- **Approximation Error:** Straightforward decomposition causes large error; ADMM is required to guarantee low-rank property without significant performance degradation [S1, unofficial].
- **Camera Pose:** The method assumes camera pose parameters (focal length $f$, parallax offset $\Delta$) are either known or treated as trainable variables alongside weights [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of ADMM iterations and the cost of SVD/TT decomposition per iteration. Each iteration involves a gradient descent step (cost of forward/backward pass) and a low-rank projection (SVD cost $O(n_1 n_2 \min(n_1, n_2))$) [S1, unofficial].
- **Memory Complexity:** Reduces storage from $O(n_1 n_2)$ to $O(r(n_1 + n_2))$ per layer [S1, unofficial].
- **Performance:** Compression ratio is controlled by rank $r$. Typical ranks 40-90 are used for light field scenes [S1, unofficial].
- **Strengths:** Maintains reconstruction quality better than naive low-rank approximation by using ADMM to optimize the weights under rank constraints [S1, unofficial].
- **Limitations:** Requires iterative optimization post-training. Performance degradation if rank is too low [S1, unofficial].

## Background
The method was proposed by J. Shi et al. in "Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression" [S1, unofficial]. It builds on the idea that NeRF model size is a bottleneck for light field transmission. It uses Tensor Train decomposition [35] and ADMM [36] to solve the rank-constrained optimization problem. The approach is specifically tailored for structured light fields where camera parameters are simplified [S1, unofficial].

## Sources
[S1] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
