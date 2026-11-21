# Matrix-Free PCG Solver
Also called: Implicit normal operator PCG, Triangle-Structured Matrix-Free PCG Solver, Vertex-Block Preconditioned PCG

## Overview
The Matrix-Free PCG Solver is a method for solving the global normal system $Hx=g$ arising in closed-form radiance-field optimization [S1, unofficial]. It evaluates the matrix-vector product $Hu$ directly from rasterization records without constructing the global observation or normal matrices $H$ [S1, unofficial]. Unlike factorization-based solvers that explicitly assemble and decompose the normal matrix, this variant maintains smaller GPU memory by never materializing $A$ and $H$ [S1, unofficial]. It builds on the CGLS Solver by providing a numerically stable solution to the least squares problem while avoiding the ill-conditioning of explicit normal equation assembly [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Set of calibrated RGB images $\{I_k\}_{k=1}^K$.
*   Rasterization records $R$ containing, for each valid pixel $p_{k,r}$ in view $k$:
    *   Triangle vertex indices $\{i_{s,k,r}\}_{s=0}^2$.
    *   Observation vectors $\{\phi_{s,k,r}\}_{s=0}^2$ (SH basis evaluations at the pixel).
    *   Weight $w_{k,r}$ (grazing-angle down-weighting).
*   Global SH regularizer $\Lambda$, composed of vertex blocks $\Lambda_i \in \mathbb{R}^{q \times q}$ for each vertex $i$.
*   Tolerance $\delta$: Prescribed relative residual tolerance for PCG termination.
*   $T_{max}$: Maximum iteration count for PCG (typically 300) [S1, unofficial].

**Outputs:**
*   Estimated appearance coefficients $X^* \in \mathbb{R}^{q N_v \times 3}$, where $N_v$ is the number of vertices and $q$ is the SH dimension per channel.

### Data Structures
*   **Vertex Blocks:** The solution vector $X$ and right-hand side $G$ are partitioned by vertex. $G_i \in \mathbb{R}^{q \times 3}$ stores the accumulated RHS for vertex $i$.
*   **Preconditioner Blocks:** $P_i \in \mathbb{R}^{q \times q}$ stores the diagonal block of the normal matrix for vertex $i$.
*   **Cholesky Factors:** $L_i \in \mathbb{R}^{q \times q}$ such that $P_i = L_i L_i^\top$.
*   **PCG Vectors:** $X^{(\tau)}$ (solution), $r$ (residual), $p$ (search direction), $z$ (preconditioned residual), $h$ (normal product result).

### Steps

#### 1. TriangleSetup
Construct the right-hand side $G$ and the vertex-block preconditioner $P$.

1.  Initialize $G_i = 0$ for all vertices $i=1 \dots N_v$.
2.  Initialize $P_i = \Lambda_i$ for all vertices $i=1 \dots N_v$.
3.  For each view $k = 1 \dots K$:
    4.  For each valid pixel $p_{k,r}$ in view $k$:
        5.  For each vertex $s \in \{0, 1, 2\}$ of the covering triangle:
            6.  Let $i_{s,k,r}$ be the vertex index.
            7.  Update RHS: $G_{i_{s,k,r}} \leftarrow G_{i_{s,k,r}} + w_{k,r} \phi_{s,k,r} I_k(p_{k,r})^\top$.
            8.  Update Preconditioner: $P_{i_{s,k,r}} \leftarrow P_{i_{s,k,r}} + w_{k,r} \phi_{s,k,r} \phi_{s,k,r}^\top$.
9.  For each vertex $i = 1 \dots N_v$:
    10. Compute Cholesky factor $L_i$ such that $P_i = L_i L_i^\top$.
11. Return $G$ and $\{L_i\}_{i=1}^{N_v}$.

#### 2. TriangleNormal(u)
Evaluate the matrix-vector product $h = Hu$ without forming $H$.

1.  For each vertex $i = 1 \dots N_v$:
    2.  Initialize $h_i = \Lambda_i u_i$.
3.  For each view $k = 1 \dots K$:
    4.  For each valid pixel $p_{k,r}$ in view $k$:
        5.  For each vertex $s \in \{0, 1, 2\}$:
            6.  Let $i_{s,k,r}$ be the vertex index.
            7.  Update: $h_{i_{s,k,r}} \leftarrow h_{i_{s,k,r}} + w_{k,r} \phi_{s,k,r} \left( \sum_{t=0}^2 \phi_{t,k,r}^\top u_{i_{t,k,r}} \right)$.
8.  Assemble $h$ from blocks $h_i$.
9.  Return $h$.

#### 3. PCG Loop (per channel $\tau \in \{R, G, B\}$)
Solve $H X^{(\tau)} = G^{(\tau)}$ using Preconditioned Conjugate Gradients.

1.  Initialize $X^{(\tau)} = 0$.
2.  Set $g \leftarrow G^{(\tau)}$, $r \leftarrow g$, $\eta \leftarrow \|g\|_2^2$.
3.  Solve $P z = r$ for $z$: For each vertex $i$, solve $L_i L_i^\top z_i = r_i$ using forward/backward substitution.
4.  Set $p \leftarrow z$, $\sigma \leftarrow r^\top z$.
5.  For $t = 1 \dots T_{max}$:
    6.  $h \leftarrow \text{TriangleNormal}(p)$.
    7.  $\alpha \leftarrow \sigma / (p^\top h)$.
    8.  $X^{(\tau)} \leftarrow X^{(\tau)} + \alpha p$.
    9.  $r \leftarrow r - \alpha h$.
    10. If $\|r\|_2^2 \le \delta^2 \eta$, break.
    11. Solve $P z = r$ for $z$ (using $\{L_i\}$).
    12. $\sigma_{new} \leftarrow r^\top z$.
    13. $\beta \leftarrow \sigma_{new} / \sigma$.
    14. $p \leftarrow z + \beta p$.
    15. $\sigma \leftarrow \sigma_{new}$.
16. End For.

### Formulas
*   **RHS Accumulation:** $G_{i_{s,k,r}} \leftarrow G_{i_{s,k,r}} + w_{k,r} \phi_{s,k,r} I_k(p_{k,r})^\top$ [S1, unofficial].
*   **Preconditioner Accumulation:** $P_{i_{s,k,r}} \leftarrow P_{i_{s,k,r}} + w_{k,r} \phi_{s,k,r} \phi_{s,k,r}^\top$ [S1, unofficial].
*   **Cholesky Decomposition:** $P_i = L_i L_i^\top$ [S1, unofficial].
*   **Regularization Term:** $h_i \leftarrow \Lambda_i u_i$ [S1, unofficial].
*   **Pixel Contribution to Normal Product:** $h_{i_{s,k,r}} \leftarrow h_{i_{s,k,r}} + w_{k,r} \phi_{s,k,r} \left( \sum_{t=0}^2 \phi_{t,k,r}^\top u_{i_{t,k,r}} \right)$ [S1, unofficial].
*   **Termination Condition:** $\sum_{i=1}^{N_v} \|G_i^{(\tau)} - H_i X_i^{(\tau)}\|_2^2 \le \delta \sum_{i=1}^{N_v} \|G_i^{(\tau)}\|_2^2$ [S1, unofficial].

### Parameters
*   $\delta$: Relative residual tolerance. Tighter tolerances reduce final residual error but require more iterations [S1, unofficial].
*   $T_{max}$: Maximum iterations. Set to 300 in evaluations to prevent infinite loops on ill-conditioned systems [S1, unofficial].

### Edge Cases and Pitfalls
*   **Ill-conditioning:** Without adaptive SH regularization, weakly supported SH components cause ill-conditioning, making the unregularized system difficult to solve within $T_{max}$ iterations [S1, unofficial].
*   **Memory:** The method processes views sequentially during matrix-free evaluation, so peak memory does not grow with the number of views $K$ [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** $M_{GPU} = O(N_v q + HW q)$ [S1, unofficial]. Peak memory depends on vertices and image resolution, not the number of views $K$ [S1, unofficial].
*   **Computational Complexity:**
    *   Setup: $O(KHWq^2 + N_v q^3)$ [S1, unofficial].
    *   Per Iteration: $O(KHWq + N_v q^2)$ [S1, unofficial].
    *   Total solve time grows linearly with $K$ [S1, unofficial].
*   **Strengths:** Maintains smaller GPU memory than factorization-based solvers by never materializing $A$ and $H$ [S1, unofficial]. The vertex-block preconditioner preserves coupling among all SH coefficients of the same vertex, improving numerical efficiency over scalar Jacobi preconditioners [S1, unofficial].
*   **Limitations:** Requires $T_{max}$ iterations; convergence depends on the conditioning provided by adaptive SH regularization [S1, unofficial].
*   **Accuracy:** At tight stopping tolerances, the residual MAE approaches that of the explicitly assembled factorization-based solver, confirming high numerical accuracy [S1, unofficial].

## Background
The Matrix-Free PCG Solver was introduced in CF-Splat: Closed-Form Triangle Splatting with Mesh Priors [S1, unofficial]. It addresses the computational and memory costs of explicitly constructing the global normal matrix $H$ for large-scale scenes [S1, unofficial]. The method exploits the three-vertex support of each pixel to evaluate the normal operator directly from rasterization records [S1, unofficial].

## Sources
[S1] CF-Splat: Closed-Form Triangle Splatting with Mesh Priors, https://arxiv.org/abs/2610.22108, fetched 2026-10-29 (unofficial)
