# Reparameterized Stochastic Gradient Descent

Also called: Reparameterized SGD, Reparameterized Stochastic Gradient Descent for Parameter Optimization

## Overview
Reparameterized Stochastic Gradient Descent is an optimization technique for inverse rendering that finds altered scattering parameters by minimizing an error function between a rendered image and a target image [S1]. It addresses the problem where standard SGD gets trapped in local minima with small gradients under low-frequency lighting, which leads to solutions that generalize poorly to high-frequency lighting [S1]. It sets itself apart from its parent [SGD with Positive t-Distribution](5809-sgd-with-positive-t-distribution.md) by explicitly reparameterizing the search space to regularize the error surface, rather than relying solely on the statistical properties of the gradient noise distribution.

## How it works
**Inputs and Outputs:**
*   **Input:** Target image $I_0$, initial guess for scattering parameters $(g, \sigma_s)$, error function $d(g, \sigma_s)$ [S1].
*   **Output:** Optimized scattering parameters $(g, \sigma_s)$ that minimize $d$ [S1].

**Data Structures:**
*   Parameters: $g$ (anisotropy factor), $\sigma_s$ (scattering coefficient).
*   Reparameterized variable: $\tilde{\sigma}_s$ (reduced scattering coefficient).

**Formulas:**
The reparameterization replaces the horizontal axis with the reduced scattering coefficient [S1]:
$$ \tilde{\sigma}_s := (1 - g)\sigma_s $$
This ensures both axes ($g$ and $\tilde{\sigma}_s$) have the same units and regularizes the error surface [S1].

**Steps:**
1.  **Reparameterize:** Transform the search space from $(g, \sigma_s)$ to $(g, \tilde{\sigma}_s)$ using the formula above [S1].
2.  **First 1D SGD (Class Selection):** Fix $\sigma_s$ and perform a one-dimensional SGD search to find $\tilde{\sigma}_s$ that minimizes the error $d$ [S1]. This step selects an equivalence class based on the order-1 similarity relation [S1].
3.  **Second 1D SGD (Intra-class Optimization):** Keep $\tilde{\sigma}_s$ fixed and perform another one-dimensional SGD search to find the best $\sigma_s$ within the selected class [S1].

**Pitfalls:**
*   **Low-frequency lighting:** Creates large regions where the error function $d(g, \sigma_s)$ is close to zero with very small gradients. These gradients can be dominated by Monte Carlo or measurement noise, causing SGD to get trapped in local minima that generalize poorly to high-frequency lighting [S1].
*   **Initial Guess:** Starting far from the solution may lead to convergence in a suboptimal equivalence class if the reparameterization does not fully resolve the ambiguity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_sgd(target_image, initial_g, initial_sigma_s, error_function, learning_rate, num_iterations):
    """
    Performs reparameterized SGD to find optimal scattering parameters.
    
    Args:
        target_image: The input image I0.
        initial_g: Initial guess for anisotropy factor g.
        initial_sigma_s: Initial guess for scattering coefficient sigma_s.
        error_function: Function d(g, sigma_s) computing error between rendered and target image.
        learning_rate: Step size for SGD.
        num_iterations: Number of iterations for each 1D search.
        
    Returns:
        g_opt: Optimized anisotropy factor.
        sigma_s_opt: Optimized scattering coefficient.
    """
    # Step 1: Reparameterization is implicit in the variable definitions below.
    # We work in the space (g, sigma_tilde_s) where sigma_tilde_s = (1 - g) * sigma_s.
    
    g = initial_g
    sigma_s = initial_sigma_s
    
    # Calculate initial reduced scattering coefficient
    sigma_tilde_s = (1 - g) * sigma_s
    
    # Step 2: First 1D SGD - Fix sigma_s, optimize sigma_tilde_s
    # Note: In the source, sigma_s is fixed during this phase.
    # However, sigma_tilde_s depends on g. The source states "fix sigma_s and look for sigma_tilde_s".
    # This implies we treat sigma_s as constant for the gradient update of sigma_tilde_s,
    # but since sigma_tilde_s is the variable, we update it directly.
    # To maintain consistency with the definition sigma_tilde_s = (1-g)*sigma_s,
    # if we update sigma_tilde_s, we must later recover sigma_s or g.
    # The source says: "First, we fix sigma_s and look for sigma_tilde_s... Then, we search for the best parameter point
    # within this class by keeping sigma_tilde_s fixed and performing another 1D search to find the best sigma_s."
    # This implies a sequential update where the variables are decoupled during their respective phases.
    
    # Phase 1: Optimize sigma_tilde_s with sigma_s fixed
    current_sigma_tilde_s = sigma_tilde_s
    current_sigma_s = sigma_s # Fixed
    
    for i in range(num_iterations):
        # Compute gradient of error with respect to sigma_tilde_s
        # d/d(sigma_tilde_s) d(g, sigma_s)
        # Note: The error function is defined on physical parameters.
        # We need to map current variables back to physical space to compute error.
        # In this phase, g is not explicitly fixed in the text, but sigma_s is.
        # The text says "fix sigma_s". It does not explicitly say fix g.
        # However, sigma_tilde_s = (1-g)*sigma_s. If sigma_s is fixed, changing sigma_tilde_s
        # effectively changes g.
        # Let's assume we update sigma_tilde_s directly.
        
        grad_sigma_tilde = compute_gradient(current_sigma_tilde_s, current_sigma_s, target_image, error_function)
        current_sigma_tilde_s -= learning_rate * grad_sigma_tilde
        
    # After Phase 1, we have an optimized sigma_tilde_s and fixed sigma_s.
    # We need to determine the corresponding g to proceed to Phase 2?
    # The text says: "Then, we search for the best parameter point within this class by keeping sigma_tilde_s fixed
    # and performing another 1D search to find the best sigma_s."
    # This implies we keep the NEW sigma_tilde_s fixed, and optimize sigma_s.
    # But sigma_tilde_s = (1-g)*sigma_s. If we fix sigma_tilde_s and optimize sigma_s, g must change.
    
    g_opt = None
    sigma_s_opt = None
    
    # Phase 2: Optimize sigma_s with sigma_tilde_s fixed
    # We use the sigma_tilde_s from the end of Phase 1.
    final_sigma_tilde_s = current_sigma_tilde_s
    current_sigma_s = sigma_s # Start from initial or previous? Source implies sequential.
    
    # We need an initial g for the second phase?
    # The source says "keeping sigma_tilde_s fixed".
    # We can derive g from sigma_tilde_s and current sigma_s: g = 1 - sigma_tilde_s / sigma_s.
    
    for i in range(num_iterations):
        # Compute gradient of error with respect to sigma_s
        # d/d(sigma_s) d(g, sigma_s)
        # Note: g is dependent on sigma_s here because sigma_tilde_s is fixed.
        # g = 1 - sigma_tilde_s / sigma_s
        
        grad_sigma_s = compute_gradient(final_sigma_tilde_s, current_sigma_s, target_image, error_function)
        current_sigma_s -= learning_rate * grad_sigma_s
        
    # Final parameters
    sigma_s_opt = current_sigma_s
    # Recover g from the fixed sigma_tilde_s and optimized sigma_s
    # g = 1 - sigma_tilde_s / sigma_s
    if sigma_s_opt != 0:
        g_opt = 1 - (final_sigma_tilde_s / sigma_s_opt)
    else:
        g_opt = 0 # Guard against division by zero
        
    return g_opt, sigma_s_opt

