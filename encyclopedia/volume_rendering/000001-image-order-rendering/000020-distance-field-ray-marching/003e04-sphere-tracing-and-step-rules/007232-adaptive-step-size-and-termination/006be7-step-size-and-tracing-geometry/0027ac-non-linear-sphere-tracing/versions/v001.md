# Non-Linear Sphere Tracing

Also called: Non-linear sphere tracing for deformed SDFs, RaymarchNonlinear, Rendering deformed signed distance fields

## Overview
Non-Linear Sphere Tracing (NLST) is a ray marching technique for directly rendering Signed Distance Fields (SDFs) that have been subjected to non-linear spatial deformations [S1]. It solves the problem that standard sphere tracing fails on deformed SDFs because the deformation invalidates the Euclidean distance metric, and computing the inverse deformation $D^{-1}$ is often impossible or unknown [S1]. Unlike [Sphere Tracing](1674-sphere-tracing.md), which assumes a static metric space, NLST formulates the ray traversal as an initial value problem (IVP) solved via numerical integration of an ordinary differential equation (ODE), allowing it to follow the curved ray trajectory in undeformed space while maintaining large step sizes [S1]. It differs from [Cone Intersection](1553-cone-intersection.md) and [Segment Tracing](1675-segment-tracing.md) by explicitly modeling the deformation's Jacobian to correct the step direction, rather than using geometric bounds or linear approximations [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A signed distance function $S(\hat{\mathbf{x}})$ defined in undeformed (object) space $\hat{\mathbf{x}}$; a forward deformation function $D(\hat{\mathbf{x}})$ mapping object space to deformed space; a ray origin $\mathbf{p}$ and direction $\boldsymbol{\omega}$ in deformed space; a maximum step count $N_{max}$; a hit threshold $\epsilon$; and a maximum travel distance $D_{max}$.
*   **Output:** A hit structure containing the deformed position $\mathbf{x}_{hit}$, the corrected normal $\mathbf{n}_{hit}$, and the distance traveled.

**Data Structures:**
The implementation uses a `SceneHit` structure to track the ray state:
*   `position`: `float3` (current position in undeformed space $\hat{\mathbf{x}}$).
*   `normal`: `float3` (current normal estimate, initialized to zero).
*   `distanceTravelled`: `float` (accumulated arc-length $\hat{s}$ in undeformed space).
*   `hit`: `bool` (flag indicating surface intersection).

**Preprocessing (Hull-based Initialization):**
To avoid computing the global inverse deformation $D^{-1}$ for the ray origin $\mathbf{p}$, NLST uses a coarse hull mesh [S1]:
1.  Generate a triangle mesh hull enclosing the implicit surface using marching cubes at a low resolution (e.g., $32^3$ grid) [S1].
2.  Deform this hull using the forward deformation $D$ [S1].
3.  Rasterize the deformed hull.
4.  For each pixel, retrieve the deformed position and transform it back to undeformed space via barycentric interpolation across the corresponding hull face [S1]. This provides an approximate initial undeformed position $\hat{\mathbf{x}}_0$ [S1].

**Algorithm Steps:**
1.  **Initialize:** Set `sceneHit.position` to $\hat{\mathbf{x}}_0$ (from hull initialization or $D^{-1}(\mathbf{p})$ if available) [S1][S2, unofficial]. Set `sceneHit.distanceTravelled` = 0. Set `sceneHit.hit` = false. Set `sceneHit.normal` = $(0,0,0)$ [S2, unofficial].
2.  **Loop:** For $i$ from 0 to $N_{max} - 1$:
    a.  **Evaluate SDF:** Compute $h = S(\text{sceneHit.position})$ [S2, unofficial].
    b.  **Check Hit:** If $h \le \epsilon$, set `sceneHit.hit` = true. Compute the standard SDF normal $\mathbf{n} = \nabla S(\text{sceneHit.position})$ using finite differences. Compute the Jacobian $J$ of the deformation $D$ at the hit position. Compute $J^{-1}$. Set `sceneHit.normal` = $\text{normalize}((J^{-1})^T \mathbf{n})$. Break loop [S2, unofficial].
    c.  **Check Max Distance:** If `sceneHit.distanceTravelled` $\ge D_{max}$, break loop [S2, unofficial].
    d.  **Compute Vector Field:** Calculate the unit tangent vector in undeformed space:
        $$ \boldsymbol{\omega}(\hat{\mathbf{x}}) = \frac{J_D^{-1}(\hat{\mathbf{x}}) \boldsymbol{\omega}_0}{\|J_D^{-1}(\hat{\mathbf{x}}) \boldsymbol{\omega}_0\|} $$
        where $\boldsymbol{\omega}_0$ is the initial ray direction in deformed space [S1][S2, unofficial].
    e.  **Integrate (Runge-Kutta):** Advance the position using a 4th-order Runge-Kutta integrator with step size $h$:
        $$ k_1 = \boldsymbol{\omega}(\text{sceneHit.position}, \boldsymbol{\omega}_0) $$
        $$ k_2 = \boldsymbol{\omega}(\text{sceneHit.position} + 0.5 h k_1, \boldsymbol{\omega}_0) $$
        $$ k_3 = \boldsymbol{\omega}(\text{sceneHit.position} + 0.5 h k_2, \boldsymbol{\omega}_0) $$
        $$ k_4 = \boldsymbol{\omega}(\text{sceneHit.position} + h k_3, \boldsymbol{\omega}_0) $$
        $$ \text{sceneHit.position} = \text{sceneHit.position} + \frac{h}{6}(k_1 + 2k_2 + 2k_3 + k_4) $$
        [S2, unofficial].
    f.  **Update Distance:** `sceneHit.distanceTravelled` += $h$ [S2, unofficial].
