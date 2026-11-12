# SNS Backward Pass with Intermediate Variable Q

Also called: SNS Gradient Computation, Skew-Normal 2D Gradient Computation

## Overview
The SNS backward pass computes gradients for the 3D Skew-Normal Splatting (SNS) parameters by decoupling the complex interaction between rotation and scale matrices through an intermediate variable $Q$. Unlike standard 3D Gaussian Splatting (3DGS) which computes gradients with respect to the inverse covariance matrix $\Omega^{-1}$ before backpropagating to rotation $R$ and scale $S$, SNS introduces $Q = ARS$ to reformulate the 2D Skew-Normal density function [S1, unofficial]. This approach avoids the high computational complexity of directly differentiating the coupled $R$ and $S$ parameters, instead evaluating shared auxiliary algebraic terms during the rendering stage and using closed-form expressions in the preprocessing stage to compute exact gradients for 3D learnable parameters [S1, unofficial].

## How it works
The backward pass follows the standard 3DGS architecture, divided into a rendering stage and a preprocessing stage [S1, unofficial]. The rendering stage computes gradients with respect to 2D image-space variables and evaluates shared auxiliary terms. The preprocessing stage uses these terms to compute gradients for 3D parameters [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Loss gradient w.r.t. rendered color: $\frac{\partial L}{\partial C} \in \mathbb{R}^3$ (per pixel).
*   2D Gaussian mean: $\mu^{2D} \in \mathbb{R}^2$.
*   Intermediate variable: $Q \in \mathbb{R}^{2 \times 2}$, where $Q = ARS$ ($A$ is projection, $R$ is rotation, $S$ is scale).
*   Skew parameter: $k \in \mathbb{R}^2$.
*   Opacity: $o \in \mathbb{R}$.
*   Color: $c \in \mathbb{R}^3$.
*   Pixel coordinate: $u \in \mathbb{R}^2$.

**Outputs:**
*   Gradients w.r.t. 3D parameters: $\frac{\partial L}{\partial \mu}$, $\frac{\partial L}{\partial S}$, $\frac{\partial L}{\partial R}$, $\frac{\partial L}{\partial c}$, $\frac{\partial L}{\partial o}$, $\frac{\partial L}{\partial k}$.

### Data Structures
*   **Auxiliary Variables:** Precomputed per Gaussian per pixel during the forward pass or early backward pass.
    *   $c_\Delta \in \mathbb{R}^2$
    *   $M_\Delta \in \mathbb{R}^{2 \times 2}$
    *   $s_\Delta \in \mathbb{R}$
    *   $p_\Delta \in \mathbb{R}^2$
    *   $D \in \mathbb{R}$ (scalar denominator)
    *   $m \in \mathbb{R}^2$ (skew direction vector)
    *   $\Omega^{2D} \in \mathbb{R}^{2 \times 2}$ (2D covariance matrix)
    *   $G_2 \in \mathbb{R}$ (Gaussian component value)
    *   $\Phi \in \mathbb{R}$ (Skew-Normal component value)

### Definitions and Intermediate Variables
Let $\Delta = u - \mu^{2D}$. The 2D Skew-Normal density is defined as:
$$
SN^{2D}(u) = G_2(u - \mu^{2D}; QQ^\top) \Phi((m^{2D})^\top(u - \mu^{2D}))
$$
where $G_2$ is the standard 2D Gaussian component and $\Phi$ is the skewing function. The skew direction vector $m$ is defined as:
$$
m = \frac{(QQ^\top)^{-1}Qk}{\sqrt{1+k^\top k - k^\top Q^\top (QQ^\top)^{-1}Qk}}
$$
Let $D$ be the denominator of $m$:
$$
D = \sqrt{1+k^\top k - k^\top Q^\top (QQ^\top)^{-1}Qk}
$$
Thus, $m = \frac{(QQ^\top)^{-1}Qk}{D}$.

The auxiliary variables are computed as follows [S1, unofficial]:
$$
\begin{aligned}
c_\Delta &= \Phi \cdot G_2 \cdot (\Omega^{2D})^{-1} \Delta \\
M_\Delta &= \Phi \cdot G_2 \cdot ((\Omega^{2D})^{-1} \Delta) \cdot ((\Omega^{2D})^{-1} \Delta)^\top \\
s_\Delta &= \frac{G_2}{\sqrt{2\pi}} \exp \left( -\frac{1}{2} (m^\top \Delta)^2 \right) \\
p_\Delta &= \frac{G_2}{\sqrt{2\pi}} \exp \left( -\frac{1}{2} (m^\top \Delta)^2 \right) \cdot (\Omega^{2D})^{-1} \Delta
\end{aligned}
$$
Note that $p_\Delta = s_\Delta \cdot (\Omega^{2D})^{-1} \Delta$.

### Numbered Steps

1.  **Compute Gradients w.r.t. 2D Image-Space Variables:**
    Calculate the gradient of the loss with respect to the rendered color $C$ and opacity $a$ using the standard alpha compositing backward pass [S1, unofficial]. The rendered color is:
    $$
    C(u) = \sum_{i=1}^N c_i a_i(u) \prod_{j=1}^{i-1} (1 - a_j(u))
    $$
    where $a(u) = o \cdot SN^{2D}(u)$. Let $\frac{\partial L}{\partial C}$ be the gradient w.r.t. the final pixel color. Compute $\frac{\partial L}{\partial a}$ and $\frac{\partial L}{\partial c}$ via the chain rule.

2.  **Compute Gradient w.r.t. Skew-Normal Density:**
    Compute $\frac{\partial L}{\partial SN^{2D}}$ from $\frac{\partial L}{\partial a}$ and $o$:
    $$
    \frac{\partial L}{\partial SN^{2D}} = \frac{\partial L}{\partial a} \cdot o
    $$

3.  **Compute Gradient w.r.t. Mean $\mu^{2D}$:**
    Using the auxiliary variables, the gradient of $SN^{2D}$ w.r.t. $\mu^{2D}$ is given by Eq. 25 [S1, unofficial]:
    $$
    \frac{\partial SN^{2D}}{\partial \mu^{2D}} = c_\Delta - s_\Delta \cdot m
    $$
    The total gradient w.r.t. $\mu$ is:
    $$
    \frac{\partial L}{\partial \mu} = \frac{\partial L}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial \mu^{2D}} \cdot \frac{\partial \mu^{2D}}{\partial \mu}
    $$
    (Note: $\frac{\partial \mu^{2D}}{\partial \mu}$ involves the projection Jacobian, handled by the parent method).

