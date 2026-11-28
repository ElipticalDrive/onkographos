# Implicit Gradients for Forward LBS

Also called: Implicit Differentiation for LBS, Detached Jacobian Inverse

## Overview
This method computes differentiable point samples for volume rendering by solving a root-finding problem via implicit differentiation, allowing gradients to flow through the inverse of a forward Linear Blend Skinning (LBS) network. It solves the problem of back-propagating gradients through non-invertible or complex deformation networks by treating the canonical-to-observation mapping as an implicit function defined by a residual equation. It differs from the [Adjoint Method](789-adjoint-method.md) by avoiding the storage of all intermediate activations for a full backward pass; instead, it computes the gradient analytically using the inverse Jacobian of the deformation function, detached from the computational graph.

## How it works
**Inputs:**
- $\bar{\mathbf{x}}^{(i)}$: Sampled points in observation space (e.g., camera rays or depth samples).
- $\hat{\mathbf{x}}^{*(i)}$: Initial estimates of canonical correspondences, obtained via iterative root-finding (e.g., Newton-Raphson) [S1].
- $LBS_{\sigma_\omega}$: The forward LBS network parameterized by $\sigma_\omega$, mapping canonical points to observation space.
- $f_{\sigma_f}$: The SDF network parameterized by $\sigma_f$ (used in joint root-finding variant) [S1].

**Outputs:**
- $\hat{\mathbf{x}}$: Differentiable point samples in canonical space that satisfy the implicit constraint.
- Gradients $\frac{\partial \mathcal{L}}{\partial \sigma_\omega}$ (and $\frac{\partial \mathcal{L}}{\partial \sigma_f}$ in joint variant) computed via standard back-propagation from the photometric loss.

**Data Structures:**
- $\hat{\mathbf{x}}^{*}$: Tensor of shape $(N, 3)$, detached from the graph.
- $\mathbf{J}^*$: Jacobian matrix of shape $(3, 3)$ for single point, or $(4, 4)$ for joint root-finding, computed at $\hat{\mathbf{x}}^{*}$ and detached.

**Steps:**
1. **Initial Root-Finding:** For each observation point $\bar{\mathbf{x}}^{(i)}$, find a canonical point $\hat{\mathbf{x}}^{*(i)}$ such that the residual is minimized:
   $$ LBS_{\sigma_\omega}(\hat{\mathbf{x}}^{*(i)}) - \bar{\mathbf{x}}^{(i)} \approx 0 $$
   This is done using iterative numerical methods [S1].

2. **Jacobian Computation:** Compute the Jacobian of the LBS function with respect to the canonical input at the detached solution $\hat{\mathbf{x}}^{*}$:
   $$ \mathbf{J}^* = \frac{\partial LBS_{\sigma_\omega}}{\partial \hat{\mathbf{x}}}(\hat{\mathbf{x}}^*) $$
   This matrix is detached from the computational graph [S1].

3. **Implicit Gradient Correction:** Compute the differentiable sample $\hat{\mathbf{x}}$ by applying the implicit differentiation formula:
   $$ \hat{\mathbf{x}} = \hat{\mathbf{x}}^* - (\mathbf{J}^*)^{-1} \cdot \left( LBS_{\sigma_\omega}(\hat{\mathbf{x}}^{*(i)}) - \bar{\mathbf{x}}^{(i)} \right) $$
   Here, $\hat{\mathbf{x}}^*$ and $\mathbf{J}^*$ are treated as constants (detached), while the residual term allows the gradient to flow through the LBS network to $\sigma_\omega$ [S1].

4. **Back-Propagation:** Use $\hat{\mathbf{x}}$ as input to subsequent networks (e.g., SDF $f_{\sigma_f}$ and color networks). Compute the photometric loss $\mathcal{L}$ and perform standard back-propagation to obtain gradients with respect to $\sigma_\omega$ (and $\sigma_f$) [S1].

