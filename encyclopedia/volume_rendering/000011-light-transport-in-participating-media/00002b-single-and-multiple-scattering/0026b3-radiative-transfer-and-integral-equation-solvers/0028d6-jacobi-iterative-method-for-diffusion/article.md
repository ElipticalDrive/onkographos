# Jacobi Iterative Method for Diffusion

Also called: Computing Diffusion Radiance and Derivative, Diffusion Solver

## Overview
This method solves the discrete diffusion equation and its differential counterpart to compute diffusion radiance ($L_d^0$), its derivative ($\partial_\pi L_d^0$), first-order radiance ($\vec{L}_1^d$), and its derivative ($\partial_\pi \vec{L}_1^d$) [S1]. It is used to simulate fluid-like light diffusion in participating media, enabling the visualization of internal structures without global segmentation [S1, Child]. It sets itself apart from sibling methods like [Discrete Ordinates Method](1046-discrete-ordinates-method.md) by operating on a grid-based iterative solver for the diffusion approximation rather than discrete angular directions, and from [Multiple Scattering Model (Integral Equation)](1177-multiple-scattering-model-integral-equation.md) by using a local Jacobi iteration on the differential form [S1].

## How it works
**Inputs:**
*   $L_d^0$: Diffusion radiance field (uninitialized or previous iteration).
*   $\partial_\pi L_d^0$: Derivative of diffusion radiance field.
*   $\vec{L}_1^d$: First-order radiance field.
*   $\partial_\pi \vec{L}_1^d$: Derivative of first-order radiance field.
*   $S$: Source term field.
*   $\kappa$: Optical coefficient field (scattering/absorption properties).
*   $\kappa_a$: Absorption coefficient field.
*   $h$: Grid spacing (implied by discrete gradient/Laplacian definitions).
*   $error\_threshold$: Convergence threshold for Mean-Square Error (MSE).

**Outputs:**
*   $L_d^0$: Converged diffusion radiance.
*   $\partial_\pi L_d^0$: Converged derivative of diffusion radiance.
*   $\vec{L}_1^d$: First-order radiance.
*   $\partial_\pi \vec{L}_1^d$: Derivative of first-order radiance.

**Data Structures:**
*   Fields $L_d^0$, $\partial_\pi L_d^0$, $\vec{L}_1^d$, $\partial_\pi \vec{L}_1^d$, $S$, $\kappa$, $\kappa_a$ are 3D arrays indexed by $(i, j, k)$ corresponding to voxel coordinates.
*   Values are floating-point numbers representing radiance or optical coefficients.

**Definitions:**
*   Discrete gradient approximation: $\nabla F_{ijk} \approx \frac{F_{i+1,j,k} - F_{i-1,j,k}}{2h}$ [S1].
*   Discrete Laplacian approximation: $\nabla^2 F_{ijk} \approx \frac{F_{i+1,j,k} + F_{i-1,j,k} - 2F_{i,j,k}}{h^2}$ [S1].
*   $\Sigma^1_{ijk}, \Sigma^2_{ijk}, \Sigma^6_{ijk}$: Intermediate terms defined in the source's Appendix B, derived from the expansion of $\nabla \partial_\pi \kappa \cdot \nabla L_d^0$, $\nabla \kappa \cdot \nabla \partial_\pi L_d^0$, and other diffusion terms [S1]. The exact expansion of these $\Sigma$ terms is not provided in the main text, but they are required for Equation 25 [S1].

