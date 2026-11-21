# Low Rank NeRF Approximation with ADMM

Also called: TT Decompression via ADMM, Tensor Train NeRF Compression

## Overview
This method reduces the model size of a trained Neural Radiance Field (NeRF) by approximating its fully connected layer weight matrices with low-rank Tensor Train (TT) decompositions [S1, unofficial]. It solves the resulting rank-constrained optimization problem using the Alternating Direction Method of Multipliers (ADMM) to minimize reconstruction error while enforcing the low-rank structure [S1, unofficial]. It is distinct from sibling methods like [CCNeRF](15008-ccnerf.md) by explicitly using ADMM to decouple the optimization of the dense weights from the projection onto the low-rank manifold, allowing for precise control over the compression rank $r$ without immediate performance degradation [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A pre-trained NeRF model with dense weight matrices $\mathbf{W}_i \in \mathbb{R}^{n_1 \times n_2}$ for each fully connected layer $i$, and a dataset of rays with ground-truth pixel values $\mathbf{c}_{gt}$.
*   **Output:** Low-rank TT cores $\mathbf{Q}_{1i} \in \mathbb{R}^{n_1 \times r}$ and $\mathbf{Q}_{2i} \in \mathbb{R}^{r \times n_2}$ for each layer, where $r$ is the target rank. The bias vectors $\mathbf{b}_i$ remain unchanged.

**Data Structures**
*   $\mathbf{W}_i$: Dense weight matrix of size $n_1 \times n_2$.
*   $\mathbf{Z}_i$: Auxiliary variable matrix of size $n_1 \times n_2$, used in ADMM to enforce constraints.
*   $\mathbf{U}_i$: Dual multiplier matrix of size $n_1 \times n_2$.
*   $\mathbf{Q}_{1i}$: First TT core, size $n_1 \times r$.
*   $\mathbf{Q}_{2i}$: Second TT core, size $r \times n_2$.
*   $\rho$: Scalar penalty parameter.

**Parameters**
*   $r$: Target rank controlling compression ratio. Typical experimental values are 40, 50, 70, and 90 [S1, unofficial].
*   $\rho$: Positive penalty parameter for the augmented Lagrangian. Value not specified in sources [G].
*   $\eta$: Learning rate for gradient descent updates. Value not specified in sources [G].

**Algorithm Steps**
The optimization minimizes the augmented Lagrangian $\mathcal{L}$ defined as:
$$
\mathcal{L}(\mathbf{W}_i, \mathbf{Z}_i, \mathbf{U}_i) = l(\mathbf{W}_i) + g(\mathbf{Z}_i) + \frac{\rho}{2}||\mathbf{W}_i - \mathbf{Z}_i + \mathbf{U}_i||_F^2 + \frac{\rho}{2}||\mathbf{U}_i||_F^2
$$
where $l(\mathbf{W}_i) = ||\mathbf{c}' - \mathbf{c}_{gt}||_2^2$ is the reconstruction error (rendering loss), and $g(\mathbf{Z}_i)$ is an indicator function that is 0 if $\mathbf{Z}_i$ has rank $\le r$ and $\infty$ otherwise [S1, unofficial].

1.  **Initialization:** Initialize $\mathbf{W}_i^0$ with the pre-trained weights. Initialize $\mathbf{Z}_i^0 = \mathbf{W}_i^0$ and $\mathbf{U}_i^0 = \mathbf{0}$. Set iteration counter $t=0$.
2.  **W-Update:** Update the dense weights $\mathbf{W}_i$ by minimizing the Lagrangian with respect to $\mathbf{W}_i$, keeping $\mathbf{Z}_i^t$ and $\mathbf{U}_i^t$ fixed. This is solved via gradient descent:
    $$
    \mathbf{W}_i^{t+1} = \arg\min_{\mathbf{W}_i} \mathcal{L}(\mathbf{W}_i, \mathbf{Z}_i^t, \mathbf{U}_i^t)
    $$
    The gradient is computed based on the rendering loss $l(\mathbf{W}_i)$ and the quadratic penalty term [S1, unofficial].
3.  **Z-Update:** Update the auxiliary variable $\mathbf{Z}_i$ by minimizing the Lagrangian with respect to $\mathbf{Z}_i$, keeping $\mathbf{W}_i^{t+1}$ and $\mathbf{U}_i^t$ fixed. This step projects the updated weights onto the low-rank manifold:
    $$
    \mathbf{Z}_i^{t+1} = \arg\min_{\mathbf{Z}_i} \mathcal{L}(\mathbf{W}_i^{t+1}, \mathbf{Z}_i, \mathbf{U}_i^t) = \Pi_r(\mathbf{W}_i^{t+1} + \mathbf{U}_i^t)
    $$
    Here, $\Pi_r(\cdot)$ denotes the operation that decomposes a matrix into the Tensor Train format with target rank $r$ [S1, unofficial].
4.  **U-Update:** Update the dual multiplier $\mathbf{U}_i$:
    $$
    \mathbf{U}_i^{t+1} = \mathbf{U}_i^t + \mathbf{W}_i^{t+1} - \mathbf{Z}_i^{t+1}
    $$
5.  **Convergence Check:** If convergence criteria are not met, increment $t$ and repeat from Step 2.
6.  **Final Decomposition:** After optimization, the final weights $\mathbf{W}_i$ are approximated by the TT cores $\mathbf{Q}_{1i}$ and $\mathbf{Q}_{2i}$ derived from $\mathbf{Z}_i$. The first dimension of $\mathbf{Q}_{1i}$ and the last dimension of $\mathbf{Q}_{2i}$ are squeezed/ignored, resulting in final sizes $n_1 \times r$ and $r \times n_2$ respectively [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def low_rank_nerf_admm(W_initial, rays, gt_pixels, rank_r, rho, num_iterations, learning_rate):
    """
    Compress NeRF weights using ADMM.
    
    Args:
        W_initial: List of dense weight matrices for each layer.
        rays: List of ray origins and directions.
        gt_pixels: Ground truth pixel values for the rays.
        rank_r: Target TT rank.
        rho: ADMM penalty parameter.
        num_iterations: Number of ADMM iterations.
        learning_rate: Step size for W-update gradient descent.
        
    Returns:
        Q1_cores: List of first TT cores.
        Q2_cores: List of second TT cores.
    """
    # Initialize variables for each layer
    W = [W.copy() for W in W_initial]
    Z = [W.copy() for W in W_initial]
    U = [np.zeros_like(W) for W in W_initial]
    
    Q1_cores = []
    Q2_cores = []
    
    for layer_idx, W_layer in enumerate(W):
        n1, n2 = W_layer.shape
        
        # Initialize Z and U for this layer
        Z_layer = W_layer.copy()
        U_layer = np.zeros_like(W_layer)
        
        for t in range(num_iterations):
            # Step 1: W-update
            # Minimize L with respect to W. 
            # L = l(W) + rho/2 * ||W - Z + U||_F^2 + const
            # Gradient: grad_l(W) + rho * (W - Z + U)
            
            # Compute rendering loss and gradient w.r.t W
            # Note: In practice, this requires backpropagation through the NeRF renderer
            rendered_pixels = render_nerf_with_weights(W_layer, rays)
            loss = np.sum((rendered_pixels - gt_pixels) ** 2)
            
            # Compute gradient of loss w.r.t W_layer
            grad_loss = compute_gradient_loss(W_layer, rays, gt_pixels)
            
            # Compute gradient of penalty term
            grad_penalty = rho * (W_layer - Z_layer + U_layer)
            
            # Full gradient
            grad_W = grad_loss + grad_penalty
            
            # Gradient descent step
            W_layer = W_layer - learning_rate * grad_W
            
            # Step 2: Z-update
            # Z = Pi_r(W + U)
            # Pi_r performs Tensor Train decomposition truncated to rank r
            Z_layer = tensor_train_decompose(W_layer + U_layer, rank_r)
            
            # Step 3: U-update
            U_layer = U_layer + W_layer - Z_layer
            
        # Final decomposition for storage
        # The sources state that after ADMM, W is low rank and can be decomposed.
        # We perform the final TT decomposition on the optimized W (or Z, they should be close)
        # The sources specify neglecting the first dim of Q1 and last of Q2.
        Q1, Q2 = tensor_train_decompose(W_layer, rank_r)
        
        # Squeeze dimensions as per source description
        # Q1 was 1 x n1 x r, now n1 x r
        # Q2 was r x n2 x 1, now r x n2
        Q1_cores.append(Q1.squeeze(axis=0))
        Q2_cores.append(Q2.squeeze(axis=-1))
        
    return Q1_cores, Q2_cores

def tensor_train_decompose(M, rank):
    """
    Decompose matrix M into TT cores Q1, Q2 with target rank.
    This is a placeholder for the specific algorithm in [37] referenced by [S1, unofficial].
    Typically involves SVD-based truncation.
    """
    # Implementation detail not fully specified in [S1, unofficial], but [37] is cited.
    # Standard approach: Iterative SVD or ALS.
    # For the purpose of this article, we assume a function exists that returns
    # Q1 (1 x n1 x r) and Q2 (r x n2 x 1).
    pass
```

**Edge Cases and Pitfalls**
*   **Bias Parameters:** Bias vectors $\mathbf{b}_i$ are rank 1 and are not optimized via ADMM; they remain unchanged during the process [S1, unofficial].
*   **Approximation Error:** Straightforward decomposition without ADMM causes large approximation errors. The ADMM procedure is critical to guarantee low-rank properties without significant performance degradation [S1, unofficial].
*   **Camera Pose:** The method assumes camera poses are either known or treated as trainable variables (focal length $f$ and parallax offset $\Delta$) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the iterative ADMM process. Each iteration involves a gradient descent step (dependent on the number of rays and network depth) and a TT decomposition step (typically $O(n_1 n_2 r)$ or similar depending on the decomposition algorithm). The total time is $O(T \cdot (N_{rays} \cdot D + n_1 n_2 r))$, where $T$ is the number of ADMM iterations, $D$ is the cost of a forward/backward pass, and $n_1, n_2$ are layer dimensions.
*   **Memory Complexity:** The memory required for weights is reduced from $O(n_1 n_2)$ to $O(r(n_1 + n_2))$ per layer. The ADMM process requires storing $\mathbf{W}, \mathbf{Z}, \mathbf{U}$, so peak memory is $3 \cdot n_1 n_2$ per layer during optimization, plus the final low-rank storage.
*   **Strengths:** Allows precise control over compression ratio via rank $r$. Maintains reconstruction quality better than direct low-rank approximation.
*   **Limitations:** Requires an initial trained NeRF. The ADMM optimization adds a significant pre-processing step. The specific decomposition algorithm $\Pi_r$ is not detailed in the source, relying on external reference [37].
*   **When to use:** When transmitting or storing NeRF models is a bottleneck, particularly for structured light fields.
*   **When not to use:** When training time is constrained, or if the target rank $r$ is too low to capture scene complexity, leading to quality loss.

## Background
The method originates from the work on distilling low-rank neural radiance fields for light field compression [S1, unofficial]. It builds on the original NeRF architecture [16] and the Tensor Train decomposition technique [35]. The use of ADMM for tensor rank optimization is based on prior work [36, 37] cited in the source. The approach addresses the large model size of NeRFs, which makes transmitting the network less efficient than transmitting compressed light field views [S1, unofficial].

## Sources
[S1] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
