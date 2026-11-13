# SNS Backward Pass with Intermediate Variable Q

Also called: SNS Gradient Computation, Skew-Normal 2D Gradient Computation

## Overview
The SNS Backward Pass with Intermediate Variable Q computes the gradients of the loss $L$ with respect to the 3D learnable parameters (position $\mu$, scale $S$, rotation $R$, opacity $o$, color $c$, and skew $k$) for the Skew-Normal 2D Gaussian Splatting model [S1, unofficial]. It solves the problem of high computational complexity and intricate coupling between rotation and scale parameters in standard 3DGS gradient computation by introducing an intermediate variable $Q = ARS$ [S1, unofficial]. Unlike standard 3DGS, which computes gradients with respect to 2D image-space variables first, this method evaluates shared auxiliary algebraic terms during the rendering stage and uses them in a preprocessing stage to directly evaluate closed-form gradients for 3D parameters [S1, unofficial].

## How it works
The backward pass follows the standard 3DGS architecture, divided into a rendering stage and a preprocessing stage [S1, unofficial].

**Inputs:**
- Loss gradient with respect to rendered color: $\frac{\partial L}{\partial C}$
- Pixel coordinates: $u$
- 3D Gaussian parameters: mean $\mu$, scale $S$, rotation $R$, opacity $o$, color $c$, skew $k$
- Projection matrix $A$ (implicit in the derivation of $Q$)

**Outputs:**
- Gradients: $\frac{\partial L}{\partial \mu}$, $\frac{\partial L}{\partial S}$, $\frac{\partial L}{\partial R}$, $\frac{\partial L}{\partial c}$, $\frac{\partial L}{\partial o}$, $\frac{\partial L}{\partial k}$

**Data Structures and Definitions:**
- $\Delta = u - \mu^{2D}$: 2D displacement vector.
- $Q = ARS$: Intermediate variable matrix.
- $\Omega^{2D}$: 2D covariance matrix, derived from $Q$ and $k$.
- $m = m^{2D}$: Skew vector, defined as $m = \frac{(QQ^\top)^{-1}Qk}{\sqrt{1+k^\top k - k^\top Q^\top (QQ^\top)^{-1}Qk}}$.
- $D = \sqrt{1+k^\top k - k^\top Q^\top (\Omega^{2D})^{-1}Qk}$: Scalar denominator.
- $\Phi(x) = \frac{1}{\sqrt{2\pi}} \exp(-\frac{1}{2}x^2)$: Gaussian PDF function.
- $G_2(\Delta; \Sigma)$: 2D Gaussian density function with covariance $\Sigma$.
- $SN^{2D}(u)$: Skew-Normal 2D density, defined as $G_2(\Delta; QQ^\top) \Phi(m^\top \Delta)$.
- Opacity contribution: $a(u) = o \cdot SN^{2D}(u)$.

**Auxiliary Variables (Computed in Rendering Stage):**
These terms are computed once per pixel/Gaussian intersection and reused in the preprocessing stage [S1, unofficial]:
1. $c_\Delta = \Phi \cdot G_2 \cdot (\Omega^{2D})^{-1} \Delta$
2. $M_\Delta = \Phi \cdot G_2 \cdot ((\Omega^{2D})^{-1} \Delta) ((\Omega^{2D})^{-1} \Delta)^\top$
3. $s_\Delta = \frac{G_2}{\sqrt{2\pi}} \exp \left( -\frac{1}{2} (m^\top \Delta)^2 \right)$
4. $p_\Delta = \frac{G_2}{\sqrt{2\pi}} \exp \left( -\frac{1}{2} (m^\top \Delta)^2 \right) \cdot (\Omega^{2D})^{-1} \Delta$

**Numbered Steps:**

1. **Compute Rendered Color and Transmittance:**
   For each pixel $u$, sort Gaussians by depth. Compute transmittance $T_k(u) = \prod_{j=1}^{k-1} (1 - a_j(u))$ and color $C(u) = \sum_{k=1}^N c_k a_k(u) T_k(u)$.