**Steps:**
1.  **Initialization:** If $L_d^0$, $\vec{L}_1^d$, $\partial_\pi L_d^0$, and $\partial_\pi \vec{L}_1^d$ are uninitialized, assign them random values [S1].
2.  **Solve for Diffusion Radiance ($L_d^0$):**
    *   Repeat until convergence:
        *   Compute $L^{new}_{ijk}$ using the discrete diffusion equation (Equation 25):
            $$ L^{new}_{ijk} = \frac{1}{h^2(\kappa_a)_{ijk} + 6\kappa_{ijk}} \left[ \frac{1}{4}(\Sigma^1_{ijk} + \Sigma^2_{ijk}) + \kappa_{ijk} \Sigma^3_{ijk} - h^2(\kappa_a)_{ijk} L^0_{d,ijk} + h^2 \Sigma^6_{ijk} \right] $$
            *(Note: The source provides the formula for $\partial_\pi L_d^0$ explicitly as Equation 25 in the notes, but Algorithm 3 uses Equation 25 for $L^{new}$. The note formula $\partial_\pi L_d^0 = [...]$ is structurally identical to a Jacobi update for the derivative. The standard diffusion equation for $L_d^0$ typically follows a similar structure. Based on the note's explicit formula for $\partial_\pi L_d^0$ and the algorithm's structure, we assume the update for $L_d^0$ uses the corresponding diffusion equation terms. However, the source explicitly labels the complex formula in the notes as $\partial_\pi L_d^0$. Algorithm 3 line 6 says $L^{new} \leftarrow$ Equation 25. The note says $\partial_\pi L_d^0 = [...]$ is Equation 25. This is a contradiction in the source's labeling. We follow Algorithm 3's logic: Step 6 solves for $L_d^0$. The note's formula is likely for the derivative step or the source has a typo. Given the note explicitly defines the formula for $\partial_\pi L_d^0$, and Algorithm 3 computes $\partial_\pi L^{new}$ in Step 12 using Equation 29, we must infer the formula for $L_d^0$. Standard diffusion equation: $-\nabla \cdot (\kappa \nabla L_d^0) + \kappa_a L_d^0 = S$. The Jacobi update for $L_d^0$ would involve neighbors. Without the explicit Equation 25 for $L_d^0$ in the text, we rely on the structure of the derivative update and standard discretization. However, the prompt requires exact facts. The note says: "$\partial_\pi L_d^0 = [1 / (h^2(\kappa_a)_{ijk} + 6\kappa_{ijk})] * [...]$". Algorithm 3 computes $L^{new}$ (for $L_d^0$) using Eq 25, then $\vec{L}_1^d$ using Eq 5, then $\partial_\pi L^{new}$ (for $\partial_\pi L_d^0$) using Eq 29. The note formula is labeled $\partial_\pi L_d^0$. It is highly probable the note formula IS the update for $\partial_\pi L_d^0$ (Eq 29) or the source text has swapped labels. We will use the provided formula for the derivative step as it is explicitly defined in the notes, and describe the $L_d^0$ step as solving the discrete diffusion equation using the Jacobi method [S1].)*
        *   Calculate error: $error = MSE(L^{new}, L_d^0)$ [S1].
        *   Update: $L_d^0 \leftarrow L^{new}$ [S1].
        *   Stop when $error < error\_threshold$ [S1].
3.  **Compute First-Order Radiance ($\vec{L}_1^d$):**
    *   Compute $\vec{L}_1^d$ using Equation 5 [S1]. (Equation 5 is not provided in the notes; it is a direct computation from $L_d^0$ and $\kappa$) [S1].
4.  **Solve for Derivative of Diffusion Radiance ($\partial_\pi L_d^0$):**
    *   Repeat until convergence:
        *   Compute $\partial_\pi L^{new}_{ijk}$ using the differential diffusion equation (Equation 29). The note provides the explicit formula for $\partial_\pi L_d^0$ which matches the structure of a Jacobi update:
            $$ \partial_\pi L^{new}_{ijk} = \frac{1}{h^2(\kappa_a)_{ijk} + 6\kappa_{ijk}} \left[ \frac{1}{4}(\Sigma^1_{ijk} + \Sigma^2_{ijk}) + \kappa_{ijk} \Sigma^3_{ijk} - h^2(\kappa_a)_{ijk} \partial_\pi L^0_{d,ijk} + h^2 \Sigma^6_{ijk} \right] $$
            Where $\Sigma^1, \Sigma^2, \Sigma^6$ are the discrete gradient/Laplacian interaction terms for the derivative [S1].
        *   Calculate error: $error = MSE(\partial_\pi L^{new}, \partial_\pi L_d^0)$ [S1].
        *   Update: $\partial_\pi L_d^0 \leftarrow \partial_\pi L^{new}$ [S1].
        *   Stop when $error < error\_threshold$ [S1].
5.  **Compute Derivative of First-Order Radiance ($\partial_\pi \vec{L}_1^d$):**
    *   Compute $\partial_\pi \vec{L}_1^d$ using Equation 30 [S1]. (Equation 30 is not provided in the notes) [S1].
6.  **Return:** $L_d^0, \vec{L}_1^d, \partial_\pi L_d^0, \partial_\pi \vec{L}_1^d$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DiffusionSolver(L0d, dL0d, L1d, dL1d, S, kappa, kappa_a, h, error_threshold):
    # 1. Initialization
    if L0d is uninitialized or L1d is uninitialized or dL0d is uninitialized or dL1d is uninitialized:
        L0d = random_assignment()
        L1d = random_assignment()
        dL0d = random_assignment()
        dL1d = random_assignment()

    # 2. Solve for Diffusion Radiance L0d
    while True:
        # Equation 25: Discrete diffusion equation for L0d
        # Note: The explicit formula for L0d update is not in the notes, 
        # but the structure is a Jacobi iteration on the diffusion equation.
        # We assume standard discretization of -div(kappa * grad(L0d)) + kappa_a * L0d = S
        # Lnew[i,j,k] = (S[i,j,k] + kappa[i,j,k] * sum_neighbors(L0d) / h^2) / (kappa_a[i,j,k] + 6*kappa[i,j,k]/h^2)
        # However, to be strictly faithful to the source which only gives the derivative formula explicitly:
        # We implement the update for L0d as solving the discrete diffusion equation.
        # Since the exact Equation 25 for L0d is missing from the provided text snippets, 
        # we note that it is solved iteratively [S1].
        # [G] Assuming standard 7-point stencil for Laplacian and source term S:
        Lnew = zeros_like(L0d)
        for i, j, k in voxels:
            # Neighbors sum for Laplacian
            laplacian = (L0d[i+1,j,k] + L0d[i-1,j,k] + L0d[i,j+1,k] + L0d[i,j-1,k] + L0d[i,j,k+1] + L0d[i,j,k-1] - 6*L0d[i,j,k]) / (h*h)
            # Diffusion term: div(kappa * grad L) approximated
            # Simplified Jacobi form for -div(kappa grad L) + kappa_a L = S
            # L_new = (S + kappa * laplacian_L) / (kappa_a + 6*kappa/h^2) is a common form if kappa is constant.
            # With variable kappa, it's more complex. The source uses Sigma terms.
            # Without Eq 25, we cannot write the exact L0d update. 
            # We will use the structure of the derivative update as a proxy for the method's iterative nature, 
            # but strictly, the L0d update formula is missing from the provided notes.
            # [G] Placeholder for missing Equation 25: Lnew[i,j,k] = solve_diffusion_step(L0d, kappa, kappa_a, S, h)
            Lnew[i,j,k] = solve_diffusion_step(L0d, kappa, kappa_a, S, h) # [G] Implementation detail for missing Eq 25
        
        error = mean_squared_error(Lnew, L0d)
        L0d = Lnew
        if error < error_threshold:
            break

    # 3. Compute L1d via Equation 5
    L1d = compute_L1d_via_Equation_5(L0d, kappa) # [G] Equation 5 not provided

    # 4. Solve for derivative dL0d
    while True:
        # Equation 29: Differential diffusion equation for dL0d
        # The note provides the explicit formula for this update
        dLnew = zeros_like(dL0d)
        for i, j, k in voxels:
            # Sigma terms depend on gradients of kappa and L0d
            # Sigma1, Sigma2, Sigma6 are defined in Appendix B [S1]
            # [G] Assuming Sigma terms are precomputed or computed on the fly from gradients
            sigma1 = compute_sigma1(kappa, L0d, h) # [G] From Appendix B
            sigma2 = compute_sigma2(kappa, L0d, h) # [G] From Appendix B
            sigma3 = compute_sigma3(kappa, L0d, h) # [G] From Appendix B
            sigma6 = compute_sigma6(kappa, L0d, h) # [G] From Appendix B
            
            denominator = h*h * kappa_a[i,j,k] + 6 * kappa[i,j,k]
            numerator = 0.25 * (sigma1 + sigma2) + kappa[i,j,k] * sigma3 - h*h * kappa_a[i,j,k] * dL0d[i,j,k] + h*h * sigma6
            dLnew[i,j,k] = numerator / denominator
            
        error = mean_squared_error(dLnew, dL0d)
        dL0d = dLnew
        if error < error_threshold:
            break

    # 5. Compute dL1d via Equation 30
    dL1d = compute_dL1d_via_Equation_30(dL0d, L1d, kappa) # [G] Equation 30 not provided

    return L0d, L1d, dL0d, dL1d
```

**Edge Cases and Pitfalls:**
*   **Uninitialized Values:** The method requires random initialization if fields are not provided [S1].
*   **Convergence:** The method stops when the MSE between iterations falls below `error_threshold` [S1].
*   **Missing Formulas:** Equations 5, 25 (for $L_d^0$), 30, and the definitions of $\Sigma^1, \Sigma^2, \Sigma^6$ are referenced but not fully detailed in the provided notes, requiring implementation from the source's Appendix B or other sections [S1].

## Subtopics

### [Iterative Radiative Transfer Equation (RTE) Solver for Light Diffusion](12647-iterative-radiative-transfer-equation-rte-solver-for-light-d.md)
A method to visualize structures and regions of interest in volumes without global segmentation by solving the Radiative Transfer Equation (RTE) iteratively to simulate fluid-like light diffusion based on light transport. It uses this Jacobi solver as a core component.

## Operating characteristics
*   **Time Complexity:** Iterative, dependent on the number of iterations required for convergence (MSE < threshold). Each iteration involves a pass over the volume grid [S1].
*   **Memory Complexity:** Requires storage for $L_d^0, \partial_\pi L_d^0, \vec{L}_1^d, \partial_\pi \vec{L}_1^d$, and intermediate fields ($S, \kappa, \kappa_a$) [S1].
*   **Strengths:** Efficiently computes diffusion radiance and its derivatives for participating media visualization [S1].
*   **Limitations:** Requires convergence of iterative solver; performance depends on `error_threshold` and grid resolution [S1].
*   **When to use:** When fluid-like light diffusion visualization is needed without global segmentation [S1, Child].
*   **When not to use:** When high-accuracy single scattering or non-diffusive regimes are required [S1].

## Background
The method is part of a differential diffusion theory for participating media [S1]. It was introduced in the paper "A Differential Diffusion Theory for Participating Media" [S1].

## Sources
[S1] A Differential Diffusion Theory for Participating Media, https://doi.org/10.1111/cgf.14956, fetched 2026-10-30
