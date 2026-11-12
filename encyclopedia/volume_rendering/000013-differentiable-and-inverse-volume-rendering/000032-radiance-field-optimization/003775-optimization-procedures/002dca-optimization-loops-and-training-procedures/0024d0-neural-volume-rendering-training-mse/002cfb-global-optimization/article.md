# Global Optimization


## Overview
Global Optimization reconstructs a complete image or volume from sparse measurements by solving for all pixels/voxels simultaneously across the entire domain [S1][S2, unofficial]. It differs from local or block-based solvers by optimizing the full domain at once, improving reconstruction quality at the cost of higher memory and per-iteration computational cost [S1][S2, unofficial]. It is applied in compressive rendering and tomographic reconstruction [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sparse measurement set (either a 2D image with missing pixels $I_{\text{sparse}}$ and mask $M$, or a 3D volume representation $V$ with reference projection $P$) [S1][S2, unofficial].
- Output: A reconstructed full image $I_{\text{full}}$ or an optimized volume representation $V'$ [S1][S2, unofficial].

**Data Structures:**
- $I_{\text{sparse}}$: 2D array of size $m \times n$ containing known pixel values; missing pixels are masked out [S2, unofficial].
- $M$: Binary 2D array of size $m \times n$, where $M_{ij} = 1$ if pixel $(i,j)$ is known, and $0$ otherwise [S2, unofficial].
- $V$: 3D volume representation (e.g., set of 3D Gaussians with mean, covariance, opacity, color parameters) [S1].
- $P$: Reference 2D projection image of size $m \times n \times p$ (where $p$ is channels) [S1].
- $\hat{P}$: Estimated projection of size $m \times n \times p$ [S1].

**Steps:**
1. **Initialization**: Initialize the target array ($I_{\text{full}}$ or $V$) with known values or random/learned parameters [G].
2. **Iteration Loop**: Repeat until convergence or a fixed iteration count [S2, unofficial]:
   a. **Projection/Forward Pass**: Map the current representation to the observation domain. For compressive rendering, this is implicit in the error computation [S2, unofficial]. For volume rendering, project $V$ using transformation $\mathcal{T}$: $\hat{P} = \mathcal{T}(V)$ [S1].
   b. **Loss Computation**: Compute reconstruction error. For compressive rendering, error is defined only on known pixels [S2, unofficial]. For volume rendering, compute total loss combining L1, SSIM, and Total Variation (TV) terms [S1].
   c. **Gradient Computation**: Compute gradients of the loss with respect to the representation parameters [G].
   d. **Parameter Update**: Apply gradient descent to update the representation [G].
   e. **Adaptive Density Control** (Volume Rendering only): Clone under-reconstructed Gaussians, split over-reconstructed ones, and prune those with near-zero gradient magnitudes [S1].
3. **Termination**: Stop when the change in the representation between iterations falls below a threshold or after a fixed number of iterations [S2, unofficial][G].

**Formulas:**
- Projection transformation: $\hat{P} = \mathcal{T}(V)$, where $\hat{P} \in \mathbb{R}^{m \times n \times p}$ [S1].
- Total loss function: $\mathcal{L}_{\text{total}} = \lambda_1 \mathcal{L}_1(\hat{P}, P) + \lambda_2 \mathcal{L}_{\text{SSIM}}(\hat{P}, P) + \lambda_3 \mathcal{L}_{\text{TV}}(V)$ [S1].
- Compressive rendering error gradient (MSE on known pixels): $\nabla E = 2 \cdot (I_{\text{full}} - I_{\text{sparse}}) \odot M$ [G].

**Parameters:**
- Loss weights: $\lambda_1 = 0.6$ (L1), $\lambda_2 = 0.2$ (SSIM), $\lambda_3 = 1$ (TV) [S1].
- Iterations: Fixed count or dynamic until desired solution is reached [S2, unofficial].
- Learning rate $\alpha$: Not specified in sources [G].

**Edge Cases and Pitfalls:**
- Precision: TV loss gradients can cause oscillations; gradient clipping or adaptive learning rates are recommended [G].
- Boundaries: Mask enforcement must use indexing rather than multiplication to avoid overwriting known pixels with zeros when missing pixels are initialized to zero [G].
- Degenerate input: If all pixels are missing, the solver returns the initialization; a fallback or regularization prior is required [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def global_solver_compressive_rendering(I_sparse, mask, iterations=100, alpha=0.1):
    """Reconstructs a full image from a sparse subset using a global gradient solver."""
    I_full = I_sparse.copy()
    for t in range(iterations):
        # Gradient of MSE error on known pixels
        error = I_full - I_sparse
        gradient = error * mask * 2.0
        # Gradient descent step
        I_full = I_full - alpha * gradient
        # Constraint enforcement: known pixels remain unchanged
        I_full[mask == 1] = I_sparse[mask == 1]
    return I_full

def global_optimizer_volume_rendering(V, P, lambda1=0.6, lambda2=0.2, lambda3=1.0, iterations=1000, lr=0.01):
    """Optimizes a 3D volume representation by minimizing projection error."""
    for t in range(iterations):
        # 1. Project volume to 2D domain
        P_hat = T(V)  # Transformation T maps V to estimated projection [S1]
        
        # 2. Compute losses
        l1_loss = mean(abs(P_hat - P))  # L1 loss [G]
        ssim_loss = structural_similarity(P_hat, P)  # Standard SSIM metric [G]
        tv_loss = total_variation(V)  # Standard TV regularization [G]
        
        # 3. Total loss
        total_loss = lambda1 * l1_loss + lambda2 * ssim_loss + lambda3 * tv_loss
        
        # 4. Compute gradients and update parameters
        gradients = compute_gradient(total_loss, V)  # Standard backpropagation [G]
        V = V - lr * gradients  # Gradient descent update [G]
        
        # 5. Adaptive density control
        # Clone under-reconstructed, split over-reconstructed, prune near-zero gradients [S1]
        # Exact heuristics not specified in sources [G]
        
    return V
```

## Subtopics

### Compressive Rendering (Global Solver)
A variant of global optimization specifically for reconstructing images from sparse pixel data [S2, unofficial]. It uses a gradient solver that operates on the entire image, as opposed to block-based approaches [S2, unofficial]. It is chosen when global consistency is more important than computational speed per iteration [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the number of iterations and the cost of gradient computation [S2, unofficial]. For compressive rendering, convolution dominates runtime and scales linearly with iteration count [S2, unofficial]. For volume rendering, each iteration projects the 3D volume and computes losses, avoiding the 30K iterations required by standard approaches [S1].
- **Memory Complexity**: Requires storing the full image or volume representation, which grows with resolution [S1][S2, unofficial].
- **Strengths**: Produces higher quality reconstructions by leveraging global context and is compatible with Fan-Beam, Cone-Beam, Sparse-View, and Limited-Angle CT geometries [S1].
- **Limitations**: Slower per-iteration runtime and high memory usage; CPU-GPU memory transfers further slow compressive rendering solvers if not optimized [S2, unofficial].
- **When to use**: When reconstruction quality and global consistency are prioritized over speed, and for sparse-view or limited-angle CT reconstruction [S1][S2, unofficial].
- **When not to use**: When real-time performance is required or when memory is constrained [S1][S2, unofficial].

## Background
Global optimization in compressive rendering was proposed in a thesis on GPU-based compressive volume rendering, which suggested a parallel-based reconstruction approach [S2, unofficial]. In neural volume rendering, global optimization using differentiable rendering and adaptive density control was introduced for discretized Gaussian representation in tomographic reconstruction, demonstrating substantial improvement over standard methods [S1].

## Sources
[S1] Discretized Gaussian Representation for Tomographic Reconstruction, https://doi.org/10.1109/ICCV51701.2025.02325, fetched 2026-10-27
[S2] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
