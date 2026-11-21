# Magnification-driven B-spline Interpolation

Also called: B-spline homography/projector/backprojector, Least-square resampling of f(x) on a basis of non-uniform B-splines, Magnification-driven interpolation, Non-uniform B-spline projection and backprojection

## Overview
Magnification-driven B-spline interpolation is a spatial resampling technique for homography, projection, and backprojection operations that adapts to local magnification factors to reduce edge artifacts and aliasing compared to linear interpolation [S1]. It solves the problem of nonstationarity in linear interpolation, which causes missing features during homography and artificial high-frequency patterns in adjoint operations [S1]. It sets itself apart from standard B-spline interpolation by explicitly incorporating the local Jacobian of the transformation into the sampling step and the correlation functions, ensuring that the interpolation kernel adapts to the local density of the source or detector grid [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   Source data vector $\mathbf{c}$ (values on source grid points $x_j$).
*   Transformation function $h(\cdot)$ mapping source coordinates to detector/projector coordinates $u$.
*   Transformation inverse $h^{-1}(\cdot)$ mapping detector coordinates to source coordinates.
*   B-spline orders: $n$ for the projection stage (source space), $m$ for the backprojection stage (detector space) [S1].
*   Grid indices: $I$ for detector/projector samples, $J$ for source samples [S1].

**Outputs:**
*   Projection vector $\mathbf{p}$ (values on detector grid $u_i$).
*   Backprojection vector $\mathbf{f}$ (values on source grid $x_j$) [S1].

### Data Structures
*   **Vectors:** $\mathbf{c}$, $\mathbf{p}$, $\mathbf{f}$, $\boldsymbol{\Delta}$, $\boldsymbol{\Delta}'$, $\mathbf{F}$, $\mathbf{F}'$.
*   **Matrices:** $\mathbf{G}$ (Gram matrix), $\tilde{\mathbf{G}}$ (surrogate Gram matrix), $\boldsymbol{\Lambda}$, $\tilde{\boldsymbol{\Lambda}}$, $\mathbf{T}_n$, $\mathbf{T}_m$ (Toeplitz matrices).
*   **Diagonal Matrices:** $\text{Diag}(\mathbf{v})$ creates a diagonal matrix from vector $\mathbf{v}$ [S1].

### Definitions and Formulas
1.  **Sampling Steps (Magnification):**
    *   For projection, the sampling step at detector index $i$ is $\Delta_i = |h^{-1}(u_i)'|$ [S1].
    *   For backprojection, the sampling step at source index $j$ is $\Delta'_j = |h'(x_j)|$ [S1].
    *   For low-order splines ($n=m=0$), $\Delta_i$ can be derived from edge locations: $\delta_1 = \delta_2 = h^{-1}(u_2) - h^{-1}(u_1)$, and $h^{-1}(u_{i+1}) - h^{-1}(u_i) = (\delta_{i+1} + \delta_i)/2$ for $i \ge 2$ [S1].

2.  **B-spline Basis Functions:**
    *   Order 0: $\beta^0(\tau) = 1$ if $\tau < 1/2$, else $0$ [S1].
    *   Order 1: $\beta^1(\tau) = 1 - \tau$ if $\tau < 1$, else $0$ [S1].
    *   General B-spline of order $k$ is denoted $b^k$ [S1].

3.  **Correlation Functions $\xi^{n,m}_\Delta(\theta)$:**
    These functions represent the inner product of scaled B-splines. The scaling property is:
    $$ (1/\Delta)\xi_{\Delta}^{m,n}(\theta) = \xi_{1/\Delta}^{n,m}(\theta/\Delta) \quad (49) $$
    *   **Case $\xi^{0,0}_\Delta(\theta)$:**
        $$ \xi_{\Delta}^{0,0}(\theta) = \begin{cases} \min(1, \Delta) & \text{if } |\theta| < a_1 \\ a_2 - |\theta| & \text{if } a_1 \leq |\theta| < a_2 \\ 0 & \text{if } |\theta| \geq a_2 \end{cases} $$
        where $a_1 = |\Delta-1|/2$ and $a_2 = (\Delta+1)/2$ [S1].
    *   **Case $\xi^{1,0}_\Delta(\theta)$:**
        Piecewise quadratic: $c_{k,0} + c_{k,1}|\theta| + c_{k,2}|\theta|^2$ for $|\theta| \in [a_{k-1}, a_k)$.
        $a_0 = 0$, $a_1 = |\Delta/2 - 1|$, $a_2 = \Delta/2 + 1$ [S1].
        Coefficients $c_{k,r}$ are defined in Table 1 below [S1].

    **Table 1: Parameters for $\xi^{1,0}_\Delta$**
    | Interval | Condition | $c_{k,0}$ | $c_{k,1}$ | $c_{k,2}$ |
    | :--- | :--- | :--- | :--- | :--- |
    | $\theta < a_1$ | $\Delta \le 2, \theta < \Delta/2$ | $\Delta - \Delta^2/4$ | $0$ | $-1$ |
    | $\theta < a_1$ | $\Delta \le 2, \theta \ge \Delta/2$ | $\Delta$ | $-\Delta$ | $0$ |
    | $\theta < a_1$ | $\Delta > 2$ | $1$ | $0$ | $0$ |
    | $a_1 \le \theta < a_2$ | $\theta \ge \Delta/2$ | $(\Delta^2+4\Delta+4)/8$ | $-1-\Delta/2$ | $1/2$ |
    | $a_1 \le \theta < a_2$ | $\theta < \Delta/2$ | $(-\Delta^2+4\Delta+4)/8$ | $-1+\Delta/2$ | $-1/2$ |

4.  **Gram Matrix Diagonal:**
    $$ G_{ii} = \Delta_i \xi_1^{n,n}(0) = \Delta_i b^{2n+1}(0) \quad (30) $$
    The full Gram matrix element is:
    $$ G_{i,l} = \Delta_i \xi_{\Delta_l/\Delta_i}^{n,n} \left( \frac{h^{-1}(u_l) - h^{-1}(u_i)}{\Delta_i} \right) \quad (29) $$

5.  **Footprint Matrix $\mathbf{F}$:**
    Elements $F_{i,j}$ involve the correlation function between source and detector bases. Specifically, for projection:
    $$ F_{i,j} = \xi_{\Delta_i}^{n,m}(h(x_j) - h^{-1}(u_i)) \quad \text{(derived from context of Eq 28 and 33)} $$
    *Note: The source text defines $F'_{j,i} = \xi_{\Delta'_j}^{n,m}(u_i - h(x_j))$ for backprojection [S1]. For projection, the structure is symmetric based on the model definition.*

