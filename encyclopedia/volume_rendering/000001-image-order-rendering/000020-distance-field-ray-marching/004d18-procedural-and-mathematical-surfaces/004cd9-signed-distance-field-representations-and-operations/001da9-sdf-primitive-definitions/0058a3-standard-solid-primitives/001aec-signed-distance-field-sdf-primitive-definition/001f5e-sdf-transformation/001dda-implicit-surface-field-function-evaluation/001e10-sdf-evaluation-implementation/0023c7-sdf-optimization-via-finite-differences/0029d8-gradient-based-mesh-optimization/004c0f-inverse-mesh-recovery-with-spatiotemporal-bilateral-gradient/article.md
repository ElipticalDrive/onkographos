# Inverse Mesh Recovery with Spatiotemporal Bilateral Gradient Filtering

Also called: Edge-aware cross-bilateral filter for mesh recovery, Spatiotemporal optimizer for inverse recovery

## Overview
This method is a gradient-based optimization technique for recovering mesh geometry from multi-view images by optimizing vertex positions. It solves the problem of slow convergence and loss of sharp features (silhouettes, thin structures) in inverse rendering, which occurs when standard Laplacian smoothing dilutes sparse silhouette gradients. It sets itself apart from its parent [Gradient-Based Mesh Optimization](4628-gradient-based-mesh-optimization.md) by replacing the Laplacian postfilter with an edge-aware cross-bilateral filter that preserves values on silhouettes, enabling faster convergence on models with thin structures or many silhouettes [S1].

## How it works
The method follows the Large Steps framework [S1], but replaces the postfilter on the intermediate variable $m_t$ with a mesh cross-bilateral filter. The optimization minimizes the geometric difference between the rendered views of the current mesh and the reference multi-view images.

**Inputs:**
- Reference multi-view images (depth/mask or photometric data).
- Initial mesh geometry (typically a sphere).
- Hyperparameters: learning rate $\alpha$, Adam momentum $\beta_1, \beta_2$, regularization $\lambda$, bilateral filter range parameters $\sigma_\phi, \sigma_d$.

**Outputs:**
- Optimized mesh vertex positions.

**Data Structures:**
- **Mesh:** Defined by vertex positions $V \in \mathbb{R}^{3 \times N}$ and connectivity.
- **Vertex Normals:** $\hat{n} \in \mathbb{R}^{3 \times N}$, computed from mesh geometry.
- **Gradient Vectors:** $h \in \mathbb{R}^N$ (scalar field to be filtered, e.g., displacement or loss gradient).
- **Laplacian Matrix:** $L \in \mathbb{R}^{N \times N}$, the standard cotangent Laplacian.