def compute_gradient(sigma_tilde_s, sigma_s, target_image, error_function):
    """
    Computes the gradient of the error function with respect to the variable.
    This is a placeholder for the actual Monte Carlo gradient estimation.
    """
    # Calculate g from sigma_tilde_s and sigma_s
    if sigma_s == 0:
        g = 0
    else:
        g = 1 - (sigma_tilde_s / sigma_s)
        
    # Compute error
    error = error_function(g, sigma_s, target_image)
    
    # Compute gradient numerically or via adjoint method
    # This is a placeholder for the actual gradient computation
    # In practice, this would involve rendering and backpropagation
    return 0.0 # Placeholder
```

## Subtopics

### [Similarity-Based Inverse Rendering](10272-similarity-based-inverse-rendering.md)
A proof-of-concept technique that warps the non-linear parameter space of translucent materials to resolve ambiguities caused by similarity equivalence classes. Unlike Reparameterized SGD, which uses a specific algebraic reparameterization ($\tilde{\sigma}_s$), this method leverages similarity theory to identify and warp ambiguous regions in the scattering parameter space.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Under low-frequency lighting, standard SGD gets trapped in local minima, while Reparameterized SGD finds solutions that generalize better to high-frequency lighting [S1].
*   **Strengths:** Regularizes the error surface, making it easier for SGD to escape local minima associated with low-frequency lighting [S1].
*   **Limitations:** Still susceptible to noise in gradient estimation due to Monte Carlo simulations [S1].
*   **When to use:** When optimizing scattering parameters for translucent materials under low-frequency lighting conditions where standard SGD fails to generalize [S1].
*   **When not to use:** When high-frequency lighting is the primary concern and standard SGD performs adequately, or when computational resources are extremely limited and the overhead of reparameterization is prohibitive (though the overhead is minimal).

## Background
The method originates from the work on high-order similarity relations in radiative transfer [S1]. It was developed to address the ill-conditioned nature of inverse rendering, where different parameters in the same equivalence class produce very similar appearances, creating ambiguities that stall standard optimization [S1]. The technique is applied in the context of forward rendering for optically dense and forward-scattering materials, which are common in the real world but challenging to render [S1].

## Sources
[S1] High-order similarity relations in radiative transfer, https://doi.org/10.1145/2601097.2601104, fetched 2026-10-26
