# Propagation of Affine Forms (PAF)

Also called: Probabilistic Affine Forms (PAF) Propagation, Uncertainty Propagation, Uncertainty Propagation through INRs

## Overview
Propagation of Affine Forms (PAF) is a technique for estimating the output value distribution of Implicit Neural Representations (INRs) over a spatial input region by tracking probabilistic affine forms through network layers [S1]. It solves the problem of overly conservative bounds in traditional Range Analysis (RA) by generating "soft" probabilistic bounds, which improves the efficiency of hierarchical iso-surface extraction [S1]. Unlike its sibling [Range Analysis with Uniform Assumption](18690-range-analysis-with-uniform-assumption.md), PAF performs probability estimation at each layer's activation function approximation, allowing for the calculation of exact estimation error variance and resulting in a more accurate output distribution estimate [S1].

## How it works
**Inputs:**
- An Implicit Neural Representation (INR) $f_\theta$ with $k$ layers.
- A 3D bounding box defined by lower corner $\mathbf{x}_l$ and upper corner $\mathbf{x}_u$.
- A confidence interval threshold hyperparameter $t$.
- An iso-surface value $c$.

**Outputs:**
- A set $A$ of active cells (sub-regions) containing the iso-surface.
- Or an empty set $\emptyset$ if the region is determined to not contain the surface.

**Data Structures:**
- **Probabilistic Affine Form (PAF):** A random variable $\hat{X}$ represented as an affine combination of unit random variables:
  $$ \hat{X} = x_0 + \sum_{i=1}^{N} x_i Z_i $$
  where $x_0$ is the mean, $x_i$ are coefficients, and $Z_i$ are unit random variables with $E(Z_i)=0$ and $Var(Z_i)=1$ [S1].
- **Input Region Representation:** A 3D bounding box is represented as a vector of PAFs $\hat{\mathbf{V}}_0$. For a box with center $\mathbf{x}_c$ and diagonal extents, the PAF vector is constructed using a diagonal matrix $\mathbf{V}$ of orthogonal axes coefficients and a vector $\boldsymbol{\epsilon}$ of unit random variables:
  $$ \hat{\mathbf{V}}_0 = \mathbf{x}_c + \mathbf{V}\boldsymbol{\epsilon} $$
  The coefficients in $\mathbf{V}$ normalize the uniform random variables to have unit standard deviation. For a uniform distribution on $[-x_1, x_1]$, the standard deviation is $x_1/\sqrt{3}$, so the coefficient is $\sqrt{3}x_1$ [S1].

**Numbered Steps:**

1.  **Initialize Input PAF:**
    Calculate the center $\mathbf{x}_c = (\mathbf{x}_l + \mathbf{x}_u)/2$ [S1].
    Construct the diagonal matrix $\mathbf{V} = \text{Diag}(\frac{1}{\sqrt{3}}(\mathbf{x}_u - \mathbf{x}_c))$ [S1].
    Define the input PAF vector $\hat{\mathbf{X}} = \mathbf{x}_c + \mathbf{V}\boldsymbol{\epsilon}$, where $\boldsymbol{\epsilon}$ is a vector of independent unit random variables [S1].

2.  **Propagate Through INR:**
    Pass $\hat{\mathbf{X}}$ through the INR layers sequentially. For each layer, apply linear transformations and activation functions using the rules below to obtain the output PAF $\hat{\mathbf{Y}}$ [S1].

3.  **Linear Function Rule:**
    For a linear function $f(\hat{X}) = \alpha \hat{X} + \beta$, the PAF is transformed exactly:
    $$ f(\hat{X}) = \alpha x_0 + \beta + \sum_{i=1}^{N} \alpha x_i Z_i $$
    The mean becomes $\alpha x_0 + \beta$ and the coefficients are scaled by $\alpha$ [S1].