**Parameters:**
- $\alpha$ (learning rate): $1 \times 10^{-2}$ for cube, $3 \times 10^{-2}$ for dragon [S1].
- $\beta_1, \beta_2$ (Adam momentum): $0.9, 0.99$ [S1].
- $\lambda$ (regularization weight): $19$ [S1].
- $\sigma_\phi$ (photometric range parameter): $0.4$ [S1].
- $\sigma_d$ (spatial distance parameter): $0.5$ [S1].
- $N_{samples}$: 16 samples per pixel for forward rendering, 1 sample per pixel for gradient [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize the mesh as a sphere with approximately 10,000 vertices [S1].
2.  **Iteration Loop:** For each optimization step $t$:
    a.  **Rendering:** Render all views using the current mesh. Use 16 samples per pixel for the forward pass and 1 sample per pixel for the gradient computation [S1].
    b.  **Loss Computation:** Compute the loss using the L1 norm between the rendered output and the reference [S1].
    c.  **Gradient Update:** Compute the gradient of the loss with respect to the mesh parameters. Update the vertex positions using the UniformAdam optimizer with parameters $\beta_1=0.9, \beta_2=0.99$ and learning rate $\alpha$ [S1].
    d.  **Prefiltering:** Apply a Laplacian prefilter to the gradient/parameter update. This step is crucial for diffusing silhouette gradients [S1].
    e.  **Postfiltering (Cross-Bilateral Filter):** Apply the mesh cross-bilateral filter to the intermediate variable $m_t$ (the update or state being filtered). This replaces the Laplacian smoothing used in Large Steps [S1].
        i.  Sample $N=32$ directions $s_i \in S^2$ on the unit sphere using the Sobol sequence [S1].
        ii. Compute diagonal data matrices $W_i$ and partition of unity matrices $\Phi_i$ for each direction $s_i$.
        iii. Construct the matrix $M$ by concatenating vectors $W_i h$ and $W_i \mathbf{1}$.
        iv. Solve the linear system $(I + \lambda L) X = M$ for $X$.
        v.  Compute the filtered result $\tilde{h}$ using component-wise operations on $X$ [S1].
    f.  **Optional Postfilter on $v_t$:** In ablation studies, applying a bilateral filter on the intermediate variable $v_t$ (along with UniformAdam infinity norm) was found to perform well, though it may cause small cracks on sharp edges [S1].
    g.  **Learning Rate Schedule:** Apply an exponential learning rate scheduler that reduces the learning rate to a factor of 0.6 at the end of the optimization [S1].
3.  **Upsampling (Conditional):** For models with thin structures (e.g., dragon), upsample the mesh at specific iterations (10, 30, 60) by remeshing to reduce the average edge length by a factor of 2. Models with sharp edges but no thin structures (e.g., cube) do not require remeshing [S1].

**Cross-Bilateral Filter Formula:**
The filtered vector $\tilde{h}$ is computed as:
$$
\tilde{h} = \frac{\sum_{i=1}^N \Phi(\hat{n}, s_i) (I + \lambda L)^{-1} W(\hat{n}, s_i) h}{\sum_{i=1}^N \Phi(\hat{n}, s_i) (I + \lambda L)^{-1} W(\hat{n}, s_i) \mathbf{1}}
$$
where:
- $h$ is the vector to be filtered.
- $\mathbf{1}$ is the vector of ones.
- $W_i = W(\hat{n}, s_i)$ is the diagonal data matrix with elements $\exp(\hat{n} \cdot s_i / \sigma_d)$.
- $\Phi_i = \Phi(\hat{n}, s_i)$ is the diagonal partition of unity matrix with elements $\exp(\hat{n} \cdot s_i / \sigma_\phi)$.
- $L$ is the Laplacian matrix.
- $\lambda$ is the regularization parameter [S1].

**Efficient Implementation:**
To avoid $N$ separate matrix solves, concatenate the vectors $W_i h$ and $W_i \mathbf{1}$ into a single matrix $M \in \mathbb{R}^{N \times 2N}$:
$$
M = \left[ \begin{array}{c|c} W_1 h & W_1 \mathbf{1} \\ \vdots & \vdots \\ W_N h & W_N \mathbf{1} \end{array} \right]
$$
Compute $X = (I + \lambda L)^{-1} M$ using a single matrix solve (e.g., Cholesky decomposition) [S1]. The final $\tilde{h}$ is obtained by component-wise division of the first $N$ columns of $X$ by the second $N$ columns, weighted by $\Phi_i$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def sobol_sequence_sphere(n):
    """
    Generate n directions on the unit sphere using Sobol sequence.
    Returns tensor of shape (n, 3).
    """
    # Implementation detail: Use a standard Sobol sampler mapped to sphere
    # For reproducibility, this is a placeholder for the specific Sobol implementation
    # referenced in [S1].
    pass

def compute_cross_bilateral_filter(h, normals, L, lambda_reg, sigma_phi, sigma_d, n_samples=32):
    """
    Compute the mesh cross-bilateral filter.
    
    Args:
        h: Tensor (N,) - vector to be filtered
        normals: Tensor (N, 3) - vertex normals
        L: Tensor (N, N) - Laplacian matrix
        lambda_reg: float - regularization parameter
        sigma_phi: float - photometric range parameter
        sigma_d: float - spatial distance parameter
        n_samples: int - number of directions (default 32)
        
    Returns:
        h_tilde: Tensor (N,) - filtered vector
    """
    N = h.shape[0]
    
    # 1. Sample directions
    directions = sobol_sequence_sphere(n_samples) # Shape (n_samples, 3)
    
    # 2. Compute weights for each direction
    # normals: (N, 3), directions: (n_samples, 3)
    # dot products: (N, n_samples)
    dot_products = torch.matmul(normals, directions.T) 
    
    # W_i diagonal elements: exp(n_hat . s_i / sigma_d)
    # Shape: (N, n_samples)
    W_weights = torch.exp(dot_products / sigma_d)
    
    # Phi_i diagonal elements: exp(n_hat . s_i / sigma_phi)
    # Shape: (N, n_samples)
    Phi_weights = torch.exp(dot_products / sigma_phi)
    
    # 3. Construct matrix M
    # M = [W_1 h ... W_N h | W_1 1 ... W_N 1]
    # We need to compute W_i * h for each i.
    # W_i is a diagonal matrix. W_i * h is element-wise multiplication of h with the i-th column of W_weights.
    
    # h: (N,), W_weights: (N, n_samples)
    # W_h: (N, n_samples) where W_h[:, i] = W_i * h
    W_h = h.unsqueeze(1) * W_weights
    
    # W_1: (N,), W_weights: (N, n_samples)
    # W_1_vec: (N, n_samples) where W_1_vec[:, i] = W_i * 1 = W_i
    W_1_vec = W_weights
    
    # Concatenate along the last dimension
    # M shape: (N, 2 * n_samples)
    M = torch.cat([W_h, W_1_vec], dim=1)
    
    # 4. Solve (I + lambda * L) * X = M
    # I + lambda * L
    A = torch.eye(N, device=h.device) + lambda_reg * L
    
    # Solve for X
    # Using Cholesky solver as per [S1]
    # X shape: (N, 2 * n_samples)
    try:
        L_chol = torch.linalg.cholesky(A)
        X = torch.linalg.cho_solve((L_chol, True), M)
    except torch.linalg.LinAlgError:
        # Fallback to general solver if Cholesky fails (e.g., non-positive definite)
        X = torch.linalg.solve(A, M)
        
    # 5. Compute h_tilde
    # h_tilde = sum(Phi_i * X[:, i]) / sum(Phi_i * X[:, i + n_samples])
    # X[:, :n_samples] corresponds to (I+lambda L)^-1 W_i h
    # X[:, n_samples:] corresponds to (I+lambda L)^-1 W_i 1
    
    numerator = 0.0
    denominator = 0.0
    
    for i in range(n_samples):
        phi_i = Phi_weights[:, i] # (N,)
        x_h = X[:, i]             # (N,)
        x_1 = X[:, i + n_samples] # (N,)
        
        numerator += phi_i * x_h
        denominator += phi_i * x_1
        
    # Avoid division by zero
    denominator = torch.clamp(denominator, min=1e-8)
    
    h_tilde = numerator / denominator
    
    return h_tilde

def optimize_mesh(mesh, views, alpha, beta1, beta2, lambda_reg, sigma_phi, sigma_d, max_iters, upsample_iters=None):
    """
    Main optimization loop.
    """
    optimizer = torch.optim.Adam(mesh.vertices.parameters(), lr=alpha, betas=(beta1, beta2))
    
    # Learning rate scheduler
    scheduler = torch.optim.lr_scheduler.ExponentialLR(optimizer, gamma=0.6**(1/max_iters))
    
    for t in range(max_iters):
        # 1. Render views
        # Render forward pass (16 samples) and gradient pass (1 sample)
        # This depends on the renderer (Mitsuba 3)
        loss = compute_loss(mesh, views)
        
        # 2. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        
        # 3. Get gradients
        grad = mesh.vertices.grad
        
        # 4. Prefilter (Laplacian)
        # [S1] states Laplacian prefilter is important
        grad_prefiltered = torch.linalg.solve(torch.eye(N) + lambda_reg * L, grad)
        
        # 5. Update parameters (UniformAdam step)
        optimizer.step()
        
        # 6. Postfilter (Cross-Bilateral) on the update or state m_t
        # Note: The exact variable m_t is not explicitly defined in the pseudocode of Large Steps,
        # but it is the variable being filtered. In practice, this often means filtering the 
        # displacement or the vertex positions themselves after the optimizer step.
        # Assuming m_t corresponds to the vertex positions or the update vector.
        # Here we filter the vertex positions.
        mesh.vertices.data = compute_cross_bilateral_filter(
            mesh.vertices.data, 
            mesh.normals, 
            L, 
            lambda_reg, 
            sigma_phi, 
            sigma_d
        )
        
        # 7. Upsampling
        if upsample_iters and t in upsample_iters:
            mesh.remesh(edge_length_factor=0.5)
            
        scheduler.step()
        
    return mesh
```

## Subtopics

### Bilateral gradient filtering for meshes
This is the core filtering mechanism used in the method. It replaces the postfilter on $m_t$ with a cross-bilateral filter using vertex normals $\hat{n}$ for the data term. It samples $N=32$ directions $s_i \in S^2$ on the unit sphere using the Sobol sequence. It computes the filter efficiently by concatenating vectors $W(\hat{n}, s_i)h$ and $W(\hat{n}, s_i)\mathbf{1}$ into matrix $M = [W(\hat{n}, s_1)h \dots W(\hat{n}, s_N)h \mid W(\hat{n}, s_1)\mathbf{1} \dots W(\hat{n}, s_N)\mathbf{1}]$, then computing $(I + \lambda L)^{-1} M$ using a single matrix solve. The filtered result is $\tilde{h} = \frac{\sum_{i=1}^N \Phi(\hat{n}, s_i) (I+\lambda L)^{-1} W(\hat{n}, s_i) h}{\sum_{i=1}^N \Phi(\hat{n}, s_i) (I+\lambda L)^{-1} W(\hat{n}, s_i) \mathbf{1}}$. The diagonal elements of $W_i$ are computed using the von Mises-Fisher kernel $\exp(\hat{n} \cdot s_i / \sigma_d)$, and $\Phi_i$ using $\exp(\hat{n} \cdot s_i / \sigma_\phi)$ [S1].

## Operating characteristics
- **Time Complexity:** The cross-bilateral filter requires solving a linear system $(I + \lambda L)^{-1} M$. With $N$ vertices and $n$ samples, the matrix solve is $O(N^3)$ for dense Cholesky, but $L$ is sparse, allowing for efficient sparse solvers (e.g., CHOLMOD) [S1]. The overhead over Large Steps (which uses Laplacian smoothing) is minor (<10%) [S1].
- **Memory Complexity:** Requires storing the Laplacian matrix $L$ and the matrix $M$ of size $N \times 2n$.
- **Performance:** On an NVIDIA GeForce RTX 3080 GPU, the method converges faster than Large Steps on models with thin structures and many silhouettes (e.g., dragon) [S1].
- **Strengths:** Preserves sharp edges and silhouettes better than Laplacian smoothing. Faster convergence on thin structures. Low overhead.
- **Limitations:** Introduces two new hyperparameters ($\sigma_\phi, \sigma_d$) that require tuning. Assumes spatial coherence in parameters. Bilateral filtering on $v_t$ can cause small cracks on sharp edges [S1].
- **When to use:** When recovering meshes with thin structures, many silhouettes, or sharp edges where Laplacian smoothing dilutes gradients.
- **When not to use:** When parameters lack obvious spatial coherence. When computational resources are extremely limited and the overhead of the matrix solve is prohibitive (though it is minor).

## Background
The method was introduced in the paper "Spatiotemporal Bilateral Gradient Filtering for Inverse Rendering" [S1]. It builds on the Large Steps framework [S1], which uses Laplacian smoothing and an infinity norm as filters. The authors found that Laplacian preconditioning smooths gradients at silhouettes, causing them to be diluted, which is problematic for models with thin structures. The proposed method replaces the postfilter with a mesh cross-bilateral filter, inspired by the generalized bilateral filter from Solomon et al. [2014] [S1]. The implementation uses PyTorch [Paszke et al. 2019] and Mitsuba 3 [Jakob et al. 2022] [S1].

## Sources
[S1] Spatiotemporal Bilateral Gradient Filtering for Inverse Rendering, https://doi.org/10.1145/3680528.3687606, fetched 2026-10-29
