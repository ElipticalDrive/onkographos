# Bézier-boundary gradient approximation

Also called: Bézier-boundary gradient approximation strategy, DisC-GS gradient approximation

## Overview
A gradient modification technique for 3D Gaussian Splatting that enables the optimization of control points defining discontinuous indicator functions (used for rendering edges and boundaries) by approximating the non-differentiable gradient of a cubic Bézier curve. It solves the problem where standard backpropagation fails because the indicator function $g(\cdot)$ is piecewise constant, yielding zero gradients that cannot guide the learning of control point coordinates [S1]. It sets itself apart from standard Gaussian Splatting by explicitly handling discontinuities via Bézier curves, and from sibling methods like [Truncated Gradient Updates](18661-truncated-gradient-updates-for-3d-gaussian-splatting.md) by focusing on the geometric boundary of the curve rather than truncating magnitude.

## How it works
**Inputs and Outputs:**
*   **Input:** The current control point coordinates $c_{\text{curve}}$, the pixel location $p=(x_p, y_p)$, the current output of the Bézier curve function $g_{sc}(p)$, the gradient of the loss with respect to the curve output $\frac{\partial \mathcal{L}}{\partial g_{sc}}$, and the hyperparameter $\epsilon$ (small constant).
*   **Output:** The approximated gradient $\frac{\partial g}{\partial c_{\text{curve}}[0,0]}$ (and similarly for other coordinates) to be used in the optimizer step.

**Data Structures:**
*   $c_{\text{curve}}$: Attribute storing control point coordinates.
    *   For 2D Gaussian Splatting: Shape $\mathbb{R}^{4M \times 2}$.
    *   For 3D Gaussian Splatting: Shape $\mathbb{R}^{4M \times 3}$.
    *   $M$: Hyperparameter for the number of control points per Gaussian. Value: 3 [S1].
    *   $c_{\text{curve}}[i, j]$ denotes the $j$-th coordinate of the $i$-th control point.

**Method Steps:**
The method approximates $\frac{\partial g}{\partial c_{\text{curve}}[0,0]}$ using the chain rule:
$$ \frac{\partial g}{\partial c_{\text{curve}}[0,0]} = \frac{\partial g}{\partial g_{sc}} \times \frac{\partial g_{sc}}{\partial c_{\text{curve}}[0,0]} $$
where $g_{sc}(p)$ is the cubic Bézier curve function evaluated at $p$. The core difficulty is approximating $\frac{\partial g_{sc}}{\partial c_{\text{curve}}[0,0]}$.

1.  **Check Modification Necessity:**
    Determine if $g_{sc}(p)$ needs modification by checking if the current state is already optimal or saturated. If any of the following conditions hold, set $\frac{\partial g_{sc}}{\partial c_{\text{curve}}[0,0]} = 0$ and terminate:
    *   $\frac{\partial \mathcal{L}}{\partial g_{sc}} = 0$ (Optimal state).
    *   $\frac{\partial \mathcal{L}}{\partial g_{sc}} > 0$ and $g_{sc}(p) = 0$ (Already at minimum).
    *   $\frac{\partial \mathcal{L}}{\partial g_{sc}} < 0$ and $g_{sc}(p) = 1$ (Already at maximum).

2.  **Solve for Target Control Point:**
    If modification is needed, identify the target value $\phi$ for $c_{\text{curve}}[0,0]$ such that the Bézier curve passes through $p$. Solve the system of equations derived from the cubic Bézier definition for parameter $t$ and target $\phi$:
    $$ x_p = (1-t)^3 \phi + 3(1-t)^2 t c_{\text{curve}}[1,0] + 3(1-t)t^2 c_{\text{curve}}[2,0] + t^3 c_{\text{curve}}[3,0] $$
    $$ y_p = (1-t)^3 c_{\text{curve}}[0,1] + 3(1-t)^2 t c_{\text{curve}}[1,1] + 3(1-t)t^2 c_{\text{curve}}[2,1] + t^3 c_{\text{curve}}[3,1] $$
    Collect all real number solutions for $\phi$ into a set $S_\phi$. This is solved in $O(1)$ time [S1].