### Algorithm Steps

#### Projection Stage (Source $\mathbf{c}$ to Detector $\mathbf{p}$)

1.  **Compute Sampling Steps:** Calculate $\Delta_i = |h^{-1}(u_i)'|$ for all $i \in I$ [S1].
2.  **Construct Footprint Matrix $\mathbf{F}$:** Compute $F_{i,j}$ using $\xi^{n,m}$ based on the relative positions $h(x_j)$ and $h^{-1}(u_i)$ scaled by $\Delta_i$ [S1].
3.  **Approximate Coefficients $\tilde{\mathbf{s}}^*_{\boldsymbol{\Delta}}$:**
    To avoid inverting the non-Toeplitz Gram matrix $\mathbf{G}$, use the surrogate $\tilde{\mathbf{G}}$.
    *   Define surrogate $\tilde{\mathbf{G}}$ such that $\tilde{G}_{i,l} = \sqrt{\Delta_i \Delta_l} b^{2n+1}(l-i)$ if $|l-i| \le n$, else $0$ [S1].
    *   This allows $\tilde{\mathbf{G}} = \text{Diag}(\boldsymbol{\Delta})^{1/2} \mathbf{T}_n \text{Diag}(\boldsymbol{\Delta})^{1/2}$, where $\mathbf{T}_n$ is Toeplitz [S1].
    *   Compute $\tilde{\mathbf{s}}^*_{\boldsymbol{\Delta}} = \text{Diag}(\boldsymbol{\Delta})^{-1/2} \mathbf{T}_n^{-1} \text{Diag}(\boldsymbol{\Delta})^{-1/2} \mathbf{F} \mathbf{c}$ [S1].
    *   Implementation of $\mathbf{T}_n^{-1}$ is done via filtering with $(b^{2n+1})^{-1}$ [S1].
