# Chambolle-Pock Algorithm

Also called: CP Algorithm, Chambolle-Pock Primal-Dual (CP) Algorithm, First-order Primal-Dual Algorithm, PSART Framework Solver

## Overview
The Chambolle-Pock (CP) algorithm is a first-order primal-dual optimization method used to solve composite convex optimization problems arising in tomographic reconstruction, specifically those formulated as minimizing a data fidelity term plus a non-smooth regularizer [S1, unofficial][S2, unofficial]. It sets itself apart from sibling methods by decoupling the complex objective into two independent proximal sub-problems: one for the data fidelity (often solved via a matrix-free iterative solver like SIRT) and one for the regularization (via its convex conjugate) [S1, unofficial][S2, unofficial]. This allows for efficient handling of large-scale sparse systems where direct inversion is infeasible, building on the [General Numerical Optimization Framework for Tomography](3251-general-numerical-optimization-framework-for-tomography.md).

## How it works
The algorithm solves the problem $\min_{\mathbf{f}} f(\mathbf{f}) + g(\mathbf{Kf})$ by introducing a dual variable and alternating updates between primal and dual spaces [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:**
    *   $\mathbf{K} \in \mathbb{R}^{d \times n}$: The linear operator (e.g., gradient or forward projection matrix) [S1, unofficial].
    *   $f(\mathbf{f})$: The data fidelity function (e.g., least-squares) [S1, unofficial].
    *   $g(\cdot)$: The regularization function [S1, unofficial].
    *   $\lambda > 0$: Step size for the primal update [S1, unofficial].
    *   $\mu > 0$: Step size for the dual update [S1, unofficial].
    *   $\theta \in [0, 1]$: Over-relaxation parameter for the primal extrapolation [S1, unofficial].
    *   $\sigma \in \mathbb{R}$: Scaling factor for the dual proximal operator (used with Moreau's decomposition) [S1, unofficial].
    *   $T$: Number of iterations.
    *   $\mathbf{x}^{(0)} \in \mathbb{R}^n$: Initial primal variable estimate [S1, unofficial].
    *   $\mathbf{z}^{(0)} \in \mathbb{R}^d$: Initial dual variable estimate [S1, unofficial].
*   **Outputs:**
    *   $\mathbf{x}^{(T)} \in \mathbb{R}^n$: The reconstructed volume estimate [S1, unofficial].

**Data Structures:**
*   $\mathbf{x}$: Vector of size $n$ (number of voxels).
*   $\mathbf{z}$: Vector of size $d$ (number of measurements/projections).
*   $\bar{\mathbf{x}}$: Vector of size $n$ for extrapolated primal variable.

**Definitions:**
*   **Proximal Operator for $f$**:
    $$ \text{prox}_{\lambda f}(\mathbf{u}) = \operatorname*{argmin}_{\mathbf{x}} f(\mathbf{x}) + \frac{1}{2\lambda} \|\mathbf{x} - \mathbf{u}\|_2^2 $$
    In tomography, this is typically solved using a matrix-free iterative solver such as [SIRT](3101-sirt-simultaneous-iterative-reconstruction-technique.md) [S1, unofficial][S2, unofficial].
*   **Proximal Operator for $g^*$ (Convex Conjugate)**:
    $$ \text{prox}_{\mu g^*}(\mathbf{u}) = \operatorname*{argmin}_{\mathbf{x}} g^*(\mathbf{x}) + \frac{1}{2\mu} \|\mathbf{x} - \mathbf{u}\|_2^2 $$
    where $g^*(\mathbf{u}) = \sup_{\mathbf{x}} \mathbf{u}^T \mathbf{x} - g(\mathbf{x})$ [S1, unofficial].
*   **Moreau's Decomposition**:
    $$ \text{prox}_{\mu g^*}(\mathbf{u}) = \mathbf{u} - \mu \, \text{prox}_{\frac{1}{\mu}g}\left(\frac{\mathbf{u}}{\mu}\right) $$
    This allows computing the dual proximal step using the proximal operator of the original function $g$ [S1, unofficial].
*   **Scaling Property**:
    $$ \text{prox}_{\mu g^*}(\mathbf{u}) = \sigma \, \text{prox}_{(\mu/\sigma)g^*}\left(\frac{\mathbf{u}}{\sigma}\right) $$
    This property is used to adapt the step size or norm constraints in the dual update [S1, unofficial].

**Constraints:**
The step sizes must satisfy:
$$ \mu \lambda \|\mathbf{K}\|^2 < 1 $$
where $\|\mathbf{K}\|$ is the spectral norm of the operator $\mathbf{K}$ [S1, unofficial][S2, unofficial].

**Algorithm Steps:**
1.  **Initialize**: Set $\bar{\mathbf{x}}^{(0)} = \mathbf{x}^{(0)}$.
2.  **Iterate** for $t = 1$ to $T$:
    a.  **Dual Update**: Compute the dual variable update using the proximal operator of the convex conjugate $g^*$:
        $$ \mathbf{z}^{(t+1)} = \text{prox}_{\mu g^*} \left( \mathbf{z}^{(t)} + \mu \mathbf{K} \bar{\mathbf{x}}^{(t)} \right) $$
        Using Moreau's decomposition and the scaling property, this can be implemented as:
        $$ \mathbf{z}^{(t+1)} = \sigma \, \text{prox}_{(\mu/\sigma)g^*} \left( \frac{\mathbf{z}^{(t)} + \mu \mathbf{K} \bar{\mathbf{x}}^{(t)}}{\sigma} \right) $$
        [S1, unofficial].
    b.  **Primal Update**: Compute the primal variable update using the proximal operator of the data fidelity $f$:
        $$ \mathbf{x}^{(t+1)} = \text{prox}_{\lambda f} \left( \mathbf{x}^{(t)} - \lambda \mathbf{K}^T \mathbf{z}^{(t+1)} \right) $$
        [S1, unofficial].
    c.  **Extrapolation**: Update the extrapolated primal variable:
        $$ \bar{\mathbf{x}}^{(t+1)} = \mathbf{x}^{(t+1)} + \theta (\mathbf{x}^{(t+1)} - \mathbf{x}^{(t)}) $$
        [S1, unofficial].
3.  **Return** $\mathbf{x}^{(T)}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chambolle_pock(K, prox_f, prox_g_star, lambda_step, mu_step, theta, sigma, T, x0, z0):
    """
    Solves min f(x) + g(Kx) using the Chambolle-Pock primal-dual algorithm.
    
    Args:
        K: Linear operator (function or matrix) mapping x to Kx.
        prox_f: Function computing prox_{lambda f}(u).
        prox_g_star: Function computing prox_{mu g*}(u).
        lambda_step: Step size for primal update (lambda).
        mu_step: Step size for dual update (mu).
        theta: Over-relaxation parameter.
        sigma: Scaling factor for dual proximal operator.
        T: Number of iterations.
        x0: Initial primal variable.
        z0: Initial dual variable.
        
    Returns:
        x: Reconstructed volume.
    """
    # Check convergence condition: mu * lambda * ||K||^2 < 1
    # Note: ||K|| must be computed or estimated beforehand.
    
    n = len(x0)
    x = x0.copy()
    z = z0.copy()
    x_bar = x0.copy()
    
    for t in range(1, T + 1):
        # Dual Update: z^(t+1) = prox_{mu g*} ( z^(t) + mu * K * x_bar^(t) )
        # Using scaling property: prox_{mu g*}(u) = sigma * prox_{(mu/sigma) g*}(u / sigma)
        # However, the source [S1, unofficial] defines prox_g_star as taking the step size mu implicitly 
        # or as a parameter. We assume prox_g_star(u, mu) computes prox_{mu g*}(u).
        # If prox_g_star is fixed to mu, we use it directly. 
        # Based on [S1, unofficial] Algorithm 3, it uses prox_{mu g*}.
        
        dual_input = z + mu_step * K(x_bar)
        z_new = prox_g_star(dual_input, mu_step)
        
        # Primal Update: x^(t+1) = prox_{lambda f} ( x^(t) - lambda * K^T * z^(t+1) )
        primal_input = x - lambda_step * K.T(z_new) # Assuming K.T is transpose operator
        x_new = prox_f(primal_input, lambda_step)
        
        # Extrapolation: x_bar^(t+1) = x^(t+1) + theta * (x^(t+1) - x^(t))
        x_bar_new = x_new + theta * (x_new - x)
        
        # Update variables
        z = z_new
        x = x_new
        x_bar = x_bar_new
        
    return x
```
*Note: The implementation of `prox_f` typically delegates to a solver like SIRT [S1, unofficial][S2, unofficial]. The implementation of `prox_g_star` depends on the specific regularizer $g$ (e.g., Total Variation) and uses Moreau's decomposition [S1, unofficial].*

**Edge Cases and Pitfalls:**
*   **Step Size Violation**: If $\mu \lambda \|\mathbf{K}\|^2 \ge 1$, the algorithm may diverge. The spectral norm $\|\mathbf{K}\|$ must be accurately estimated [S1, unofficial][S2, unofficial].
*   **Proximal Operator Implementation**: The efficiency of the algorithm relies heavily on the ability to compute $\text{prox}_f$ and $\text{prox}_{g^*}$ efficiently. For large-scale tomography, $\text{prox}_f$ is often approximated by a few iterations of SIRT [S1, unofficial][S2, unofficial].
*   **Regularizer Choice**: The choice of $g$ (e.g., ATV, ITV, STP) determines the structure of $\text{prox}_{g^*}$. Incorrect implementation of the proximal operator for non-standard regularizers (like STP) can lead to convergence issues [S2, unofficial].

## Subtopics

### [Proximal Operator](10110-proximal-operator.md)
The proximal operator is a mathematical tool used in proximal algorithms to solve non-smooth, constrained, or large-scale optimization problems by decomposing complex objective functions into simpler sub-problems. It generalizes projections onto convex sets, allowing an algorithm to move closer to the optimal solution while remaining close to the current estimate. In the context of tomographic reconstruction, it enables the handling of composite objectives, such as combining smooth data fidelity terms with non-smooth regularization (e.g., sparsity or total variation), which are otherwise difficult to minimize jointly. See [Proximal Operator](10110-proximal-operator.md).

### Primal-Dual Proximal Algorithm for STP Reconstruction
An iterative algorithm to solve the reconstruction problem $\min_f \|Af - p\|_2^2 + \lambda STP_p(f)$ by converting it into a saddle-point problem and solving it using a primal-dual Chambolle-Pock (CP) algorithm [S2, unofficial]. It initializes $\bar{f}^{(0)} = 0$, solves for the primal variable $f^{t+1}$ by applying the proximal operator of the data term to $f^t - \tau J_K^* Y^{t+1}$ with step size $\tau$ (using a SART solver), updates the extrapolated variable $\bar{f}^{t+1}$ using heavy-ball momentum, and updates the dual variable $Y^{t+1}$ using the proximal operator of the convex conjugate of the STP regularizer (projection onto a norm ball) [S2, unofficial]. The problem is reformulated as a saddle-point problem: $\min_f \max_H \|Af - p\|_2^2 + \langle H, J_K f \rangle - \iota_{\lambda B_{\infty, q}}(H)$, where $\iota$ is the indicator function of the norm ball $\lambda B_{\infty, q}$ [S2, unofficial]. Inputs include $\lambda$ (regularization weight), $\eta$ (step size for dual variable), $\tau$ (step size for primal variable), $\theta$ (extrapolation parameter), $p$ (Schatten norm parameter), and $l$ (neighborhood size) [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Each iteration involves matrix-vector multiplications with $\mathbf{K}$ and $\mathbf{K}^T$, and two proximal operator evaluations. The complexity per iteration is dominated by the cost of $\text{prox}_f$ (often SIRT, which is $O(MN)$ per iteration if dense, but faster with sparse $W$) and $\text{prox}_{g^*}$ (often $O(N)$ for TV-like regularizers). The number of iterations $T$ required for convergence depends on the condition number and step sizes.
*   **Memory Complexity**: $O(N + D)$ for storing the primal and dual variables, plus storage for the operator $\mathbf{K}$ (often implicit/matrix-free).
*   **Measured Performance**: The CP algorithm is noted for its robustness and ability to handle large-scale problems where second-order methods are too expensive [S1, unofficial][S2, unofficial]. It converges faster than standard gradient descent for non-smooth regularizers [S2, unofficial].
*   **Strengths**: Handles non-smooth regularizers (e.g., L1, Total Variation) efficiently; decouples data fidelity and regularization; robust to noise; suitable for large-scale tomographic data.
*   **Limitations**: Convergence rate can be slow for ill-conditioned problems; requires careful tuning of step sizes $\lambda$ and $\mu$; the proximal operator for the data term may require an inner iterative solver (like SIRT), adding computational overhead [S1, unofficial][S2, unofficial].
*   **When to Use**: When the objective function is composite (smooth data term + non-smooth regularizer) and the problem size is too large for direct methods. Particularly effective in tomographic reconstruction with sparsity or total variation priors.
*   **When Not to Use**: When the problem is small enough for direct solvers, or when the regularizer is smooth and differentiable (gradient-based methods may be simpler). Also not suitable if the proximal operators cannot be computed efficiently.

## Background
The Chambolle-Pock algorithm was proposed by Antonin Chambolle and Thomas Pock in 2011 [S1, unofficial][S2, unofficial]. It is a first-order primal-dual algorithm designed for convex optimization problems. In the context of tomography, it has been adopted as a key solver within proximal frameworks, such as the PSART (Proximal SART) framework, to handle complex regularizers like Total Variation and Space-Time Tomographic Prior (STP) [S1, unofficial][S2, unofficial]. The algorithm leverages Moreau's decomposition to compute the dual proximal operator, enabling the use of various regularizers without deriving new dual solvers from scratch [S1, unofficial][S2, unofficial].

## Sources
[S1] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
[S2] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