3.  **Approximate Gradient Based on $S_\phi$:**
    *   **No Side:** If $S_\phi = \emptyset$, set $\frac{\partial g_{sc}}{\partial c_{\text{curve}}[0,0]} = 0$. No real value of $c_{\text{curve}}[0,0]$ can cause the curve to pass through $p$ [S1].
    *   **Single Side:** If all solutions in $S_\phi$ lie on the same side of the current $c_{\text{curve}}[0,0]$, let $\phi_e$ be the solution nearest to $c_{\text{curve}}[0,0]$. Approximate the gradient via linear interpolation:
        $$ \frac{\partial g_{sc}}{\partial c_{\text{curve}}[0,0]} = \frac{g_{sc}(p) - \hat{g}_{sc}(p)}{\phi_e - c_{\text{curve}}[0,0] + \epsilon} $$
        where $\hat{g}_{sc}(p) = 1$ if $g_{sc}(p)=0$ else $0$. The epsilon $\epsilon$ is set to $10^{-5}$ if $\phi_e > c_{\text{curve}}[0,0]$ and $-10^{-5}$ otherwise [S1].
    *   **Both Sides:** If solutions exist on both left and right sides of $c_{\text{curve}}[0,0]$, let $\phi_1$ be the nearest solution from the left and $\phi_2$ be the nearest from the right. Approximate the gradient as the sum of two interpolation terms:
        $$ \frac{\partial g_{sc}}{\partial c_{\text{curve}}[0,0]} = \frac{g_{sc}(p) - \hat{g}_{sc}(p)}{\phi_1 - c_{\text{curve}}[0,0] + \epsilon_1} + \frac{g_{sc}(p) - \hat{g}_{sc}(p)}{\phi_2 - c_{\text{curve}}[0,0] + \epsilon_2} $$
        where $\epsilon_1, \epsilon_2$ follow the same sign rule as $\epsilon$ above [S1].