3.  **Final Transform:** If `sceneHit.hit` is true, transform `sceneHit.position` to deformed space using $D(\text{sceneHit.position})$ [S1]. Recompute camera-space depth for depth testing [S1].

**Jacobian Calculation:**
The Jacobian $J$ of the deformation $D$ is computed using central finite differences with step size $h_{diff}$ (e.g., 0.0001) [S2, unofficial]:
$$ J_{ij} = \frac{D_i(\mathbf{x} + h_{diff}\mathbf{e}_j) - D_i(\mathbf{x} - h_{diff}\mathbf{e}_j)}{2 h_{diff}} $$
where $\mathbf{e}_j$ are the basis vectors [S2, unofficial].

**Parameters:**
*   $N_{max}$ (MAX_STEPS): Integer, default 200 [S2, unofficial].
*   $\epsilon$ (THRESHOLD): Float, default 0.0001 [S2, unofficial].
*   $D_{max}$ (MAX_DISTANCE): Float, default 400.0 [S2, unofficial].
*   $h_{diff}$ (NORMAL_DIFFERENTIAL): Float, default 0.0001 [S2, unofficial].
*   Deformation parameters (e.g., $\alpha$ for sinusoidal deformation): Default 1.0 [S2, unofficial].

**Pitfalls:**
*   **Singular Jacobian:** If the deformation causes $J_D$ to be singular or near-singular, $J_D^{-1}$ becomes unstable [S2, unofficial]. Deformations must be locally foldover-free [S1].
*   **Inverse Requirement:** The method requires $D^{-1}$ at least once for the initial seed $\hat{\mathbf{x}}_0$. If $D^{-1}$ does not exist (e.g., non-bijective deformations), the hull-based initialization is mandatory [S1].
*   **Concave Objects:** Limited $N_{max}$ may cause missed intersections for concave objects or grazing angles [S1].
*   **Performance:** SDF evaluations are expensive. Naive step reduction degrades performance significantly [S1].

## Subtopics

### Hull-based initialization for non-linear sphere tracing
A variant that avoids computing the global inverse deformation $D^{-1}$ by using a coarse marching cubes mesh. The hull is deformed, rasterized, and the deformed position is mapped back to object space via barycentric interpolation to provide the initial ray start point $\hat{\mathbf{x}}_0$. This enables the use of deformations where $D^{-1}$ is unknown or non-existent, such as complex skinning or sculpting brushes [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by SDF evaluations and Jacobian inversions per step. The Runge-Kutta method requires multiple vector field evaluations per step but no additional SDF lookups for sub-stepping, preserving the performance advantage over naive ray marching [S1].
*   **Memory Complexity:** Low, requiring storage for the SDF and the deformation function. The hull mesh is small and transient [S1].
*   **Performance:** On an NVIDIA Titan RTX at 1080p, NLST takes ~10-13ms for complex scenes (Dinosaur, Head, Elephant), significantly faster than high-resolution Marching Cubes (e.g., S=1024 takes >9s) [S1]. It is more than twice as fast as Marching Cubes at the lowest resolution (S=32) while preserving thin features [S1].
*   **Strengths:** Preserves infinite resolution and sharp features; handles non-linear deformations (LBS, FFDs, Kelvinlets) without artifacts; avoids inverse deformation computation via hull initialization [S1].
*   **Limitations:** Requires $D$ to be a space deformation (evaluable at arbitrary points); fails if $J_D$ is singular; performance degrades at grazing angles or for highly concave objects with limited steps; does not support collision detection in deformed space due to lack of arbitrary point evaluation [S1].

## Background
NLST was introduced by Dario Seyb, Alec Jacobson, Derek Nowrouzezahrai, and Wojciech Jarosz in their paper "Non-linear sphere tracing for rendering deformed signed distance fields" (2020) [S1]. It builds on Barr's [1984] formulation for deforming parametric curves/surfaces by casting sphere tracing as an ODE initial value problem [S1]. The method addresses the limitations of meshing-based deformation (loss of detail, recomputation) and previous implicit deformation techniques that required inverse deformations or suffered from discontinuities [S1]. It has been implemented in prototype modeling applications using the Unity3D engine [S1].

## Sources
[S1] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
[S2] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
