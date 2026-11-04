# Tetrahedral Regularization

Also called: Softplus Quality Regularizer

## Overview
Tetrahedral Regularization is a penalty term added to the optimization loss function to prevent tetrahedral elements from becoming inverted (negative volume) or degenerate (volume approaching zero) during the differentiable rendering of non-regular tetrahedral meshes [S1, unofficial]. It sets itself apart from sibling methods like [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md) by actively correcting vertex positions via gradient descent rather than modifying the mesh topology, and from [Bezier Tetrahedra Evaluation](1422-bezier-tetrahedra-evaluation.md) by operating on linear basis functions with a smooth, differentiable quality metric [S1, unofficial].

## How it works
**Inputs:**
- Set of tetrahedra $T$.
- For each tetrahedron $t \in T$, the positions of its four vertices $p_0, p_1, p_2, p_3 \in \mathbb{R}^3$.
- Hyperparameters: $\lambda$ (regularization weight), $\beta$ (softplus steepness).

**Outputs:**
- Scalar regularization loss value.
- Gradients of the loss with respect to vertex positions $p_i$.

**Data Structures:**
- Vertices are stored as 3D vectors.
- Edge lengths are computed on-the-fly from vertex positions.

**Steps:**
1. For each tetrahedron $t \in T$:
   a. Compute the signed volume $V$ using the scalar triple product of the edges originating from $p_0$:
      $$V = \langle (p_3 - p_0), (p_1 - p_0) \times (p_2 - p_0) \rangle$$
   b. Compute the six edge lengths $l_i$ for $i=0..5$ corresponding to the pairs $(p_0,p_1), (p_0,p_2), (p_0,p_3), (p_1,p_2), (p_1,p_3), (p_2,p_3)$.
   c. Compute the root-mean-square edge length $l_{rms}$:
      $$l_{rms} = \sqrt{\frac{1}{6} \sum_{i=0}^5 l_i^2}$$
   d. Compute the smooth quality measure $Q$:
      $$Q = 6\sqrt{2} \frac{V}{l_{rms}^3}$$
   e. Compute the softplus penalty:
      $$\text{Softplus}(Q) = \frac{1}{\beta} \log(1 + \exp(\beta Q))$$
   f. Accumulate the penalty to the total loss.
2. Multiply the accumulated sum by $\lambda$ and add to the image loss $\text{loss}_{\text{img}}$.

**Formulas:**
- **Signed Volume:** $V = \langle (p_3 - p_0), (p_1 - p_0) \times (p_2 - p_0) \rangle$ [S1, unofficial].
- **RMS Edge Length:** $l_{rms} = \sqrt{\frac{1}{6} \sum_{i=0}^5 l_i^2}$ [S1, unofficial].
- **Quality Metric:** $Q = 6\sqrt{2} \frac{V}{l_{rms}^3}$ [S1, unofficial]. This metric is derived from Shewchuk and Parthasarathy et al. [S1, unofficial].
- **Softplus Function:** $\text{Softplus}(x) = \frac{1}{\beta} \log(1 + \exp(\beta x))$ [S1, unofficial].
- **Softplus Derivative:** $\frac{\partial \text{Softplus}(x)}{\partial x} = \frac{1}{1 + \exp(-\beta x)}$ [S1, unofficial].
- **Total Loss:** $\text{loss} = \text{loss}_{\text{img}} + \lambda \sum_{t \in T} \text{Softplus}(Q(t))$ [S1, unofficial].

**Parameters:**
- $\beta$: Controls the steepness of the penalty. Typical values are 10 or 100. Higher values penalize slightly deformed tetrahedra less [S1, unofficial].
- $\lambda$: Weight of the regularization term. Typical value is 10 [S1, unofficial].

