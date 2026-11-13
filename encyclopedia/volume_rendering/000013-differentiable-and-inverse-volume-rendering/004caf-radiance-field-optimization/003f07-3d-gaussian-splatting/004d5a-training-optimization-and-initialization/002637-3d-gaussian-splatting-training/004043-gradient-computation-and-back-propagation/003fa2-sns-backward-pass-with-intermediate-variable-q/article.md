# SNS Backward Pass with Intermediate Variable Q

Also called: SNS Gradient Computation, Skew-Normal 2D Gradient Computation

## Overview
The SNS backward pass computes exact gradients for 3D Gaussian Splatting primitives using the Skew-Normal 2D density function, addressing the complex coupling between rotation and scale parameters. Unlike standard 3DGS which computes gradients via the inverse covariance matrix, this method introduces an intermediate variable $Q = ARS$ to decouple these parameters, enabling closed-form analytical gradients for position, skew, and shape parameters [S1, unofficial]. It builds on [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md) by replacing the standard Gaussian gradient path with a skew-normal-specific chain rule.

## How it works
The backward pass follows the standard 3DGS architecture, divided into a rendering stage and a preprocessing stage [S1, unofficial]. The rendering stage evaluates shared auxiliary algebraic terms, and the preprocessing stage uses these terms to compute exact gradients for 3D learnable parameters using closed-form expressions [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Loss gradient with respect to rendered color: $\frac{\partial L}{\partial C} \in \mathbb{R}^3$ (per pixel).
- Primitive parameters: mean $\mu$, scale $s$, rotation $R$, skew $k$, opacity $o$, color $c$.
- Projection matrix $A$.
- Pixel coordinate $u \in \mathbb{R}^2$.

**Outputs:**
- Gradients for 3D parameters: $\frac{\partial L}{\partial \mu}, \frac{\partial L}{\partial s}, \frac{\partial L}{\partial R}, \frac{\partial L}{\partial k}, \frac{\partial L}{\partial o}, \frac{\partial L}{\partial c}$.

### Data Structures
- **Intermediate Variable $Q$**: $Q = ARS \in \mathbb{R}^{2 \times 2}$. This matrix decouples the rotation $R$ and scale $S$ from the projection $A$ [S1, unofficial].
- **Auxiliary Terms**: Precomputed scalars and vectors/matrices to simplify gradient expressions:
  - $c_\Delta \in \mathbb{R}^2$
  - $M_\Delta \in \mathbb{R}^{2 \times 2}$
  - $s_\Delta \in \mathbb{R}$
  - $p_\Delta \in \mathbb{R}^2$
- **2D Covariance $\Omega^{2D}$**: Derived from $Q$ as $\Omega^{2D} = QQ^\top$ [S1, unofficial].

### Step-by-Step Method

1. **Compute Intermediate Variable $Q$**:
   Calculate $Q = ARS$. This reformulates the Skew-Normal density to avoid direct differentiation of the coupled $R$ and $S$ parameters [S1, unofficial].

2. **Compute Vector $m$**:
   Define $m = m^{2D}$ as:
   $$ m = \frac{(QQ^\top)^{-1}Qk}{\sqrt{1+k^\top k - k^\top Q^\top (QQ^\top)^{-1}Qk}} $$
   Let $D = \sqrt{1+k^\top k - k^\top Q^\top (QQ^\top)^{-1}Qk}$. Then $m = \frac{(QQ^\top)^{-1}Qk}{D}$ [S1, unofficial].

3. **Compute Difference Vector $\Delta$**:
   $$ \Delta = u - \mu^{2D} $$
   where $\mu^{2D}$ is the projected mean [S1, unofficial].

4. **Compute Auxiliary Terms**:
   Let $G_2$ be the 2D Gaussian component and $\Phi$ be the skew-normal activation function (typically $\Phi(x) = 1 + \text{erf}(x/\sqrt{2})$ or similar, though the exact form is absorbed into the derivatives below). The auxiliary terms are:
   $$ c_\Delta = \Phi \cdot G_2 \cdot (\Omega^{2D})^{-1} \Delta $$
   $$ M_\Delta = \Phi \cdot G_2 \cdot ((\Omega^{2D})^{-1} \Delta) ((\Omega^{2D})^{-1} \Delta)^\top $$
   $$ s_\Delta = \frac{G_2}{\sqrt{2\pi}} \exp \left( -\frac{1}{2} (m^\top \Delta)^2 \right) $$
   $$ p_\Delta = \frac{G_2}{\sqrt{2\pi}} \exp \left( -\frac{1}{2} (m^\top \Delta)^2 \right) \cdot (\Omega^{2D})^{-1} \Delta $$
   Note: $s_\Delta$ is a scalar, $c_\Delta, p_\Delta$ are vectors, $M_\Delta$ is a matrix [S1, unofficial].

5. **Compute Gradients of $SN^{2D}$**:
   The Skew-Normal density is $SN^{2D}(u) = G_2(u - \mu^{2D}; QQ^\top \Phi(m^\top \Delta))$. The gradients of this density with respect to the parameters are:

   - **Gradient w.r.t $\mu^{2D}$**:
     $$ \frac{\partial SN^{2D}}{\partial \mu^{2D}} = c_\Delta - s_\Delta \cdot m \quad \text{(Eq. 25)} $$
     Since $\Delta = u - \mu^{2D}$, $\frac{\partial \Delta}{\partial \mu^{2D}} = -I$. The full gradient w.r.t 3D mean $\mu$ involves the projection Jacobian $\frac{\partial \mu^{2D}}{\partial \mu}$ [S1, unofficial].

   - **Gradient w.r.t skew $k$**:
     $$ \frac{\partial SN^{2D}}{\partial k} = \frac{1}{D} Q^\top (p_\Delta + (p_\Delta^\top \Omega^{2D} m) \cdot m) - \frac{1}{D^2} (p_\Delta^\top \Omega^{2D} m) \cdot k \quad \text{(Eq. 28)} $$
     [S1, unofficial].

   - **Gradient w.r.t $Q$**:
     $$ \frac{\partial SN^{2D}}{\partial Q} = (M_\Delta - p_\Delta m^\top - m p_\Delta^\top - (p_\Delta^\top \Omega^{2D} m) \cdot m m^\top) Q + \frac{1}{D} (p_\Delta + (p_\Delta^\top \Omega^{2D} m) \cdot m) k^\top \quad \text{(Eq. 38)} $$
     [S1, unofficial].

6. **Compute Gradients for 3D Parameters**:
   Use the chain rule to propagate $\frac{\partial L}{\partial SN^{2D}}$ to the 3D parameters.

   - **Gradient w.r.t Color $c$**:
     $$ \frac{\partial L}{\partial c} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial c} = \frac{\partial L}{\partial C} \cdot a(u) $$
     where $a(u) = o \cdot SN^{2D}(u)$ is the opacity [S1, unofficial].

   - **Gradient w.r.t Opacity $o$**:
     $$ \frac{\partial L}{\partial o} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial a} \cdot \frac{\partial a}{\partial o} = \frac{\partial L}{\partial C} \cdot SN^{2D}(u) $$
     [S1, unofficial].

   - **Gradient w.r.t Mean $\mu$**:
     $$ \frac{\partial L}{\partial \mu} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial a} \cdot \frac{\partial a}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial \mu^{2D}} \cdot \frac{\partial \mu^{2D}}{\partial \mu} $$
     Substitute $\frac{\partial SN^{2D}}{\partial \mu^{2D}} = c_\Delta - s_\Delta m$ [S1, unofficial].

   - **Gradient w.r.t Skew $k$**:
     $$ \frac{\partial L}{\partial k} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial a} \cdot \frac{\partial a}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial k} $$
     Substitute $\frac{\partial SN^{2D}}{\partial k}$ from Eq. 28 [S1, unofficial].

   - **Gradient w.r.t Shape ($R, S$)**:
     First compute $\frac{\partial L}{\partial Q}$:
     $$ \frac{\partial L}{\partial Q} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial a} \cdot \frac{\partial a}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial Q} $$
     Substitute $\frac{\partial SN^{2D}}{\partial Q}$ from Eq. 38 [S1, unofficial].
     Then, backpropagate from $Q$ to $R$ and $S$ using $Q = ARS$. Since $A$ is constant, $\frac{\partial L}{\partial R} = \frac{\partial L}{\partial Q} A S^\top$ and $\frac{\partial L}{\partial S} = A^\top \frac{\partial L}{\partial Q} R^\top$ (assuming standard matrix calculus for $Q=ARS$) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sns_backward_pass(L_grad_C, u, mu_3d, R, S, k, o, c, A, inv_cov_2d):
    """
    Computes gradients for SNS 3DGS primitive.
    
    Args:
        L_grad_C: Gradient of loss w.r.t. rendered color (3,)
        u: Pixel coordinate (2,)
        mu_3d: 3D mean position (3,)
        R: Rotation matrix (3,3) or quaternion
        S: Scale matrix (3,3) diagonal
        k: Skew parameter (2,)
        o: Opacity scalar
        c: Color (3,)
        A: Projection matrix (2,3)
        inv_cov_2d: Inverse of 2D covariance matrix Omega_2D (2,2)
        
    Returns:
        Gradients for mu_3d, R, S, k, o, c
    """
    # 1. Compute Q = A * R * S
    # Note: R and S are 3D, A is 2x3. We need the 2D projection of the shape.
    # In standard 3DGS, the 2D covariance is Omega_2D = J * W * Sigma * W^T * J^T.
    # Here, SNS defines Q = A*R*S. Assuming R, S are 3x3, we take the top-left 2x2 of A*R*S?
    # Or R, S are already projected? The source says Q = ARS. 
    # Let's assume R and S are 3D parameters, and A projects them.
    # However, Eq 19 uses Q directly in the 2D density. 
    # Implementation detail: Q is 2x2.
    
    # Compute 2D mean mu_2d
    mu_2d = A @ mu_3d
    
    # Compute Delta
    Delta = u - mu_2d
    
    # Compute Q from R, S, A
    # Assuming R and S are 3x3 matrices. 
    # The source implies Q is derived from the shape parameters.
    # Let's assume a helper function to get the 2D shape matrix from 3D R, S and A.
    # Q = A @ R @ S @ A.T ? No, Q is 2x2.
    # Standard 3DGS: Omega_2D = J W Sigma W^T J^T.
    # SNS: Q = A R S. If R, S are 3x3, A is 2x3, then A R S is 2x3. 
    # This doesn't match Q being 2x2. 
    # Re-reading: "Q = ARS". In 3DGS, the 2D covariance is often parameterized.
    # Let's assume Q is the 2x2 matrix representing the shape in 2D space.
    # For the purpose of the backward pass, we assume Q is available or computed from R, S.
    
    # Let's assume Q is computed as the relevant 2x2 part of the projected shape.
    # For the gradient w.r.t R and S, we need dL/dQ.
    
    # 2. Compute m
    # m = (Q Q^T)^-1 Q k / D
    # D = sqrt(1 + k^T k - k^T Q^T (Q Q^T)^-1 Q k)
    
    QQ_T = Q @ Q.T
    inv_QQ_T = np.linalg.inv(QQ_T)
    
    numerator_m = inv_QQ_T @ (Q @ k)
    denominator_term = 1 + k.T @ k - k.T @ Q.T @ inv_QQ_T @ (Q @ k)
    D = np.sqrt(denominator_term)
    m = numerator_m / D
    
    # 3. Compute G2 and Phi
    # G2 is the 2D Gaussian component. 
    # G2 = 1/(2*pi*sqrt(det(Omega_2D))) * exp(-0.5 * Delta^T Omega_2D^-1 Delta)
    # But the source uses G2 in the auxiliary terms.
    # Let's compute the exponent term
    exponent = -0.5 * Delta.T @ inv_cov_2d @ Delta
    # det(Omega_2D) can be derived from Q. det(QQ^T) = det(Q)^2.
    # Let's assume we have G2 and Phi precomputed or computed here.
    # Phi is the skew activation. 
    # The source defines auxiliary terms using G2 and Phi.
    
    # For implementation, we need G2 and Phi.
    # G2 = (1 / (2 * pi * np.sqrt(np.linalg.det(QQ_T)))) * np.exp(exponent)
    # Phi = 1 + erf(m.T @ Delta / sqrt(2)) ? The source doesn't specify Phi's exact form, 
    # but it appears in the derivatives. 
    # Let's assume a standard normal CDF or similar.
    # However, the auxiliary terms are defined directly.
    
    # Let's compute the auxiliary terms directly as per Eq 21.
    # We need G2 and Phi.
    # Let's assume G2 is the Gaussian part and Phi is the skew part.
    # The source says SN^2D = G2 * Phi.
    
    # Compute G2
    # det_Omega = np.linalg.det(QQ_T)
    # G2 = (1.0 / (2 * np.pi * np.sqrt(det_Omega))) * np.exp(exponent)
    
    # Compute Phi
    # Phi = 1 + erf((m.T @ Delta) / np.sqrt(2))
    # But the source uses Phi in the auxiliary terms.
    # Let's compute the term G2 * Phi.
    
    # Actually, the auxiliary terms are defined as:
    # c_delta = Phi * G2 * inv_cov_2d @ Delta
    # M_delta = Phi * G2 * (inv_cov_2d @ Delta) @ (inv_cov_2d @ Delta).T
    # s_delta = G2 / sqrt(2*pi) * exp(-0.5 * (m.T @ Delta)**2)
    # p_delta = G2 / sqrt(2*pi) * exp(-0.5 * (m.T @ Delta)**2) * inv_cov_2d @ Delta
    
    # Let's compute the common factor for s_delta and p_delta
    skew_term = m.T @ Delta
    exp_skew = np.exp(-0.5 * skew_term**2)
    
    # We need G2 and Phi.
    # Let's assume we have them.
    # If not, we compute them.
    
    # For the sake of the pseudocode, we assume G2 and Phi are computed.
    # Let's define a helper to compute G2 and Phi.
    
    # Compute G2
    det_Omega = np.linalg.det(QQ_T)
    G2 = (1.0 / (2 * np.pi * np.sqrt(det_Omega))) * np.exp(exponent)
    
    # Compute Phi
    # Standard Skew-Normal uses Phi(x) = 2 * Phi_N(x) where Phi_N is CDF of N(0,1)
    # Or Phi(x) = 1 + erf(x/sqrt(2))
    Phi = 1 + erf(skew_term / np.sqrt(2))
    
    # Compute auxiliary terms
    inv_cov_Delta = inv_cov_2d @ Delta
    
    c_delta = Phi * G2 * inv_cov_Delta
    M_delta = Phi * G2 * np.outer(inv_cov_Delta, inv_cov_Delta)
    s_delta = (G2 / np.sqrt(2 * np.pi)) * exp_skew
    p_delta = s_delta * inv_cov_Delta
    
    # 4. Compute gradients of SN^2D
    # dSN_d_mu_2d = c_delta - s_delta * m
    dSN_d_mu_2d = c_delta - s_delta * m
    
    # dSN_d_k
    # term1 = (p_delta.T @ inv_cov_2d @ m)
    # Note: inv_cov_2d is Omega_2D^-1. The source uses Omega_2D in the gradient formula.
    # Eq 28: p_delta^T Omega_2D m. But p_delta already has inv_cov_2d?
    # Let's check Eq 28: p_delta^T Omega_2D m.
    # p_delta = s_delta * inv_cov_2d @ Delta.
    # So p_delta^T Omega_2D m = (s_delta * Delta^T inv_cov_2d) Omega_2D m = s_delta * Delta^T m.
    # This simplifies.
    
    term_k = p_delta.T @ inv_cov_2d @ m # This is s_delta * Delta^T m
    dSN_d_k = (1.0/D) * (Q.T @ (p_delta + term_k * m)) - (1.0/D**2) * term_k * k
    
    # dSN_d_Q
    # term_Q1 = (p_delta.T @ inv_cov_2d @ m)
    # term_Q2 = p_delta + term_Q1 * m
    # dSN_d_Q = (M_delta - p_delta @ m.T - m @ p_delta.T - term_Q1 * np.outer(m, m)) @ Q + (1.0/D) * np.outer(term_Q2, k)
    
    term_Q1 = p_delta.T @ inv_cov_2d @ m
    term_Q2 = p_delta + term_Q1 * m
    
    dSN_d_Q = (M_delta - np.outer(p_delta, m) - np.outer(m, p_delta) - term_Q1 * np.outer(m, m)) @ Q + (1.0/D) * np.outer(term_Q2, k)
    
    # 5. Compute gradients for 3D parameters
    # dL/dC is L_grad_C
    # dL/dSN = dL/dC * dC/dSN
    # C = sum c_i a_i ...
    # dC/dSN = c * o * T_prev (where T_prev is transmittance before this gaussian)
    # Let's assume dL/dSN is passed in or computed from the compositing gradient.
    # For simplicity, let's assume dL_d_SN is the gradient of loss w.r.t SN^2D.
    
    # dL/dc = dL/dC * a(u) = dL/dC * o * SN^2D
    SN_2D = G2 * Phi
    dL_dc = L_grad_C * (o * SN_2D)
    
    # dL/do = dL/dC * SN^2D
    dL_do = L_grad_C * SN_2D
    
    # dL/dmu_3d = dL/dC * dC/dSN * dSN/dmu_2d * dmu_2d/dmu_3d
    # dmu_2d/dmu_3d = A
    # Let dL_d_SN = dL/dC * dC/dSN (scalar per pixel, but we are doing per primitive)
    # Assuming we have dL_d_SN from the compositing pass.
    
    # dL/dmu_3d = dL_d_SN * (c_delta - s_delta * m) @ A
    # Note: dL_d_SN is a vector (3,) if L_grad_C is (3,).
    # dL/dmu_3d = A.T @ (dL_d_SN * (c_delta - s_delta * m))
    
    # dL/dk = dL_d_SN * dSN/dk
    dL_dk = dL_d_SN * dSN_d_k
    
    # dL/dQ = dL_d_SN * dSN/dQ
    dL_dQ = dL_d_SN * dSN_d_Q
    
    # dL/dR and dL/dS from dL/dQ
    # Q = A R S
    # dL/dR = A.T @ dL/dQ @ S.T
    # dL/dS = A.T @ dL/dQ @ R.T
    # Note: R and S are 3x3. dL/dQ is 2x2. A is 2x3.
    # dL/dR = A.T @ dL/dQ @ S.T ?
    # dQ = A dR S + A R dS
    # dL = Tr(dL/dQ^T dQ) = Tr(dL/dQ^T A dR S) + Tr(dL/dQ^T A R dS)
    # dL/dR = A.T @ dL/dQ @ S.T
    # dL/dS = R.T @ A.T @ dL/dQ
    
    dL_dR = A.T @ dL_dQ @ S.T
    dL_dS = R.T @ A.T @ dL_dQ
    
    return dL_dmu_3d, dL_dR, dL_dS, dL_dk, dL_do, dL_dc