2. **Compute Initial Gradients w.r.t. 2D Variables:**
   Compute $\frac{\partial L}{\partial C}$ from the loss function.
   Compute $\frac{\partial L}{\partial a_k}$ and $\frac{\partial L}{\partial \mu^{2D}}$ using the chain rule through the alpha compositing equation. Note that $\frac{\partial L}{\partial SN^{2D}} = \frac{\partial L}{\partial a} \cdot \frac{1}{o}$ [S1, unofficial].

3. **Rendering Stage: Compute Auxiliary Terms:**
   For each Gaussian $k$ contributing to pixel $u$:
   - Calculate $\Delta = u - \mu^{2D}_k$.
   - Calculate $m_k$ and $D_k$ using current $Q_k$ and $k_k$.
   - Calculate $G_2$ and $\Phi$.
   - Compute $c_\Delta, M_\Delta, s_\Delta, p_\Delta$ using the formulas above.

4. **Preprocessing Stage: Compute 3D Parameter Gradients:**
   Use the auxiliary terms to compute gradients for 3D parameters [S1, unofficial]:

   - **Gradient w.r.t. Mean $\mu$:**
     $$ \frac{\partial SN^{2D}}{\partial \mu^{2D}} = c_\Delta - s_\Delta \cdot m \quad \text{(Eq. 25)} $$
     Then, $\frac{\partial L}{\partial \mu} = \frac{\partial L}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial \mu^{2D}} \cdot \frac{\partial \mu^{2D}}{\partial \mu}$.

   - **Gradient w.r.t. Skew $k$:**
     $$ \frac{\partial SN^{2D}}{\partial k} = \frac{1}{D} Q^\top (p_\Delta + (p_\Delta^\top \Omega^{2D} m) \cdot m) - \frac{1}{D^2} \cdot (p_\Delta^\top \Omega^{2D} m) \cdot k \quad \text{(Eq. 28)} $$
     Then, $\frac{\partial L}{\partial k} = \frac{\partial L}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial k}$.

   - **Gradient w.r.t. Intermediate Variable $Q$:**
     $$ \frac{\partial SN^{2D}}{\partial Q} = (M_\Delta - p_\Delta m^\top - m p_\Delta^\top - (p_\Delta^\top \Omega^{2D} m) \cdot m m^\top) Q + \frac{1}{D} (p_\Delta + (p_\Delta^\top \Omega^{2D} m) \cdot m) k^\top \quad \text{(Eq. 38)} $$
     Then, $\frac{\partial L}{\partial Q} = \frac{\partial L}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial Q}$.

   - **Gradient w.r.t. Rotation $R$ and Scale $S$:**
     Since $Q = ARS$, the gradients $\frac{\partial L}{\partial R}$ and $\frac{\partial L}{\partial S}$ are obtained by backpropagating $\frac{\partial L}{\partial Q}$ through the matrix multiplication $Q = A R S$. This avoids the complex coupling of $R$ and $S$ in $\Omega^{2D}$ directly [S1, unofficial].

   - **Gradient w.r.t. Color $c$ and Opacity $o$:**
     $$ \frac{\partial L}{\partial c} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial c} $$
     $$ \frac{\partial L}{\partial o} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial a} \cdot \frac{\partial a}{\partial o} = \frac{\partial L}{\partial C} \cdot T_k \cdot SN^{2D} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sns_backward_pass(loss_grad_C, pixel_coords, gaussians, projection_matrix_A):
    """
    Computes gradients for SNS 3D Gaussian Splatting.
    
    Args:
        loss_grad_C: Gradient of loss w.r.t. rendered color C. Shape: (H, W, 3)
        pixel_coords: Array of pixel coordinates (u, v). Shape: (N, 2)
        gaussians: List of Gaussian objects with attributes:
                   - mu: 3D mean position
                   - S: Scale matrix (3x3)
                   - R: Rotation quaternion or matrix
                   - o: Opacity scalar
                   - c: Color vector
                   - k: Skew vector
                   - Q: Intermediate variable matrix (2x2 or 3x3 depending on projection)
                   - mu_2d: Projected 2D mean
                   - Omega_2d_inv: Inverse 2D covariance
        projection_matrix_A: Projection matrix A used in Q = ARS.
        
    Returns:
        Gradients dict: {
            'mu': ..., 'S': ..., 'R': ..., 'o': ..., 'c': ..., 'k': ...
        }
    """
    grads = {
        'mu': np.zeros_like(gaussians[0].mu),
        'S': np.zeros_like(gaussians[0].S),
        'R': np.zeros_like(gaussians[0].R),
        'o': np.zeros_like(gaussians[0].o),
        'c': np.zeros_like(gaussians[0].c),
        'k': np.zeros_like(gaussians[0].k)
    }
    
    # 1. Compute gradients w.r.t. 2D variables (simplified for single Gaussian per pixel for clarity)
    # In practice, this involves alpha compositing backward pass
    # loss_grad_a = ... # Gradient of loss w.r.t. opacity contribution a
    # loss_grad_mu_2d = ... # Gradient of loss w.r.t. 2D mean
    
    for i, g in enumerate(gaussians):
        u = pixel_coords[i]
        delta = u - g.mu_2d
        Q = g.Q
        k = g.k
        Omega_inv = g.Omega_2d_inv
        
        # Compute m and D
        # m = (Q Q^T)^-1 Q k / sqrt(1 + k^T k - k^T Q^T Omega_inv Q k)
        QQ_T_inv = np.linalg.inv(Q @ Q.T)
        numerator_m = QQ_T_inv @ (Q @ k)
        denominator_D = np.sqrt(1 + k.T @ k - k.T @ Q.T @ Omega_inv @ Q @ k)
        m = numerator_m / denominator_D
        D = denominator_G # D is the scalar denominator defined in the text
        
        # Compute G2 and Phi
        # G2 is the standard 2D Gaussian density with covariance QQ^T
        # Phi is the skew factor
        G2 = compute_gaussian_density(delta, Q @ Q.T)
        Phi = gaussian_pdf(m.T @ delta)
        
        # 2. Rendering Stage: Compute Auxiliary Terms
        Omega_inv_delta = Omega_inv @ delta
        
        c_delta = Phi * G2 * Omega_inv_delta
        M_delta = Phi * G2 * np.outer(Omega_inv_delta, Omega_inv_delta)
        s_delta = (G2 / np.sqrt(2 * np.pi)) * np.exp(-0.5 * (m.T @ delta)**2)
        p_delta = s_delta * Omega_inv_delta
        
        # 3. Preprocessing Stage: Compute Gradients
        
        # Gradient w.r.t. mu_2d (Eq. 25)
        d_SN2d_d_mu_2d = c_delta - s_delta * m
        
        # Gradient w.r.t. k (Eq. 28)
        # Term 1: 1/D * Q^T * (p_delta + (p_delta^T Omega m) * m)
        term1_inner = p_delta + (p_delta.T @ (g.Omega_2d) @ m) * m # Note: Omega_2d is inverse of Omega_inv
        # Wait, formula uses Omega^2D in the dot product, but p_delta uses Omega^-1.
        # Let's stick to the formula: p_delta^T Omega^2D m.
        # If Omega_inv is stored, Omega^2D = Omega_inv^-1.
        Omega_2d = np.linalg.inv(g.Omega_2d_inv)
        term1_inner = p_delta + (p_delta.T @ Omega_2d @ m) * m
        term1 = (1.0 / D) * (Q.T @ term1_inner)
        
        # Term 2: -1/D^2 * (p_delta^T Omega^2D m) * k
        term2 = -(1.0 / D**2) * (p_delta.T @ Omega_2d @ m) * k
        
        d_SN2d_d_k = term1 + term2
        
        # Gradient w.r.t. Q (Eq. 38)
        # Part 1: (M_delta - p_delta m^T - m p_delta^T - (p_delta^T Omega^2D m) m m^T) Q
        coeff = p_delta.T @ Omega_2d @ m
        part1_matrix = M_delta - np.outer(p_delta, m) - np.outer(m, p_delta) - coeff * np.outer(m, m)
        part1 = part1_matrix @ Q
        
        # Part 2: 1/D * (p_delta + (p_delta^T Omega^2D m) m) k^T
        part2 = (1.0 / D) * (p_delta + coeff * m) @ k.T
        
        d_SN2d_d_Q = part1 + part2
        
        # Aggregate gradients
        # Assuming loss_grad_SN2d is computed from alpha compositing backward pass
        loss_grad_SN2d = ... # Placeholder for actual compositing gradient
        
        grads['k'][i] += loss_grad_SN2d * d_SN2d_d_k
        grads['Q'][i] = loss_grad_SN2d * d_SN2d_d_Q # Store for R, S backprop
        
        # Backprop from Q to R and S
        # Q = A R S => dL/dR = dL/dQ * A^T * S^T, dL/dS = A^T * dL/dQ * S^T (simplified)
        # Exact matrix calculus depends on parameterization of R and S
        grads['R'][i] += backprop_Q_to_R_S(grads['Q'][i], A, g.R, g.S)
        
        # Gradient w.r.t. mu
        grads['mu'][i] += loss_grad_SN2d * d_SN2d_d_mu_2d * d_mu_2d_d_mu
        
        # Gradient w.r.t. c and o
        grads['c'][i] += loss_grad_C[i] * g.a * T_k # Simplified
        grads['o'][i] += loss_grad_C[i] * T_k * G2 * Phi # Simplified
        
    return grads