**Pitfalls:**
- Tetrahedra may invert or degenerate due to $C^0$-continuity of the color field and high learning rates [S1, unofficial].
- The derivative of $Q$ with respect to vertex positions is complex; it is recommended to use automatic differentiation tools (e.g., SymPy) to generate the GLSL implementation, as provided in the DiffTetVR source code [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tetrahedral_regularization(vertices, tetrahedra_indices, lambda_reg=10.0, beta=10.0):
    """
    Computes the tetrahedral regularization loss and its gradient w.r.t. vertices.
    
    Args:
        vertices: Array of shape (N, 3) containing vertex positions.
        tetrahedra_indices: Array of shape (M, 4) containing vertex indices for each tetrahedron.
        lambda_reg: Weight for the regularization term.
        beta: Steepness parameter for the softplus function.
        
    Returns:
        loss: Scalar regularization loss.
        grad_vertices: Gradient array of shape (N, 3).
    """
    loss = 0.0
    grad_vertices = np.zeros_like(vertices)
    
    for idx in tetrahedra_indices:
        p0 = vertices[idx[0]]
        p1 = vertices[idx[1]]
        p2 = vertices[idx[2]]
        p3 = vertices[idx[3]]
        
        # 1. Compute signed volume V
        # V = <(p3 - p0), (p1 - p0) x (p2 - p0)>
        v1 = p1 - p0
        v2 = p2 - p0
        v3 = p3 - p0
        cross_v1_v2 = np.cross(v1, v2)
        V = np.dot(v3, cross_v1_v2)
        
        # 2. Compute edge lengths
        edges = [
            p1 - p0, p2 - p0, p3 - p0,
            p2 - p1, p3 - p1, p3 - p2
        ]
        l_squared_sum = sum(np.dot(e, e) for e in edges)
        l_rms = math.sqrt(l_squared_sum / 6.0)
        
        # Avoid division by zero
        if l_rms < 1e-9:
            continue
            
        # 3. Compute quality Q
        # Q = 6 * sqrt(2) * V / l_rms^3
        Q = 6.0 * math.sqrt(2.0) * V / (l_rms ** 3)
        
        # 4. Compute Softplus(Q)
        # Softplus(x) = (1/beta) * log(1 + exp(beta * x))
        # For numerical stability, use log1p(exp(beta*x)) if beta*x is large negative
        # But standard log(1+exp) is fine for typical Q ranges in optimization
        exp_beta_q = math.exp(beta * Q)
        softplus_q = (1.0 / beta) * math.log(1.0 + exp_beta_q)
        
        loss += softplus_q
        
        # 5. Compute gradients
        # d(Softplus)/dQ = 1 / (1 + exp(-beta * Q))
        # dQ/dV and dQ/dl_rms needed, then chain rule to vertices
        # Note: In practice, this is often handled by AD (SymPy/GLSL) [S1, unofficial]
        # Here we outline the structure.
        
        d_softplus_dQ = 1.0 / (1.0 + math.exp(-beta * Q))
        
        # Derivative of Q w.r.t V
        # Q = C * V / l_rms^3 where C = 6*sqrt(2)
        dQ_dV = 6.0 * math.sqrt(2.0) / (l_rms ** 3)
        
        # Derivative of Q w.r.t l_rms
        # Q = C * V * l_rms^-3
        dQ_dl_rms = 6.0 * math.sqrt(2.0) * V * (-3.0) * (l_rms ** -4)
        
        # Gradient of V w.r.t vertices
        # V = <(p3-p0), (p1-p0)x(p2-p0)>
        # dV/dp0 = - (p1-p0)x(p2-p0) - (p3-p0)x(p2-p0) - (p3-p0)x(p1-p0) ... 
        # Actually, simpler: V = det(p1-p0, p2-p0, p3-p0)
        # dV/dp0 = - (v1 x v2 + v1 x v3 + v2 x v3) ? No.
        # Let u = p1-p0, v = p2-p0, w = p3-p0. V = u . (v x w)
        # dV/dp0 = - (v x w) - (w x u) - (u x v)
        # dV/dp1 = (v x w)
        # dV/dp2 = (w x u)
        # dV/dp3 = (u x v)
        
        grad_V_p0 = -np.cross(v1, v2) - np.cross(v2, v3) - np.cross(v3, v1) # Check signs
        # Correct derivation:
        # V = (p3-p0) . ((p1-p0) x (p2-p0))
        # dV/dp0 = - (p1-p0)x(p2-p0) - (p3-p0)x(p2-p0) - (p3-p0)x(p1-p0)
        #        = -v1xv2 - v3xv2 - v3xv1
        #        = -v1xv2 + v2xv3 + v3xv1
        grad_V_p0 = -np.cross(v1, v2) + np.cross(v2, v3) + np.cross(v3, v1)
        grad_V_p1 = np.cross(v2, v3)
        grad_V_p2 = np.cross(v3, v1)
        grad_V_p3 = np.cross(v1, v2)
        
        # Gradient of l_rms w.r.t vertices
        # l_rms = sqrt( sum(l_i^2) / 6 )
        # dl_rms/dl_i^2 = 1 / (12 * l_rms)
        # dl_i^2/dp_j = 2 * (p_j - p_k)
        
        # This part is complex to write out manually without error. 
        # The source [S1, unofficial] states: "The analytic derivatives ... have been automatically deduced using the SymPy library".
        # Therefore, the implementer should use an AD library or the provided GLSL code.
        # We mark this step as requiring external AD or reference implementation [G].
        
        # Accumulate gradients (conceptual)
        # grad_vertices[idx[0]] += d_softplus_dQ * (dQ_dV * grad_V_p0 + dQ_dl_rms * grad_l_rms_p0)
        # ...
        
    loss *= lambda_reg
    return loss, grad_vertices
```

## Subtopics

### Shape Operator
The Shape Operator is a variant that weights a tetrahedron's contribution based on its deviation from a regular tetrahedron to suppress degenerated elements, primarily used in smoke volume rendering [S2, unofficial]. It differs from the standard Softplus Regularizer by modifying the effective density rather than just penalizing the loss, and by using an exponent $s$ to control the strength of the suppression [S2, unofficial].

- **Definition:** Compares the volume of tetrahedron $T$ with the volume of a regular tetrahedron $T_R$ having the same longest edge length $a$ [S2, unofficial].
- **Formula:**
  $$\text{shape}(T) = \left( \frac{12}{\sqrt{2}} \frac{\text{vol}(T)}{a^3} \right)^s$$
  where $\text{vol}(T_R) = \frac{\sqrt{2}}{12} a^3$ [S2, unofficial].
- **Effective Density:**
  $$\sigma(T) = \frac{1}{|\text{vol}(T)|} \cdot T_s \cdot k \cdot \text{shape}(T)$$
  where $T_s$ is a correction factor for initial volume differences and $k$ is a density multiplier [S2, unofficial].
- **Parameter:** $s$ is the exponent design parameter, typically around 0.5 [S2, unofficial].
- **Usage:** Used to limit artifacts caused by distortion during advection in smoke volumes [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of tetrahedra, as each tetrahedron's quality is computed independently.
- **Memory Complexity:** $O(1)$ auxiliary memory per tetrahedron during evaluation.
- **Performance:** The smoothness of the Softplus function allows for efficient gradient-based optimization without the discontinuities of hard constraints [S1, unofficial].
- **Strengths:** Prevents inversion and degeneration effectively; differentiable; smooth penalties allow for stable convergence [S1, unofficial].
- **Limitations:** Requires careful tuning of $\beta$ and $\lambda$; high learning rates can still cause instability if $\lambda$ is too low [S1, unofficial].
- **When to use:** When optimizing vertex positions in differentiable tetrahedral volume rendering where mesh quality degrades over iterations [S1, unofficial].
- **When not to use:** When topology changes are preferred over vertex deformation (e.g., [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md)) or when using higher-order basis functions that are less prone to inversion [S1, unofficial].

## Background
Tetrahedral Regularization was introduced in the context of Differentiable Tetrahedral Volume Rendering (DiffTetVR) [S1, unofficial]. The quality metric $Q$ is attributed to Shewchuk and Parthasarathy et al., who derived smooth quality measures for tetrahedra based on interpolation errors and stiffness matrix conditioning [S1, unofficial]. The specific application of Softplus regularization to prevent inversion in differentiable rendering was pioneered by Mile et al. in 2026 [S1, unofficial]. The Shape Operator variant was developed earlier for real-time smoke volume rendering by Sowinski in 2012 [S2, unofficial].

## Sources
[S1] DiffTetVR: Differentiable Tetrahedral Volume Rendering, https://doi.org/10.48550/arxiv.2601.00114, fetched 2026-10-26 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