4.  **Nonlinear Activation Function Rule:**
    For a nonlinear function $f(\hat{X})$, approximate it with a linear function $\bar{f}(\hat{X}) = \alpha x_0 + \beta + \sum_{i=1}^{N} \alpha x_i Z_i$ and introduce a new random variable $\gamma Z_{N+1}$ to denote the approximation error [S1].
    The application is converted to:
    $$ f(\hat{X}) = \bar{f}(\hat{X}) + \gamma Z_{N+1} = \alpha x_0 + \beta + \sum_{i=1}^{N} \alpha x_i Z_i + \gamma Z_{N+1} $$
    where $\gamma$ is the standard deviation of the approximation error, ensuring $Var(Z_{N+1})=1$ [S1].
    
    **Calculating $\gamma$:**
    The linear approximation is determined using the least squares method to minimize the Mean Squared Error (MSE) between the nonlinear function and the linear approximation over the input Gaussian distribution [S1].
    The MSE is decomposed into error variance and squared bias: $MSE = \gamma^2 + [E(Z_{N+1})]^2$ [S1].
    The least squares method ensures the solution is unbiased ($E(Z_{N+1})=0$), so minimizing MSE is equivalent to minimizing $\gamma^2$ [S1].
    The variance $\gamma^2$ is calculated by minimizing the integral of the squared error weighted by the input PDF $g(x|\mu, \sigma^2)$ [S1]. Analytical solutions exist for common activations like ReLU, ELU, and Sine [S1].

5.  **Extract Distribution Parameters:**
    After propagation through $k$ layers, the output PAF $\hat{\mathbf{Y}}$ is approximated as a Gaussian distribution using the Lindeberg Central Limit Theorem (CLT) [S1].
    The mean $\mu$ is the zeroth-order coefficient $y_0$ of $\hat{\mathbf{Y}}$ [S1].
    The variance $\sigma^2$ is the sum of the squares of the higher-order coefficients $y_i$:
    $$ \sigma^2 = \sum_{i=1}^{M} y_i^2 $$
    where $M$ is the number of random variables in the output PAF [S1].