**Pitfalls:**
- **Detachment:** $\hat{\mathbf{x}}^*$ and $\mathbf{J}^*$ must be explicitly detached from the computational graph. If gradients flow through them, the method fails to provide the correct implicit gradient and may introduce instability [S1].
- **Jacobian Inversion:** The Jacobian $\mathbf{J}^*$ must be non-singular. Degenerate poses or singularities in the LBS network can make inversion unstable.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def implicit_gradients_forward_lbs(obs_points, lbs_network, initial_canonical_guess):
    """
    obs_points: Tensor of shape (N, 3), observation space points.
    lbs_network: Differentiable LBS function f_sigma_w.
    initial_canonical_guess: Tensor of shape (N, 3), detached canonical points x_hat_star.
    """
    # Step 1: Compute residual at detached guess
    # Note: initial_canonical_guess is detached, so this forward pass
    # does not contribute to the gradient graph for the correction term.
    lbs_output = lbs_network(initial_canonical_guess)
    residual = lbs_output - obs_points
    
    # Step 2: Compute Jacobian at detached guess
    # J_star = d(LBS) / d(x_hat) evaluated at x_hat_star
    # This requires computing the Jacobian-vector product or full Jacobian.
    # We detach the result to prevent gradient flow through the Jacobian computation.
    with torch.no_grad():
        # Compute Jacobian matrix J* of shape (N, 3, 3)
        # Assuming lbs_network(x) returns (N, 3)
        J_star = compute_jacobian(lbs_network, initial_canonical_guess)
        
        # Invert Jacobian. Add small epsilon for stability if needed.
        # J_inv shape: (N, 3, 3)
        J_inv = torch.inverse(J_star)
        
    # Step 3: Compute differentiable samples
    # x_hat = x_hat_star - J_inv * residual
    # The residual term is NOT detached, allowing gradients to flow through lbs_network
    # to sigma_omega during backprop.
    correction = torch.bmm(J_inv, residual.unsqueeze(-1)).squeeze(-1)
    differentiable_samples = initial_canonical_guess - correction
    
    return differentiable_samples
```

## Subtopics

### Implicit Gradients for Joint Root-finding
This variant extends the method to jointly solve for canonical points $\hat{\mathbf{x}}$ and depth $d$. It defines a joint residual function $g_{\sigma_f, \sigma_\omega}$ involving both the SDF $f_{\sigma_f}$ and the LBS $LBS_{\sigma_\omega}$. The differentiable samples $[\hat{\mathbf{x}}; d]$ are computed as:
$$ \begin{bmatrix} \hat{\mathbf{x}} \\ d \end{bmatrix} = \begin{bmatrix} \hat{\mathbf{x}}^* \\ d^* \end{bmatrix} - (\mathbf{J}^*)^{-1} \cdot g_{\sigma_f, \sigma_\omega}(\hat{\mathbf{x}}^*, d^*) $$
The Jacobian $\mathbf{J}^*$ is a block matrix:
$$ \mathbf{J}^* = \begin{bmatrix} \frac{\partial f_{\sigma_f}}{\partial \hat{\mathbf{x}}}(\hat{\mathbf{x}}^*) & 0 \\ \frac{\partial LBS_{\sigma_\omega}}{\partial \hat{\mathbf{x}}}(\hat{\mathbf{x}}^*) & -\mathbf{v} \end{bmatrix} $$
where $\mathbf{v}$ is the view direction. Gradients with respect to both $\sigma_f$ and $\sigma_\omega$ are computed from the photometric loss [S1].

## Operating characteristics
- **Time Complexity:** Dominated by the iterative root-finding for initial guesses and the Jacobian inversion ($O(1)$ for fixed dimension 3x3 or 4x4). The implicit correction is $O(N)$.
- **Memory Complexity:** Significantly lower than the [Adjoint Method](789-adjoint-method.md) as it does not store all intermediate activations. Only the detached initial guesses and Jacobians are needed.
- **Strengths:** Enables gradient flow through complex, non-invertible deformation networks without storing heavy activation histories. Allows optimization of pose and shape parameters ($\theta, \beta$) in SMPL models [S1].
- **Limitations:** Requires the Jacobian to be invertible. Numerical instability if the Jacobian is near-singular. The initial root-finding must be robust.

## Background
Originating from the ARAH paper (Wang et al., 2022), this method builds on implicit differentiation techniques used in [87] (likely "Neural Implicit Surface..." or similar literature on implicit gradients). It is used in animatable volume rendering of articulated human SDFs, particularly when optimizing pose and shape from monocular videos where SMPL estimations are noisy [S1].

## Sources
[S1] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