4.  **Compute Projection Vector $\mathbf{p}$:**
    *   Compute intermediate vector $\tilde{\mathbf{p}}$ using discrete convolution:
        $$ \tilde{p}_i = \Delta_i (b^n * \tilde{\mathbf{s}}^*_{\boldsymbol{\Delta}})(i) \quad (46) $$
    *   Alternatively, using the full matrix form if precision requires:
        $$ \mathbf{p} = \text{Diag}(\boldsymbol{\Delta}) \boldsymbol{\Lambda} \boldsymbol{\Gamma}^{-1} \mathbf{F} \mathbf{c} \quad (33) $$
        where $\boldsymbol{\Lambda}_{i,l} = \beta_{\Delta_i}(h^{-1}(u_l) - h^{-1}(u_i))$ [S1].

#### Backprojection Stage (Detector $\mathbf{p}$ to Source $\mathbf{f}$)

1.  **Define Detector Coefficients:** Assume $p(u) = \sum_{i \in I} c'(i) \beta^n(u - u_i)$ [S1]. Here $\mathbf{c}'$ corresponds to the input projection data (potentially pre-filtered or raw depending on the specific pipeline stage, but structurally treated as coefficients in the B-spline basis) [S1].
2.  **Compute Sampling Steps:** Calculate $\Delta'_j = |h'(x_j)|$ for all $j \in J$ [S1].
3.  **Construct Footprint Matrix $\mathbf{F}'$:**
    $$ F'_{j,i} = \xi_{\Delta'_j}^{n,m}(u_i - h(x_j)) \quad (38) $$
4.  **Approximate Backprojection Vector $\tilde{\mathbf{f}}$:**
    *   Define surrogate $\tilde{\mathbf{G}}'$ such that $\tilde{G}'_{j,l} = \sqrt{\Delta'_j \Delta'_l} b^{2m+1}(l-j)$ if $|l-j| \le m$, else $0$ [S1].
    *   Compute:
        $$ \tilde{\mathbf{f}} = \text{Diag}(\boldsymbol{\Delta}') \tilde{\boldsymbol{\Lambda}}' \text{Diag}(\boldsymbol{\Delta}')^{-1/2} \mathbf{T}_m^{-1} \text{Diag}(\boldsymbol{\Delta}')^{-1/2} \mathbf{F}' \mathbf{c}' \quad (48) $$
    *   Where $\tilde{\Lambda}'_{j,l} = \beta_{\Delta'_l}^m (h(x_l) - h(x_j))$ [S1].
5.  **Final Scaling:**
    The reconstructed value is $f(x_j) = \Delta'_j p(h(x_j))$ [S1]. The vector $\mathbf{f}$ computed in step 4 approximates this.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def b_spline_basis(order, tau):
    """
    Computes the B-spline basis function of given order at tau.
    Order 0: 1 if tau < 0.5 else 0
    Order 1: 1 - tau if tau < 1 else 0
    Higher orders can be computed via convolution or Cox-de Boor recursion.
    """
    if order == 0:
        return 1.0 if tau < 0.5 else 0.0
    elif order == 1:
        return max(0.0, 1.0 - tau)
    else:
        # Placeholder for higher order B-spline computation
        # In practice, use precomputed kernel or recursive definition
        raise NotImplementedError("Higher order B-spline basis not implemented in this snippet")