4.  **Compute Gradient w.r.t. Skew Parameter $k$:**
    Using the auxiliary variables, the gradient of $SN^{2D}$ w.r.t. $k$ is given by Eq. 28 [S1, unofficial]:
    $$
    \frac{\partial SN^{2D}}{\partial k} = \frac{1}{D} Q^\top (p_\Delta + (p_\Delta^\top \Omega^{2D} m) \cdot m) - \frac{1}{D^2} \cdot (p_\Delta^\top \Omega^{2D} m) \cdot k
    $$
    The total gradient w.r.t. $k$ is:
    $$
    \frac{\partial L}{\partial k} = \frac{\partial L}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial k}
    $$

5.  **Compute Gradient w.r.t. Intermediate Variable $Q$:**
    Using the auxiliary variables, the gradient of $SN^{2D}$ w.r.t. $Q$ is given by Eq. 38 [S1, unofficial]:
    $$
    \frac{\partial SN^{2D}}{\partial Q} = (M_\Delta - p_\Delta m^\top - m p_\Delta^\top - (p_\Delta^\top \Omega^{2D} m) \cdot m m^\top) Q + \frac{1}{D} (p_\Delta + (p_\Delta^\top \Omega^{2D} m) \cdot m) k^\top
    $$
    The total gradient w.r.t. $Q$ is:
    $$
    \frac{\partial L}{\partial Q} = \frac{\partial L}{\partial SN^{2D}} \cdot \frac{\partial SN^{2D}}{\partial Q}
    $$

6.  **Compute Gradient w.r.t. Color $c$:**
    $$
    \frac{\partial L}{\partial c} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial c}
    $$
    From the compositing equation, $\frac{\partial C}{\partial c_i} = T_i(u) \alpha_i(u)$, where $T_i$ is transmittance and $\alpha_i$ is opacity.

7.  **Compute Gradient w.r.t. Opacity $o$:**
    $$
    \frac{\partial L}{\partial o} = \frac{\partial L}{\partial C} \cdot \frac{\partial C}{\partial a} \cdot \frac{\partial a}{\partial o} = \frac{\partial L}{\partial a} \cdot SN^{2D}(u)
    $$

