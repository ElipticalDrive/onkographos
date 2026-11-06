# Tetrahedral Regularization

Also called: Softplus Quality Regularizer

## Overview
Tetrahedral Regularization is a penalty term added to the optimization loss to prevent tetrahedral elements from becoming inverted (negative volume) or degenerate (volume approaching zero) during the optimization of vertex positions and colors [S1, unofficial]. It operates by computing a smooth quality metric for each element and applying a differentiable softplus penalty, distinguishing it from topology-modifying methods like [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md) [S1, unofficial].

## How it works
**Inputs:**
- Set of tetrahedra $T$.
- For each tetrahedron $t \in T$, the positions of its four vertices $p_0, p_1, p_2, p_3 \in \mathbb{R}^3$ [S1, unofficial].
- Hyperparameters: $\lambda$ (regularization weight), $\beta$ (softplus steepness) [S1, unofficial].

**Outputs:**
- Scalar regularization loss value.
- Gradients of the loss with respect to vertex positions $p_i$ [S1, unofficial].

**Data Structures:**
- Vertices are stored as 3D vectors.
- Edge lengths are computed on-the-fly from vertex positions [S1, unofficial].

**Steps:**
1. For each tetrahedron $t \in T$:
   a. Compute the signed volume $V$ using the scalar triple product of the edges originating from $p_0$:
      $$V = \langle (p_3 - p_0), (p_1 - p_0) \times (p_2 - p_0) \rangle$$ [S1, unofficial].
   b. Compute the six edge lengths $l_i$ for $i=0..5$ corresponding to the pairs $(p_0,p_1), (p_0,p_2), (p_0,p_3), (p_1,p_2), (p_1,p_3), (p_2,p_3)$ [S1, unofficial].
   c. Compute the root-mean-square edge length $l_{rms}$:
      $$l_{rms} = \sqrt{\frac{1}{6} \sum_{i=0}^5 l_i^2}$$ [S1, unofficial].
   d. Compute the smooth quality measure $Q$:
      $$Q = 6\sqrt{2} \frac{V}{l_{rms}^3}$$ [S1, unofficial].
   e. Compute the softplus penalty:
      $$\text{Softplus}(Q) = \frac{1}{\beta} \log(1 + \exp(\beta Q))$$ [S1, unofficial].
   f. Accumulate the penalty to the total loss [S1, unofficial].
2. Multiply the accumulated sum by $\lambda$ and add to the image loss $\text{loss}_{\text{img}}$ [S1, unofficial].

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

def tetrahedral_regularization(vertices, tet_indices, lambda_reg=10.0, beta=10.0):
    """
    Computes tetrahedral regularization loss and gradients w.r.t. vertex positions.
    vertices: list of [x, y, z] tuples for all vertices.
    tet_indices: list of 4-tuples of vertex indices.
    """
    def cross(a, b):
        return [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]
    def dot(a, b):
        return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

    total_loss = 0.0
    grad_vertices = [[0.0, 0.0, 0.0] for _ in vertices]

    for idx in tet_indices:
        p = [vertices[i] for i in idx]
        p0, p1, p2, p3 = p

        v1, v2, v3 = [p1[j]-p0[j] for j in range(3)], [p2[j]-p0[j] for j in range(3)], [p3[j]-p0[j] for j in range(3)]
        cross_v1_v2 = cross(v1, v2)
        V = dot(v3, cross_v1_v2)

        edges_sq = [
            sum((p[i][j]-p[k][j])**2 for j in range(3))
            for i, k in [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]
        ]
        l_rms = math.sqrt(sum(edges_sq) / 6.0)
        if l_rms < 1e-9: continue

        Q = 6.0 * math.sqrt(2.0) * V / (l_rms ** 3)
        exp_bQ = math.exp(beta * Q)
        softplus_Q = (1.0 / beta) * math.log(1.0 + exp_bQ)
        total_loss += softplus_Q

        d_softplus_dQ = 1.0 / (1.0 + math.exp(-beta * Q))
        dQ_dV = 6.0 * math.sqrt(2.0) / (l_rms ** 3)
        dQ_dl_rms = -18.0 * math.sqrt(2.0) * V / (l_rms ** 4)

        cross_v3_v2 = cross(v3, v2)
        cross_v3_v1 = cross(v3, v1)

        dV_dp0 = [-cross_v1_v2[j] - cross_v3_v2[j] - cross_v3_v1[j] for j in range(3)]
        dV_dp1 = cross_v3_v2
        dV_dp2 = cross_v3_v1
        dV_dp3 = cross_v1_v2

        dl_rms_dS = 1.0 / (12.0 * l_rms)
        dl_rms_dp = [[0.0, 0.0, 0.0] for _ in range(4)]
        for i, k in [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]:
            dl2_dpi = [2.0 * (p[i][m] - p[k][m]) for m in range(3)]
            dl2_dpk = [-x for x in dl2_dpi]
            for m in range(3):
                dl_rms_dp[i][m] += dl_rms_dS * dl2_dpi[m]
                dl_rms_dp[k][m] += dl_rms_dS * dl2_dpk[m]

        for i in range(4):
            for m in range(3):
                grad_vertices[idx[i]][m] += d_softplus_dQ * (dQ_dV * dV_dp[i][m] + dQ_dl_rms * dl_rms_dp[i][m])

    total_loss *= lambda_reg
    return total_loss, grad_vertices
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
- **Time Complexity:** $O(N)$ where $N$ is the number of tetrahedra, as each tetrahedron's quality is computed independently [G].
- **Memory Complexity:** $O(1)$ auxiliary memory per tetrahedron during evaluation [G].
- **Performance:** The smoothness of the Softplus function allows for efficient gradient-based optimization without the discontinuities of hard constraints [S1, unofficial].
- **Strengths:** Prevents inversion and degeneration effectively; differentiable; smooth penalties allow for stable convergence [S1, unofficial].
- **Limitations:** High learning rates or low $\lambda$ can still cause instability if the penalization factor is insufficient [S1, unofficial].
- **When to use:** When optimizing vertex positions in differentiable tetrahedral volume rendering where mesh quality degrades over iterations [S1, unofficial].
- **When not to use:** When topology changes are preferred over vertex deformation (e.g., [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md)) [S1, unofficial].

## Background
Tetrahedral Regularization was introduced in the context of Differentiable Tetrahedral Volume Rendering (DiffTetVR) [S1, unofficial]. The quality metric $Q$ is attributed to Shewchuk and Parthasarathy et al., who derived smooth quality measures for tetrahedra based on interpolation errors and stiffness matrix conditioning [S1, unofficial]. The specific application of Softplus regularization to prevent inversion in differentiable rendering was pioneered by Mile et al. in 2026 [S1, unofficial]. The Shape Operator variant was developed earlier for real-time smoke volume rendering by Sowinski in 2012 [S2, unofficial].

## Sources
[S1] DiffTetVR: Differentiable Tetrahedral Volume Rendering, https://doi.org/10.48550/arxiv.2601.00114, fetched 2026-10-26 (unofficial)
[S2] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