def xi_00_delta(theta, delta):
    """
    Computes xi^{0,0}_Delta(theta)
    """
    a1 = abs(delta - 1) / 2.0
    a2 = (delta + 1) / 2.0
    
    if abs(theta) < a1:
        return min(1.0, delta)
    elif a1 <= abs(theta) < a2:
        return a2 - abs(theta)
    else:
        return 0.0

def xi_10_delta(theta, delta):
    """
    Computes xi^{1,0}_Delta(theta) using Table 1 parameters.
    """
    a0 = 0.0
    a1 = abs(delta / 2.0 - 1.0)
    a2 = delta / 2.0 + 1.0
    
    abs_theta = abs(theta)
    
    # Determine interval k
    if abs_theta < a1:
        k = 1
    elif abs_theta < a2:
        k = 2
    else:
        return 0.0
        
    # Determine coefficients based on conditions
    if k == 1:
        if delta <= 2:
            if theta < delta / 2.0:
                c0 = delta - delta**2 / 4.0
                c1 = 0.0
                c2 = -1.0
            else:
                c0 = delta
                c1 = -delta
                c2 = 0.0
        else: # delta > 2
            c0 = 1.0
            c1 = 0.0
            c2 = 0.0
    else: # k == 2
        if theta >= delta / 2.0:
            c0 = (delta**2 + 4*delta + 4) / 8.0
            c1 = -1.0 - delta / 2.0
            c2 = 0.5
        else:
            c0 = (-delta**2 + 4*delta + 4) / 8.0
            c1 = -1.0 + delta / 2.0
            c2 = -0.5
            
    return c0 + c1 * abs_theta + c2 * abs_theta**2

def compute_xi(n, m, theta, delta):
    """
    Computes xi^{n,m}_Delta(theta).
    Uses symmetry property: (1/delta)xi^{m,n}_Delta(theta) = xi^{n,m}_{1/delta}(theta/delta)
    For simplicity, this example implements n=0,m=0 and n=1,m=0 directly.
    """
    if n == 0 and m == 0:
        return xi_00_delta(theta, delta)
    elif n == 1 and m == 0:
        return xi_10_delta(theta, delta)
    else:
        # Use symmetry for other cases if needed
        # xi^{n,m}_Delta(theta) = delta * xi^{m,n}_{1/delta}(theta/delta)
        return delta * compute_xi(m, n, theta / delta, 1.0 / delta)

def magnification_driven_projection(c, h_inv, h_inv_prime, u_indices, n):
    """
    Projects source data c to detector space using magnification-driven B-splines.
    c: Source values
    h_inv: Function returning h^{-1}(u)
    h_inv_prime: Function returning derivative of h^{-1}
    u_indices: Array of detector coordinates u_i
    n: B-spline order for projection
    """
    N_I = len(u_indices)
    Delta = np.array([abs(h_inv_prime(u)) for u in u_indices])
    
    # Construct Footprint Matrix F
    # F_{i,j} involves xi^{n,m} where m is source order (assumed same as n for simplicity in this snippet, 
    # or m is defined by the source representation). 
    # Note: The source text implies F connects source basis (order m) to detector basis (order n).
    # Assuming source is also represented by B-splines of order m.
    # For this snippet, we assume m=n for the footprint calculation structure, 
    # but strictly F_{i,j} = xi^{n,m}_{Delta_i}(h(x_j) - h^{-1}(u_i))
    
    # Placeholder for F construction
    # F = np.zeros((N_I, N_J))
    # for i in range(N_I):
    #     for j in range(N_J):
    #         theta = h(x_j) - h_inv(u_indices[i])
    #         F[i,j] = compute_xi(n, m, theta, Delta[i])
        
    # Approximate Inversion using Surrogate Gram Matrix
    # T_n is Toeplitz with first row/column derived from b^{2n+1}
    # s_star = Diag(Delta)^{-1/2} T_n^{-1} Diag(Delta)^{-1/2} F c
    
    # Compute p = Diag(Delta) Lambda s_star
    # p_i = Delta_i (b^n * s_star)(i)
    
    raise NotImplementedError("Full matrix construction omitted for brevity. See formulas (33)-(46).")