4.  **Final Gradient:**
    Multiply the result by $\frac{\partial g}{\partial g_{sc}}$ to obtain the final gradient for the control point.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bezier_boundary_gradient_approximation(
    c_curve,  # Array of shape (4*M, 2) or (4*M, 3)
    p,        # Pixel coordinate (x_p, y_p)
    g_sc,     # Current output of Bézier curve at p
    dL_dg_sc, # Gradient of loss w.r.t. g_sc
    M=3       # Hyperparameter for control points
):
    """
    Approximates the gradient of the discontinuous indicator function 
    with respect to the first control point's x-coordinate.
    """
    # 1. Check modification necessity
    if dL_dg_sc == 0:
        return 0.0
    if dL_dg_sc > 0 and g_sc == 0:
        return 0.0
    if dL_dg_sc < 0 and g_sc == 1:
        return 0.0

    # 2. Solve for target phi
    # Extract fixed control points for x and y
    # c_curve[0,0] is the variable we are solving for (phi)
    # c_curve[1,0], c_curve[2,0], c_curve[3,0] are fixed x-coords
    # c_curve[0,1], c_curve[1,1], c_curve[2,1], c_curve[3,1] are fixed y-coords
    
    c0_x = c_curve[0, 0]
    c1_x = c_curve[1, 0]
    c2_x = c_curve[2, 0]
    c3_x = c_curve[3, 0]
    
    c0_y = c_curve[0, 1]
    c1_y = c_curve[1, 1]
    c2_y = c_curve[2, 1]
    c3_y = c_curve[3, 1]
    
    x_p, y_p = p
    
    # We need to solve for t and phi such that:
    # x_p = (1-t)^3 * phi + 3(1-t)^2*t*c1_x + 3(1-t)*t^2*c2_x + t^3*c3_x
    # y_p = (1-t)^3*c0_y + 3(1-t)^2*t*c1_y + 3(1-t)*t^2*c2_y + t^3*c3_y
    
    # From the y-equation, we can solve for t (cubic equation in t)
    # y_p = c0_y*(1-t)^3 + c1_y*3t(1-t)^2 + c2_y*3t^2(1-t) + c3_y*t^3
    # This is a standard cubic Bezier evaluation for y.
    # We find roots t in [0, 1] for this equation.
    
    # Coefficients for cubic polynomial in t: At^3 + Bt^2 + Ct + D = 0
    # y(t) = c0_y(1 - 3t + 3t^2 - t^3) + 3c1_y(t - 2t^2 + t^3) + 3c2_y(t^2 - t^3) + c3_y t^3
    # y(t) = -c0_y + t(3c0_y - 3c1_y + c3_y) + t^2(-3c0_y + 6c1_y - 3c2_y) + t^3(c0_y - 3c1_y + 3c2_y - c3_y)
    # Wait, standard form: y(t) = (1-t)^3 c0 + 3(1-t)^2 t c1 + 3(1-t) t^2 c2 + t^3 c3
    # Let's expand:
    # (1-3t+3t^2-t^3)c0 + (3t-6t^2+3t^3)c1 + (3t^2-3t^3)c2 + t^3 c3
    # t^3: -c0 + 3c1 - 3c2 + c3
    # t^2: 3c0 - 6c1 + 3c2
    # t^1: -3c0 + 3c1
    # t^0: c0
    
    A_y = -c0_y + 3*c1_y - 3*c2_y + c3_y
    B_y = 3*c0_y - 6*c1_y + 3*c2_y
    C_y = -3*c0_y + 3*c1_y
    D_y = c0_y
    
    # Solve A_y t^3 + B_y t^2 + C_y t + (D_y - y_p) = 0
    coeffs_y = [A_y, B_y, C_y, D_y - y_p]
    t_roots = np.roots(coeffs_y)
    
    # Filter real roots in [0, 1]
    valid_t = []
    for t in t_roots:
        if np.isreal(t) and 0 <= np.real(t) <= 1:
            valid_t.append(np.real(t))
            
    S_phi = []
    for t in valid_t:
        # From x-equation: x_p = (1-t)^3 * phi + K
        # K = 3(1-t)^2*t*c1_x + 3(1-t)*t^2*c2_x + t^3*c3_x
        K = 3*(1-t)**2 * t * c1_x + 3*(1-t) * t**2 * c2_x + t**3 * c3_x
        denom = (1-t)**3
        if abs(denom) > 1e-9:
            phi = (x_p - K) / denom
            S_phi.append(phi)
            
    # 3. Approximate gradient
    if len(S_phi) == 0:
        d_g_sc_d_c0_x = 0.0
    else:
        # Check sides
        left_sols = [phi for phi in S_phi if phi < c0_x]
        right_sols = [phi for phi in S_phi if phi > c0_x]
        
        # Define target output for interpolation
        # If g_sc is 0, we want to move it to 1 (if gradient encourages increase)
        # If g_sc is 1, we want to move it to 0 (if gradient encourages decrease)
        # The formula uses g_sc(p) - g_hat_sc(p)
        # g_hat_sc(p) = 1 if g_sc(p)==0 else 0
        if g_sc == 0:
            g_hat_sc = 1
        else:
            g_hat_sc = 0
            
        numerator = g_sc - g_hat_sc
        
        if len(left_sols) == 0 and len(right_sols) > 0:
            # Single side (right)
            phi_e = min(right_sols) # Nearest to c0_x
            if phi_e > c0_x:
                eps = 1e-5
            else:
                eps = -1e-5
            d_g_sc_d_c0_x = numerator / (phi_e - c0_x + eps)
            
        elif len(right_sols) == 0 and len(left_sols) > 0:
            # Single side (left)
            phi_e = max(left_sols) # Nearest to c0_x
            if phi_e > c0_x:
                eps = 1e-5
            else:
                eps = -1e-5
            d_g_sc_d_c0_x = numerator / (phi_e - c0_x + eps)
            
        elif len(left_sols) > 0 and len(right_sols) > 0:
            # Both sides
            phi_1 = max(left_sols) # Nearest from left
            phi_2 = min(right_sols) # Nearest from right
            
            if phi_1 > c0_x: # Should not happen, but for safety
                eps_1 = 1e-5
            else:
                eps_1 = -1e-5
                
            if phi_2 > c0_x:
                eps_2 = 1e-5
            else:
                eps_2 = -1e-5
                
            term1 = numerator / (phi_1 - c0_x + eps_1)
            term2 = numerator / (phi_2 - c0_x + eps_2)
            d_g_sc_d_c0_x = term1 + term2
        else:
            # All solutions are exactly equal to c0_x? Or empty?
            # If S_phi is not empty but no left/right, they are equal.
            # This is a degenerate case. Set to 0.
            d_g_sc_d_c0_x = 0.0
            
    return dL_dg_sc * d_g_sc_d_c0_x