6.  **Decision and Recursion:**
    Check if the region contains the iso-surface using the soft bound:
    If $\mu + t\sigma < c$ or $\mu - t\sigma > c$, return $\emptyset$ [S1].
    Else, if the maximum depth is reached, return the current box $\{(\mathbf{x}_l, \mathbf{x}_u)\}$ [S1].
    Else, split the box along the widest dimension into $\mathbf{x}'_l, \mathbf{x}'_u$ and $\mathbf{x}''_l, \mathbf{x}''_u$ [S1].
    Recursively call the procedure for both sub-boxes and return the union of the results [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def propagate_affine_forms(f_theta, x_l, x_u, c, t, max_depth, current_depth=0):
    """
    Propagates Affine Forms through an INR to determine if a bounding box contains an iso-surface.
    
    Args:
        f_theta: The INR function.
        x_l: Lower corner of the bounding box (array).
        x_u: Upper corner of the bounding box (array).
        c: Iso-surface value.
        t: Confidence interval threshold.
        max_depth: Maximum recursion depth.
        current_depth: Current recursion depth.
        
    Returns:
        A set of active cells (bounding boxes) or an empty set.
    """
    # Step 1: Initialize Input PAF
    # Calculate center
    x_c = (x_l + x_u) / 2.0
    
    # Calculate diagonal matrix V for orthogonal axes
    # V = Diag(1/sqrt(3) * (x_u - x_c))
    # We represent the PAF vector implicitly by storing mean and covariance structure
    # Mean
    mean = x_c
    
    # Covariance structure: The PAF is mean + V * epsilon
    # V is diagonal with entries 1/sqrt(3) * (x_u_i - x_c_i)
    # The variance of each component i is (V_ii)^2 * Var(epsilon_i) = (V_ii)^2 * 1
    # So variance_i = (1/3) * (x_u_i - x_c_i)^2
    # We don't need to explicitly store V if we just track mean and variance for the CLT step,
    # but for propagation through layers, we need the affine form structure.
    # However, the source describes propagating the PAF vector. 
    # For implementation, we assume f_theta can accept PAF inputs or we approximate.
    # The source implies sequential application of linear and activation functions.
    
    # Let's represent the PAF as a mean vector and a list of coefficient vectors for each random variable.
    # Initial state: 3 independent random variables for x, y, z.
    # X_hat = x_c + V * epsilon
    # epsilon = [Z1, Z2, Z3]
    # V is 3x3 diagonal.
    # So X_hat_i = x_c_i + V_ii * Z_i
    
    # Initialize PAF state: mean vector and coefficient matrix (rows are variables, cols are Z_i)
    # Actually, let's use the notation from the source: X_hat = x0 + sum(x_i Z_i)
    # For 3D box, we have 3 Z's.
    # x0 = x_c
    # Coefficients for Z1 (x-axis): [V_11, 0, 0] ? No, the PAF is a vector.
    # Let's treat the input as a vector of PAFs.
    
    # PAF Vector V_hat_0
    # Component 0 (x): mean x_c[0], coeffs [V_00, 0, 0] corresponding to Z1, Z2, Z3
    # Component 1 (y): mean x_c[1], coeffs [0, V_11, 0]
    # Component 2 (z): mean x_c[2], coeffs [0, 0, V_22]
    
    V_diag = (x_u - x_c) / np.sqrt(3.0)
    
    # Represent PAF as (mean, coeffs_matrix)
    # coeffs_matrix shape: (num_components, num_random_vars)
    num_components = 3
    num_random_vars = 3
    coeffs_matrix = np.zeros((num_components, num_random_vars))
    
    for i in range(3):
        coeffs_matrix[i, i] = V_diag[i]
        
    paf_mean = x_c.copy()
    paf_coeffs = coeffs_matrix
    
    # Step 2: Propagate Through INR
    # The source says "sequentially applies linear and activation functions on PAFs".
    # We assume f_theta is composed of layers. We need to simulate the forward pass.
    # Since we don't have the internal structure of f_theta in the source, we assume a helper function
    # that applies one layer to a PAF.
    
    paf_mean, paf_coeffs = apply_inr_layers(f_theta, paf_mean, paf_coeffs)
    
    # Step 5: Extract Distribution Parameters using CLT
    # Mean is paf_mean (scalar output for iso-surface check)
    # Variance is sum of squares of coefficients
    # The output of INR is a scalar SDF value.
    # So paf_mean is scalar, paf_coeffs is a vector of size num_random_vars
    
    mu = paf_mean
    sigma_sq = np.sum(paf_coeffs**2)
    sigma = np.sqrt(sigma_sq)
    
    # Step 6: Decision
    if mu + t * sigma < c or mu - t * sigma > c:
        return set()
    
    if current_depth >= max_depth:
        return {(x_l, x_u)}
    
    # Split along widest dimension
    dims = x_u - x_l
    widest_idx = np.argmax(dims)
    mid = (x_l[widest_idx] + x_u[widest_idx]) / 2.0
    
    x_l_left = x_l.copy()
    x_u_left = x_u.copy()
    x_l_left[widest_idx] = mid
    
    x_l_right = x_l.copy()
    x_u_right = x_u.copy()
    x_u_right[widest_idx] = mid
    
    A_left = propagate_affine_forms(f_theta, x_l_left, x_u_left, c, t, max_depth, current_depth + 1)
    A_right = propagate_affine_forms(f_theta, x_l_right, x_u_right, c, t, max_depth, current_depth + 1)
    
    return A_left.union(A_right)

def apply_inr_layers(f_theta, paf_mean, paf_coeffs):
    """
    Simulates the forward pass of the INR on PAFs.
    This is a placeholder for the specific layer-by-layer propagation logic described in the source.
    The source mentions applying linear and activation functions sequentially.
    """
    # Note: The source does not provide the exact internal structure of f_theta or the exact 
    # implementation of apply_inr_layers. It states that the process is summarized in Fig 2(b).
    # We must implement the linear and nonlinear rules defined in the text.
    
    # Assume f_theta has a known structure of linear weights W, biases b, and activation functions.
    # We iterate through layers.
    
    current_mean = paf_mean
    current_coeffs = paf_coeffs
    
    # This part requires knowledge of f_theta's architecture which is not in the source.
    # However, the source provides the rules for Linear and Nonlinear functions.
    
    # For a linear layer: y = Wx + b
    # new_mean = W * current_mean + b
    # new_coeffs = W * current_coeffs (matrix multiplication)
    
    # For a nonlinear activation: f(x)
    # Approximate with linear f_bar(x) = alpha * x + beta
    # new_mean = alpha * current_mean + beta
    # new_coeffs = alpha * current_coeffs
    # Add new random variable Z_new with coefficient gamma (std dev of error)
    # new_coeffs = concatenate(current_coeffs, [gamma])
    
    # Since we cannot implement the specific INR without its definition, we return the input
    # as a placeholder, noting that the actual implementation would iterate through layers.
    # [G] In a real implementation, this function would unpack the INR's weights and activations
    # and apply the PAF rules defined in the text.
    
    return current_mean, current_coeffs

def linear_paf_rule(paf_mean, paf_coeffs, alpha, beta):
    """
    Applies a linear function f(X) = alpha * X + beta to a PAF.
    """
    new_mean = alpha * paf_mean + beta
    new_coeffs = alpha * paf_coeffs
    return new_mean, new_coeffs

def nonlinear_paf_rule(paf_mean, paf_coeffs, activation_func, input_mean, input_std):
    """
    Applies a nonlinear activation function to a PAF using linear approximation.
    """
    # Calculate alpha and beta using least squares to minimize MSE
    # alpha, beta = calculate_least_squares(activation_func, input_mean, input_std)
    # [G] The source states analytical solutions exist for ReLU, ELU, Sine.
    # The exact formulas for alpha and beta are in the appendix of [S1].
    # Here we assume a helper function exists.
    alpha, beta = get_activation_linear_params(activation_func, input_mean, input_std)
    
    # Calculate error variance gamma^2
    # gamma_sq = calculate_error_variance(activation_func, paf_mean, paf_coeffs)
    # [G] This requires integrating the squared error over the Gaussian distribution.
    gamma_sq = get_error_variance(activation_func, paf_mean, paf_coeffs)
    gamma = np.sqrt(gamma_sq)
    
    new_mean, new_coeffs = linear_paf_rule(paf_mean, paf_coeffs, alpha, beta)
    
    # Add new random variable Z_{N+1} with coefficient gamma
    new_coeffs = np.append(new_coeffs, gamma)
    
    return new_mean, new_coeffs
```

**Parameters:**
- $t$: Confidence interval threshold. Typical values are not given in the sources.
- $Z_i$: Unit random variables with $E(Z_i)=0$ and $Var(Z_i)=1$ [S1].
- Assumption: $Z_i$ are independent [S1].

**Edge Cases and Pitfalls:**
- The approximation of the PAF distribution with a Gaussian distribution via CLT is an approximation and may introduce inaccuracy, though the source states this inaccuracy is not introduced by the linear function itself [S1].
- The method relies on the analytical solution for the linear approximation of activation functions. If an activation function does not have an analytical least-squares solution, this method may not be directly applicable without numerical approximation [S1].

## Subtopics

### [Linear Approximation of Activation Functions for Probabilistic Affine Forms](18691-linear-approximation-of-activation-functions-for-probabilist.md)
This method approximates nonlinear activation functions with linear functions to enable the analytical propagation of probabilistic affine forms (PAFs) through Implicit Neural Representations (INRs). It solves the least-squares optimization problem analytically to determine the slope and intercept of the linear approximation, allowing uncertainty to be propagated efficiently without the conservative bounds associated with range analysis. It is distinct from range analysis by maintaining efficiency and accuracy for periodic functions like Sine, where range analysis suffers from high approximation error.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** The source states that range analysis bounds are "too conservative, resulting in poor efficiency when used for hierarchical iso-surface extraction" [S1]. PAF is expected to have a "more accurate output distribution estimate" compared to RA-UA [S1].
- **Strengths:** More accurate output distribution estimate than RA-UA; efficient for periodic functions like Sine; generates "soft" bounds for adaptive subdivision.
- **Limitations:** Relies on CLT approximation; requires analytical solutions for activation function approximations.
- **When to use:** When hierarchical iso-surface extraction on INRs requires efficient pruning of empty regions and high accuracy for periodic activations.
- **When not to use:** When exact bounds are required instead of probabilistic bounds, or when activation functions do not have analytical least-squares approximations.

## Background
- **Origin:** Proposed in "Improving Efficiency of Iso-Surface Extraction on Implicit Neural Representations Using Uncertainty Propagation" [S1].
- **History:** Probabilistic affine forms were previously proposed by Bouissou et al. [25, 26] for general distributions and dependencies, but this work adapts them for simpler explanation and specific application to INRs [S1].
- **Usage:** Used for hierarchical iso-surface extraction from Implicit Neural Representations (INRs) [S1].

## Sources
[S1] Improving Efficiency of Iso-Surface Extraction on Implicit Neural Representations Using Uncertainty Propagation, https://doi.org/10.1109/tvcg.2024.3365089, fetched 2026-10-26