8.  **Backpropagate $Q$ to $R$ and $S$:**
    The gradient $\frac{\partial L}{\partial Q}$ must be propagated to the original 3D parameters $R$ and $S$. Since $Q = ARS$, and $A$ is constant for a given view, we have $\frac{\partial L}{\partial (RS)} = \frac{\partial L}{\partial Q} A^\top$. The decomposition of $Q$ into $R$ and $S$ requires further differentiation based on the specific parameterization of $R$ (e.g., quaternion) and $S$ (e.g., log-scale). This step is implementation-specific to the 3DGS parameterization and is not detailed in the source [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_auxiliary_variables(Q, k, mu_2d, u, Omega_2d, G2, Phi):
    """
    Computes auxiliary variables c_delta, M_delta, s_delta, p_delta, m, D.
    """
    Delta = u - mu_2d  # R^2
    
    # Compute m and D
    # m = (Q Q^T)^-1 Q k / D
    # D = sqrt(1 + k^T k - k^T Q^T (Q Q^T)^-1 Q k)
    
    QQT_inv = np.linalg.inv(Q @ Q.T)
    temp_vec = QQT_inv @ (Q @ k)
    D_sq = 1.0 + (k @ k) - (k @ Q.T @ QQT_inv @ (Q @ k))
    
    # Guard against numerical instability in D
    if D_sq <= 0:
        D_sq = 1e-8
        
    D = np.sqrt(D_sq)
    m = temp_vec / D
    
    # Compute Omega_inv_Delta
    Omega_inv = np.linalg.inv(Omega_2d)
    Omega_inv_Delta = Omega_inv @ Delta
    
    # c_delta = Phi * G2 * Omega_inv * Delta
    c_delta = Phi * G2 * Omega_inv_Delta
    
    # M_delta = Phi * G2 * (Omega_inv * Delta) * (Omega_inv * Delta)^T
    M_delta = Phi * G2 * np.outer(Omega_inv_Delta, Omega_inv_Delta)
    
    # s_delta = (G2 / sqrt(2pi)) * exp(-0.5 * (m^T Delta)^2)
    mT_Delta = m @ Delta
    s_delta = (G2 / np.sqrt(2 * np.pi)) * np.exp(-0.5 * mT_Delta**2)
    
    # p_delta = s_delta * Omega_inv * Delta
    p_delta = s_delta * Omega_inv_Delta
    
    return c_delta, M_delta, s_delta, p_delta, m, D

def compute_gradients_SN2D(c_delta, M_delta, s_delta, p_delta, m, D, Q, k, Omega_2d, dL_dSN2D):
    """
    Computes gradients of SN2D w.r.t mu_2d, k, and Q.
    """
    # Gradient w.r.t mu_2d (Eq. 25)
    # dSN2D/dmu_2d = c_delta - s_delta * m
    dSN2D_dmu_2d = c_delta - s_delta * m
    
    # Gradient w.r.t k (Eq. 28)
    # dSN2D/dk = (1/D) * Q^T * (p_delta + (p_delta^T Omega_2d m) * m) - (1/D^2) * (p_delta^T Omega_2d m) * k
    term1_inner = p_delta + (p_delta @ (Omega_2d @ m)) * m
    term1 = (1.0 / D) * (Q.T @ term1_inner)
    
    term2_coeff = (p_delta @ (Omega_2d @ m))
    term2 = (1.0 / (D**2)) * term2_coeff * k
    
    dSN2D_dk = term1 - term2
    
    # Gradient w.r.t Q (Eq. 38)
    # dSN2D/dQ = (M_delta - p_delta m^T - m p_delta^T - (p_delta^T Omega_2d m) * m m^T) Q + (1/D) (p_delta + (p_delta^T Omega_2d m) * m) k^T
    
    coeff_mm = p_delta @ (Omega_2d @ m)
    
    # Matrix part
    # M_delta - p_delta m^T - m p_delta^T - coeff_mm * m m^T
    part1 = M_delta
    part2 = np.outer(p_delta, m)
    part3 = np.outer(m, p_delta)
    part4 = coeff_mm * np.outer(m, m)
    
    matrix_term = part1 - part2 - part3 - part4
    
    # Vector part
    vector_term = (1.0 / D) * (p_delta + coeff_mm * m)
    
    dSN2D_dQ = matrix_term @ Q + np.outer(vector_term, k)
    
    # Scale by dL/dSN2D
    dL_dmu_2d = dL_dSN2d * dSN2D_dmu_2d
    dL_dk = dL_dSN2d * dSN2D_dk
    dL_dQ = dL_dSN2d * dSN2D_dQ
    
    return dL_dmu_2d, dL_dk, dL_dQ

def backward_pass(dL_dC, dL_da, c, a, T, o, SN2D, Q, k, mu_2d, u, Omega_2d, G2, Phi):
    """
    Computes gradients for SNS parameters.
    """
    # 1. Gradient w.r.t. Color c
    # dL/dc = dL/dC * dC/dc = dL/dC * T * a
    dL_dc = dL_dC * T * a
    
    # 2. Gradient w.r.t. Opacity o
    # dL/do = dL/da * SN2D
    dL_do = dL_da * SN2D
    
    # 3. Gradient w.r.t. SN2D
    # dL/dSN2D = dL/da * o
    dL_dSN2D = dL_da * o
    
    # 4. Compute Auxiliary Variables
    c_delta, M_delta, s_delta, p_delta, m, D = compute_auxiliary_variables(
        Q, k, mu_2d, u, Omega_2d, G2, Phi
    )
    
    # 5. Compute Gradients w.r.t. mu_2d, k, Q
    dL_dmu_2d, dL_dk, dL_dQ = compute_gradients_SN2D(
        c_delta, M_delta, s_delta, p_delta, m, D, Q, k, Omega_2d, dL_dSN2D
    )
    
    # 6. Backpropagate Q to R and S
    # Q = ARS => dL/d(RS) = dL/dQ * A^T
    # This step depends on the specific parameterization of R and S.
    # Not fully detailed in source [S1, unofficial].
    
    return {
        'dL_dc': dL_dc,
        'dL_do': dL_do,
        'dL_dmu_2d': dL_dmu_2d,
        'dL_dk': dL_dk,
        'dL_dQ': dL_dQ
    }
```

### Edge Cases and Pitfalls
*   **Numerical Stability of $D$:** The denominator $D$ involves a square root of a term that can approach zero if the skew parameter $k$ is large or if $Q$ is ill-conditioned. A small epsilon guard should be applied to $D^2$ before taking the square root [S1, unofficial].
*   **Matrix Inversion:** Computing $(QQ^\top)^{-1}$ and $(\Omega^{2D})^{-1}$ can be numerically unstable if the matrices are singular or near-singular. Regularization or pseudo-inversion may be required [S1, unofficial].
*   **Gradient Explosion:** The terms involving $1/D$ and $1/D^2$ can lead to gradient explosion if $D$ is small. Clipping or normalization of gradients may be necessary [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The computation of auxiliary variables is $O(1)$ per Gaussian per pixel (constant size matrices). The gradient computations are also $O(1)$. The overall complexity is dominated by the rendering stage and the compositing backward pass, which is $O(N)$ per pixel, where $N$ is the number of Gaussians.
*   **Memory Complexity:** Requires storing auxiliary variables $c_\Delta, M_\Delta, s_\Delta, p_\Delta, m, D$ per Gaussian per pixel during the backward pass. This increases memory usage compared to standard 3DGS, which does not compute these specific auxiliary terms [S1, unofficial].
*   **Strengths:** Decouples the complex interaction between rotation and scale, leading to more efficient and stable gradient computation for the skew parameters. The use of closed-form expressions reduces computational overhead compared to numerical differentiation [S1, unofficial].
*   **Limitations:** Requires additional memory for auxiliary variables. The backpropagation from $Q$ to $R$ and $S$ is non-trivial and depends on the specific parameterization [S1, unofficial].

## Background
The SNS backward pass was introduced in the "3D Skew-Normal Splatting" paper [S1, unofficial]. It addresses the limitations of standard 3DGS and SSS in handling skewed Gaussians by providing an efficient and exact gradient computation method. The method builds on the 3DGS architecture but modifies the gradient computation stage to handle the Skew-Normal density function [S1, unofficial].

## Sources
[S1] 3D Skew-Normal Splatting, https://doi.org/10.48550/arxiv.2605.15010, fetched 2026-10-26 (unofficial)