```

**Edge Cases and Pitfalls:**
- **Precision:** The denominator $D$ involves a square root and subtraction, which may lead to numerical instability if $k^\top Q^\top (\Omega^{2D})^{-1}Qk$ is close to $1+k^\top k$ [S1, unofficial].
- **Matrix Inversion:** Computing $(QQ^\top)^{-1}$ and $(\Omega^{2D})^{-1}$ can be unstable if the matrices are ill-conditioned. Regularization or pseudo-inverses may be needed [G].
- **Degenerate Input:** If $Q$ is singular, $Q^\top Q$ inversion fails. This should be guarded against [G].

## Operating characteristics
- **Time Complexity:** The method reduces computational complexity compared to standard 3DGS by avoiding the direct differentiation of $\Omega^{-1}$ with respect to $R$ and $S$. The use of auxiliary variables $c_\Delta, M_\Delta, s_\Delta, p_\Delta$ allows for efficient reuse of intermediate calculations [S1, unofficial].
- **Memory Complexity:** Requires storing auxiliary variables per Gaussian per pixel during the rendering stage. This is comparable to standard 3DGS but may be slightly higher due to the additional skew parameter $k$ and its associated terms [S1, unofficial].
- **Strengths:** Decouples the complex interaction between rotation and scale in the gradient computation, leading to more stable and efficient training [S1, unofficial].
- **Limitations:** Introduces additional complexity in the gradient derivation and implementation due to the intermediate variable $Q$ and skew parameter $k$ [S1, unofficial].

## Background
The SNS (Skew-Normal Splatting) method was introduced to address the limitations of standard 3D Gaussian Splatting in modeling complex shapes and anisotropic features. The backward pass with intermediate variable $Q$ was proposed to simplify the gradient computation for the skew-normal distribution, which includes a skew parameter $k$ [S1, unofficial]. The method builds on the 3DGS architecture but modifies the gradient computation to handle the additional degrees of freedom introduced by the skew-normal distribution [S1, unofficial].

## Sources
[S1] 3D Skew-Normal Splatting, https://doi.org/10.48550/arxiv.2605.15010, fetched 2026-10-26 (unofficial)