def magnification_driven_backprojection(p, h, h_prime, x_indices, n, m):
    """
    Backprojects detector data p to source space.
    p: Detector values (treated as coefficients c' in B-spline basis)
    h: Function returning h(x)
    h_prime: Function returning derivative of h
    x_indices: Array of source coordinates x_j
    n: B-spline order for projection (detector basis)
    m: B-spline order for backprojection (source basis)
    """
    N_J = len(x_indices)
    Delta_prime = np.array([abs(h_prime(x)) for x in x_indices])
    
    # Construct Footprint Matrix F'
    # F'_{j,i} = xi^{n,m}_{Delta'_j}(u_i - h(x_j))
    
    # Approximate Backprojection
    # f = Diag(Delta') Lambda' Diag(Delta')^{-1/2} T_m^{-1} Diag(Delta')^{-1/2} F' c'
    
    raise NotImplementedError("Full matrix construction omitted for brevity. See formulas (37)-(48).")
```

### Edge Cases and Pitfalls
*   **Non-Toeplitz Gram Matrix:** The exact Gram matrix $\mathbf{G}$ is not Toeplitz due to non-uniform sampling, making direct inversion expensive ($O(N^3)$) [S1]. The surrogate approximation is necessary for performance [S1].
*   **Linear Interpolation Artifacts:** Linear interpolation ($H_r$) exhibits nonstationarity, causing missing wires during homography and artificial high-frequency patterns in adjoint operations on constant images [S1].
*   **Noise Correlation:** Higher-order models ($s > 1$) yield less distortion but introduce small undershoots and higher noise correlation patterns [S1].
*   **Zero-Order Patchiness:** Zero-order B-splines ($s=1$) produce a patchy look [S1].

## Operating characteristics
*   **Time Complexity:** The surrogate method reduces complexity by allowing the use of Toeplitz matrix inversion (filtering) instead of full matrix inversion. Implementations b and c run 1.15× and 1.4× faster than the baseline implementation a for tested homographies [S1].
*   **Memory Complexity:** Requires storage of the footprint matrices $\mathbf{F}$ and $\mathbf{F}'$, which are dense but sparse in support due to the compact support of B-splines. The surrogate matrices $\tilde{\mathbf{G}}$ and $\tilde{\mathbf{G}}'$ are banded (bandwidth $2n+1$ and $2m+1$ respectively) [S1].
*   **Strengths:** Reduces edge artifacts and aliasing compared to linear interpolation. Eliminates missing features in homography. Higher resolution than linear interpolation [S1].
*   **Limitations:** Higher-order models increase noise correlation and may show small undershoots. Computationally more expensive than linear interpolation, though faster than exact least-squares B-spline resampling [S1].
*   **When to Use:** When high-fidelity resampling is required for homography, projection, or backprojection, especially in iterative reconstruction or when linear interpolation artifacts are unacceptable [S1].
*   **When Not to Use:** When computational resources are extremely limited and linear interpolation artifacts are tolerable, or when zero-order B-spline "patchy" look is acceptable for speed [S1].

## Background
The method was introduced in the paper "Magnification-driven B-spline Interpolation for Cone Beam Projection and Backprojection" [S1]. It addresses the limitations of standard B-spline interpolation in non-uniform sampling scenarios common in cone beam CT and homography operations. The approach builds on the Cox-de Boor recursion for B-spline basis functions [S1].

## Sources
[S1] Magnification-driven B-spline Interpolation for Cone Beam Projection and Backprojection, https://doi.org/10.1002/mp.15179, fetched 2026-10-26