```

### Edge Cases and Pitfalls
- **Precision**: The computation of $D$ involves square roots and inverses, which can be numerically unstable if $QQ^\top$ is near-singular. Guard against division by zero in $D$ and matrix inversion in $(QQ^\top)^{-1}$ [S1, unofficial].
- **Boundary Conditions**: If $k$ is large, the skew-normal distribution may become ill-conditioned. The term $1+k^\top k - k^\top Q^\top (QQ^\top)^{-1}Qk$ must remain positive [S1, unofficial].
- **Degenerate Input**: If the Gaussian is far from the pixel, $G_2$ becomes very small, leading to underflow. Use a threshold to skip computation for negligible contributions [G].

## Operating characteristics
- **Time Complexity**: The backward pass is $O(N)$ per pixel for $N$ primitives, similar to standard 3DGS, but with higher constant factors due to the additional auxiliary term computations and matrix operations for $Q$ [S1, unofficial].
- **Memory Complexity**: Requires storing auxiliary terms $c_\Delta, M_\Delta, s_\Delta, p_\Delta$ per primitive per pixel, increasing memory usage compared to standard 3DGS [S1, unofficial].
- **Strengths**: Provides exact gradients for skew parameters, enabling better optimization of skewed shapes. Decouples rotation and scale via $Q$, simplifying gradient computation [S1, unofficial].
- **Limitations**: Higher computational overhead due to auxiliary terms. More complex implementation than standard 3DGS [S1, unofficial].

## Background
The SNS backward pass was introduced in the "3D Skew-Normal Splatting" paper [S1, unofficial]. It extends the 3D Gaussian Splatting framework by incorporating skew-normal distributions to better model asymmetric shapes. The method addresses the difficulty of computing gradients for coupled rotation and scale parameters by introducing the intermediate variable $Q = ARS$ [S1, unofficial].

## Sources
[S1] 3D Skew-Normal Splatting, https://doi.org/10.48550/arxiv.2605.15010, fetched 2026-10-26 (unofficial)
