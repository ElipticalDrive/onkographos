# Chambolle-Pock Algorithm

Also called: CP Algorithm, Chambolle-Pock Primal-Dual (CP) Algorithm, First-order Primal-Dual Algorithm, PSART Framework Solver

## Overview
The Chambolle-Pock (CP) algorithm is a first-order primal-dual optimization method used to solve composite convex optimization problems of the form $\min_{\mathbf{f}} f(\mathbf{f}) + g(\mathbf{Kf})$, which is the standard formulation for regularized tomographic reconstruction [S1, S2, unofficial]. It solves the problem by alternating between updates of a primal variable $\mathbf{f}$ (the volume estimate) and a dual variable $\mathbf{z}$ (associated with the regularizer), utilizing proximal operators for the data fidelity term $f$ and the convex conjugate $g^*$ of the regularization term $g$ [S1, S2, unofficial]. It is distinguished from its siblings, [Coupled Density and Velocity Estimation](4622-coupled-density-and-velocity-estimation.md) and [Stochastic Tomography](8886-stochastic-tomography.md), by its deterministic, first-order nature and its specific reliance on the Moreau decomposition to handle non-smooth regularization terms efficiently within a primal-dual framework [S1, S2, unofficial].

## How it works
The algorithm minimizes the objective function defined in the parent [General Numerical Optimization Framework for Tomography](3251-general-numerical-optimization-framework-for-tomography.md). It requires the implementation of two specific proximal operators: one for the data term $f(\mathbf{f})$ and one for the convex conjugate $g^*(\cdot)$ of the regularization term $g(\cdot)$ [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   $\mathbf{K} \in \mathbb{R}^{d \times n}$: The linear operator (e.g., forward projection matrix or gradient operator), where $d$ is the number of measurements and $n$ is the number of voxels [S1, unofficial].
    *   $\mathbf{p} \in \mathbb{R}^d$: The observed sinogram or projection data [S1, unofficial].
    *   $\tau > 0$: Step size for the primal variable update [S1, S2, unofficial].
    *   $\mu > 0$: Step size for the dual variable update [S1, S2, unofficial].
    *   $\theta \in [0, 1]$: Over-relaxation (extrapolation) parameter [S1, S2, unofficial].
    *   $\sigma \in \mathbb{R}$: Scaling parameter for the dual proximal operator, used in the scaling property of the proximal operator [S1, unofficial].
    *   $T$: Total number of iterations [S1, S2, unofficial].
*   **Outputs**:
    *   $\mathbf{f}^{(T)} \in \mathbb{R}^n$: The final reconstructed volume estimate [S1, S2, unofficial].

### Data Structures
*   $\mathbf{x}^{(t)} \in \mathbb{R}^n$: Primal variable at iteration $t$ [S1, unofficial].
*   $\mathbf{z}^{(t)} \in \mathbb{R}^d$: Dual variable at iteration $t$ [S1, unofficial].
*   $\bar{\mathbf{x}}^{(t)} \in \mathbb{R}^n$: Extrapolated primal variable at iteration $t$ [S1, unofficial].

### Proximal Operators
The algorithm relies on the definition of the proximal operator for a function $h$ and parameter $\zeta$:
$$
\text{prox}_{\zeta h}(\mathbf{q}) \triangleq \operatorname*{argmin}_{\mathbf{f}} h(\mathbf{f}) + \frac{1}{2\zeta} \|\mathbf{f} - \mathbf{q}\|_2^2
$$
where $\mathbf{q}$ is the input vector and $\zeta$ is the weighting parameter [S1, S2, unofficial].

1.  **Data Fidelity Proximal Operator**: $\text{prox}_{\tau f}(\mathbf{q})$. For the least-squares data term $f(\mathbf{f}) = \|\mathbf{Kf} - \mathbf{p}\|_2^2$, this operator is typically solved using a matrix-free iterative solver such as [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md) because it converges better than Conjugate Gradients for sparse projection matrices [S1, unofficial].
2.  **Regularization Conjugate Proximal Operator**: $\text{prox}_{\mu g^*}(\mathbf{q})$. The convex conjugate is defined as $g^*(\mathbf{u}) = \sup_{\mathbf{x}} \mathbf{u}^T \mathbf{x} - g(\mathbf{x})$ [S1, unofficial]. Computing this directly is often difficult. Instead, the algorithm uses **Moreau's Decomposition** [S1, unofficial]:
    $$
    \text{prox}_{\mu g^*}(\mathbf{u}) = \mathbf{u} - \mu \, \text{prox}_{\frac{1}{\mu}g}\left(\frac{\mathbf{u}}{\mu}\right)
    $$
    Additionally, a scaling property is used to simplify the dual update [S1, unofficial]:
    $$
    \text{prox}_{\mu g^*}(\mathbf{u}) = \sigma \, \text{prox}_{(\mu/\sigma)g^*}\left(\frac{\mathbf{u}}{\sigma}\right)
    $$
    This allows the dual step to be computed using the proximal operator of the original regularization function $g$ (e.g., soft-thresholding for L1 norms) [S1, unofficial].

### Algorithm Steps
The stability condition requires that the step sizes satisfy $\mu \lambda \| \mathbf{K} \|^2 < 1$, where $\lambda$ corresponds to the primal step size $\tau$ in the notation of [S1, unofficial] (i.e., $\mu \tau \| \mathbf{K} \|^2 < 1$) [S1, S2, unofficial]. The spectral norm $\| \mathbf{K} \|$ must be calculated or estimated (e.g., via power iteration) [S1, S2, unofficial].

1.  **Initialize**: Set $\bar{\mathbf{x}}^{(0)} = \mathbf{x}^{(0)}$ and $\mathbf{z}^{(0)} = \mathbf{0}$ (or other initial values) [S1, S2, unofficial].
2.  **Iterate** for $t = 1$ to $T$:
    a.  **Dual Update**: Compute the dual variable using the proximal operator of the convex conjugate $g^*$:
        $$
        \mathbf{z}^{(t+1)} = \text{prox}_{\mu g^*} \left( \mathbf{z}^{(t)} + \mu \mathbf{K} \bar{\mathbf{x}}^{(t)} \right)
        $$
        Using the scaling property, this is often implemented as:
        $$
        \mathbf{z}^{(t+1)} = \sigma \, \text{prox}_{(\mu/\sigma)g^*} \left( \frac{\mathbf{z}^{(t)} + \mu \mathbf{K} \bar{\mathbf{x}}^{(t)}}{\sigma} \right)
        $$
        [S1, unofficial].
    b.  **Primal Update**: Compute the primal variable using the proximal operator of the data term $f$:
        $$
        \mathbf{x}^{(t+1)} = \text{prox}_{\tau f} \left( \mathbf{x}^{(t)} - \tau \mathbf{K}^T \mathbf{z}^{(t+1)} \right)
        $$
        [S1, S2, unofficial].
    c.  **Extrapolation**: Update the extrapolated primal variable using the heavy-ball momentum formula:
        $$
        \bar{\mathbf{x}}^{(t+1)} = \mathbf{x}^{(t+1)} + \theta (\mathbf{x}^{(t+1)} - \mathbf{x}^{(t)})
        $$
        [S1, S2, unofficial].
3.  **Return**: $\mathbf{x}^{(T)}$ [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chambolle_pock(K, K_T, prox_f, prox_g_star, p, tau, mu, theta, sigma, T, x_init=None):
    """
    Chambolle-Pock Primal-Dual Algorithm for Tomographic Reconstruction.
    
    Args:
        K: Linear operator function K(x) returning K @ x.
        K_T: Linear operator function K_T(z) returning K^T @ z.
        prox_f: Function implementing prox_tau_f(q).
        prox_g_star: Function implementing prox_mu_g_star(q) using Moreau's decomposition 
                      or direct definition.
        p: Observed sinogram data.
        tau: Primal step size.
        mu: Dual step size.
        theta: Extrapolation parameter (typically 1.0).
        sigma: Scaling parameter for dual proximal operator.
        T: Number of iterations.
        x_init: Initial volume estimate. Defaults to zeros.
        
    Returns:
        Reconstructed volume f.
    """
    # Determine dimensions
    # n is number of voxels, d is number of measurements
    # These are typically inferred from K or p
    n = K_T(p).shape[0] # Placeholder for dimension inference; in practice, n is known from grid
    d = p.shape[0]
    
    if x_init is None:
        x = [0.0] * n
    else:
        x = list(x_init)
        
    z = [0.0] * d
    x_bar = list(x)
    
    for t in range(1, T + 1):
        # 1. Dual Update
        # Compute z_t + mu * K(x_bar_t)
        k_x_bar = K(x_bar)
        z_input = [z[i] + mu * k_x_bar[i] for i in range(d)]
        
        # Apply proximal operator for g*
        # Using scaling property: prox_mu_g*(u) = sigma * prox_(mu/sigma)_g*(u/sigma)
        # Note: The source [S1, unofficial] defines prox_g_star as the operator for the conjugate.
        # In practice, this is often computed via Moreau's decomposition:
        # prox_mu_g*(u) = u - mu * prox_(1/mu)_g(u/mu)
        # However, the pseudocode below follows the direct call to prox_g_star 
        # as defined in the algorithm description [S1, unofficial].
        z_new = prox_g_star(z_input, mu) # Implementation detail depends on g
        
        # If using the scaling property explicitly as in [S1, unofficial]:
        # z_scaled_input = [z_input[i] / sigma for i in range(d)]
        # z_new = [sigma * prox_g_star_scaled(z_scaled_input[i], mu/sigma) for i in range(d)]
        
        z = z_new
        
        # 2. Primal Update
        # Compute x_t - tau * K_T(z_{t+1})
        k_t_z = K_T(z)
        x_input = [x[i] - tau * k_t_z[i] for i in range(n)]
        
        # Apply proximal operator for f
        # For f(f) = ||Kf - p||^2, this is solved via SIRT or similar [S1, unofficial]
        x_new = prox_f(x_input, tau, p)
        
        x = x_new
        
        # 3. Extrapolation
        x_bar_new = [x[i] + theta * (x[i] - x_prev[i]) for i in range(n)]
        
        # Store previous x for next iteration's extrapolation
        x_prev = x
        x_bar = x_bar_new
        
    return x
```
*Note: The `prox_f` and `prox_g_star` functions are implementation-specific. `prox_f` typically calls an inner solver like SIRT [S1, unofficial]. `prox_g_star` typically uses Moreau's decomposition with the proximal operator of $g$ (e.g., soft-thresholding for TV) [S1, unofficial].*

### Edge Cases and Pitfalls
*   **Step Size Stability**: The condition $\mu \tau \| \mathbf{K} \|^2 < 1$ is critical for convergence. If $\| \mathbf{K} \|$ is not accurately estimated, the algorithm may diverge [S1, S2, unofficial].
*   **Precision**: The use of extrapolation ($\theta$) can accelerate convergence but may introduce oscillations if $\theta$ is too large or if the problem is ill-conditioned [S1, S2, unofficial].
*   **Sparse Operators**: The forward and adjoint operators $\mathbf{K}$ and $\mathbf{K}^T$ must be implemented efficiently, as they are called in every iteration [S1, S2, unofficial].

## Subtopics

### [Proximal Operator](10110-proximal-operator.md)
The proximal operator is a mathematical tool used in proximal algorithms to solve non-smooth, constrained, or large-scale optimization problems by iteratively getting closer to the optimal solution while staying close to the current estimate. It serves as a generalization of projections onto convex sets, allowing complex optimization problems to be split into smaller, simpler sub-problems. In the context of tomographic reconstruction, it enables the handling of regularization terms (such as sparsity or smoothing norms) that often lack closed-form solutions in standard gradient-based methods.

### Primal-Dual Proximal Algorithm for STP Reconstruction
An iterative algorithm to solve the reconstruction problem $\min_f \|Af - p\|_2^2 + \lambda STP_p(f)$ by converting it into a saddle-point problem and solving it using a primal-dual Chambolle-Pock (CP) algorithm [S2, unofficial]. It initializes $\bar{f}^{(0)} = 0$, updates the primal variable $f^{t+1}$ by applying the proximal operator of the data term to $f^t - \tau J_K^* Y^{t+1}$ (using a SART solver), and updates the extrapolated variable $\bar{f}^{t+1}$ using heavy-ball momentum [S2, unofficial]. The problem is reformulated as a saddle-point problem involving an indicator function of a norm ball [S2, unofficial]. Inputs include regularization weight $\lambda$, step sizes $\eta, \tau$, extrapolation $\theta$, and STP parameters $p, l$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Each iteration involves two matrix-vector multiplications ($\mathbf{K}$ and $\mathbf{K}^T$) and two proximal operator evaluations. The complexity per iteration is dominated by the forward and adjoint projections, which are $O(N)$ for sparse matrices, where $N$ is the number of non-zero entries in $\mathbf{K}$ [S1, S2, unofficial].
*   **Memory Complexity**: $O(n + d)$ to store the primal and dual variables, plus the storage for the sparse matrix $\mathbf{K}$ [S1, S2, unofficial].
*   **Performance**: The algorithm is robust for large-scale problems due to its first-order nature and matrix-free implementation capabilities [S1, S2, unofficial]. Convergence is linear for strongly convex problems and sublinear for general convex problems [S1, unofficial].
*   **Strengths**: Handles non-smooth regularization (e.g., Total Variation) efficiently via proximal operators; suitable for large-scale tomographic data; flexible framework for different priors [S1, S2, unofficial].
*   **Limitations**: Requires careful tuning of step sizes $\tau$ and $\mu$; convergence can be slow for ill-conditioned problems; the proximal operator for the data term may require an inner iterative solver (e.g., SIRT) which adds computational overhead [S1, unofficial].
*   **When to Use**: Suitable for tomographic reconstruction with non-smooth regularization (TV, L1) where first-order methods are preferred over second-order methods due to memory constraints [S1, S2, unofficial].
*   **When Not to Use**: Not suitable for problems requiring high-accuracy solutions in very few iterations where second-order methods (like Newton's method) might be more efficient, or when the proximal operators are too expensive to compute [S1, unofficial].

## Background
The Chambolle-Pock algorithm was proposed by Antonin Chambolle and Patrick Pock in 2011 [S1, S2, unofficial]. It is a first-order primal-dual algorithm designed to solve convex optimization problems of the form $\min_x f(x) + g(Kx)$ [S1, S2, unofficial]. The algorithm has been widely adopted in image processing and tomographic reconstruction due to its ability to handle non-smooth regularization terms efficiently [S1, S2, unofficial]. It is often used within frameworks such as PSART (Proximal SART) to solve inverse problems in computed tomography [S1, S2, unofficial].

## Sources
[S1] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
[S2] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