```

**Parameters:**
*   $\epsilon$: Small constant to prevent gradient explosion. Value: $10^{-5}$ if target $\phi >$ current $c_{\text{curve}}[0,0]$, else $-10^{-5}$ [S1].
*   $M$: Number of control points per Gaussian. Value: 3 [S1].
*   Initial learning rate for $c_{\text{curve}}$: $2 \times 10^{-4}$ [S1].

**Edge Cases and Pitfalls:**
*   **No Side:** If $S_\phi$ is empty, the gradient is set to 0. This occurs when the curve geometry cannot be adjusted by changing $c_{\text{curve}}[0,0]$ to pass through $p$ [S1].
*   **Precision:** The epsilon value is critical. Using a fixed sign based on the direction of movement prevents division by zero and stabilizes training [S1].
*   **Cloning/Splitting:** When Gaussians are cloned or split during densification, the new Gaussians inherit the same $c_{\text{curve}}$ attribute as the original [S1].

## Subtopics

### [Learned Static–Dynamic Partition](20807-learned-static-dynamic-partition.md)
A method to jointly optimize a binary gate $g_i$ and motion parameters to classify Gaussians as static or dynamic, using a hard-concrete relaxation with straight-through estimator. It solves the problem of incorrectly treating moving Gaussians as static, which leaves frozen copies of moving objects in the scene, by learning a sparse static-dynamic partition. It sets itself apart from sibling methods by explicitly maintaining a binary classification per Gaussian that is committed and frozen after training, enabling efficient rendering of static background without temporal overhead.

## Operating characteristics
*   **Time Complexity:** The core approximation step (solving for $t$ and $\phi$) is $O(1)$ per control point coordinate [S1].
*   **Memory Complexity:** Requires storing the control point attributes $c_{\text{curve}}$ with shape $\mathbb{R}^{4M \times 2}$ (2D) or $\mathbb{R}^{4M \times 3}$ (3D) per Gaussian [S1].
*   **Strengths:** Enables differentiable rendering of discontinuities (edges, boundaries) which standard Gaussian Splatting cannot handle natively. Consistently outperforms existing methods on PSNR, SSIM, and LPIPS metrics on novel view synthesis tasks [S1].
*   **Limitations:** Relies on the approximation of non-differentiable functions, which may introduce artifacts if the approximation is poor. Requires careful tuning of $\epsilon$ and learning rates for the control points [S1].
*   **When to Use:** When rendering scenes with sharp discontinuities, edges, or boundaries that standard smooth Gaussian Splatting fails to represent accurately.

## Background
Originated in **DisC-GS: Discontinuity-aware Gaussian Splatting** [S1]. The method addresses the limitation of standard Gaussian Splatting, which assumes differentiable rendering, by incorporating a discontinuous indicator function $g(\cdot)$ into the $\alpha$-blending process. Since $g(\cdot)$ is piecewise constant, its gradients are zero, preventing the optimization of the control points $c_{\text{curve}}$ that define the discontinuity boundaries. The Bézier-boundary gradient approximation strategy was proposed to bridge this gap by approximating the gradient of the Bézier curve function with respect to its control points, allowing the network to learn the geometry of discontinuities [S1].

## Sources
[S1] DisC-GS: Discontinuity-aware Gaussian Splatting, https://arxiv.org/abs/2405.15196v2, fetched 2026-10-26
