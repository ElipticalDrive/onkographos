# BFGS Solver with Moreau Envelope

Also called: BFGS with Moreau Envelope, Huber Reconstruction

## Overview
This method solves inverse problems, specifically tomographic reconstruction, by minimizing a composite objective function consisting of a data fidelity term and a regularizer. It sets itself apart by using the BFGS quasi-Newton algorithm to handle the non-smooth L1 regularization via its Moreau envelope, providing a differentiable approximation that allows second-order optimization steps. This contrasts with first-order Gradient Descent solvers and avoids the explicit projection steps required by constrained material solvers.

## How it works
The method minimizes an objective function $F(x)$ defined as the sum of a squared L2 data discrepancy and a smoothed L1 regularizer applied to the image gradient.

**Inputs:**
- Forward operator $A$ (e.g., Ray Transform).
- Noisy measurement data $y$.
- Image space $X$ (e.g., uniform discrete space).
- Gradient operator $\nabla$.
- Hyperparameters: smoothing parameter $\sigma$, regularization weight $\lambda$, maximum iterations $maxiter$, and memory size $num\_store$.

**Outputs:**
- Reconstructed image $x$.

**Data Structures:**
- $x$: The current estimate of the image, stored as a tensor/array in $X$.
- $H^{-1}_0$: Initial inverse Hessian estimate, implemented as a scaling operator.
- $s, y$ vectors: Stored for Limited Memory BFGS (L-BFGS) history.

**Steps:**
1. **Initialize Operator Norm:** Compute the operator norm of $A$ using the power method to scale the initial Hessian estimate [S1, unofficial].
   $$ opnorm = \|A\| $$
2. **Initialize Inverse Hessian:** Set the initial inverse Hessian estimate $H^{-1}_0$ as a scaled identity operator [S1, unofficial]:
   $$ H^{-1}_0 = \frac{1}{2 \|A\|^2} I $$
3. **Define Data Term:** Construct the squared L2 norm of the residual [S1, unofficial]:
   $$ D(x) = \|Ax - y\|_2^2 $$
4. **Define Regularizer:**
   - Compute the gradient of the image: $g = \nabla x$.
   - Apply the Group L1 norm to the gradient: $\|g\|_{1,2}$.
   - Smooth this non-smooth norm using the Moreau envelope with parameter $\sigma$ [S1, unofficial]:
     $$ R(x) = \text{MoreauEnvelope}_{\sigma}(\|\nabla x\|_{1,2}) $$
5. **Define Objective:** Combine terms [S1, unofficial]:
   $$ F(x) = D(x) + \lambda R(x) $$
6. **Solve:** Run the BFGS solver (specifically L-BFGS if $num\_store$ is limited) starting from an initial guess (often the FBP pseudoinverse result) [S1, unofficial].
   - Use the precomputed $H^{-1}_0$ as the initial Hessian approximation.
   - Limit iterations to $maxiter$.
   - Store $num\_store$ vectors for the limited memory update.

**Parameters:**
- $\sigma = 0.03$: Smoothing parameter for the Moreau envelope [S1, unofficial].
- $\lambda = 0.3$: Regularization weight [S1, unofficial].
- $maxiter = 200$: Maximum number of iterations [S1, unofficial].
- $num\_store = 10$: Number of vectors stored for L-BFGS [S1, unofficial].

**Pitfalls:**
- The initial reconstruction (pseudoinverse) must use the 'Hann' filter to match the reference implementation's assumptions [S1, unofficial].
- Precision issues may arise if the operator norm is not computed accurately, affecting the scaling of the Hessian estimate.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import odl

def bfgs_moreau_envelope_solver(A, y, space, sigma=0.03, lam=0.3, maxiter=200, num_store=10):
    """
    Solve inverse problem using BFGS with Moreau Envelope regularization.
    
    Args:
        A: Forward operator (e.g., odl.tomo.RayTransform)
        y: Measurement data
        space: Image space (e.g., odl.uniform_discr)
        sigma: Smoothing parameter for Moreau envelope
        lam: Regularization weight
        maxiter: Maximum iterations
        num_store: Memory size for L-BFGS
        
    Returns:
        x: Reconstructed image
    """
    # 1. Compute operator norm
    opnorm = odl.power_method_opnorm(A)
    
    # 2. Initialize inverse Hessian estimate
    # H_inv_0 = 0.5 / opnorm^2 * I
    hessinv_estimate = odl.ScalingOperator(space, 0.5 / opnorm**2)
    
    # 3. Define data term: ||Ax - y||_2^2
    l2_norm = odl.solvers.L2NormSquared(A.range)
    data_discrepancy = l2_norm * (A - y)
    
    # 4. Define regularizer using Moreau Envelope
    gradient = odl.Gradient(space)
    l1_norm = odl.solvers.GroupL1Norm(gradient.range)
    smoothed_l1 = odl.solvers.MoreauEnvelope(l1_norm, sigma=sigma)
    regularizer = smoothed_l1 * gradient
    
    # 5. Define objective function
    obj_fun = data_discrepancy + lam * regularizer
    
    # 6. Initial guess (e.g., FBP pseudoinverse)
    # Note: Must use 'Hann' filter as per reference [S1, unofficial]
    pseudoinverse = odl.tomo.fbp_op(A, filter_type='Hann')
    x = pseudoinverse(y)
    
    # 7. Solve using BFGS
    # Using odl.solvers.bfgs_method which supports L-BFGS via num_store
    callback = None # Optional callback for logging
    
    x = odl.solvers.bfgs_method(
        obj_fun, 
        x, 
        maxiter=maxiter, 
        num_store=num_store, 
        hessinv_estimate=hessinv_estimate, 
        callback=callback
    )
    
    return x
```

## Subtopics

### [Monte Carlo Path Contribution Estimation](14539-monte-carlo-path-contribution-estimation.md)
A Monte Carlo estimator to calculate the radiance contribution of a stationary path found by the BFGS solver, based on the path formulation of the Radiative Radiative Transfer Equation (RRTE).

## Operating characteristics
- **Time Complexity:** Each iteration involves computing the gradient of the data term and the smoothed regularizer. The L-BFGS update is $O(m \cdot d)$ where $m$ is $num\_store$ and $d$ is the dimension of the image. The dominant cost is usually the forward and adjoint operator evaluations ($A$ and $A^*$).
- **Memory Complexity:** $O(d)$ for the image and gradients, plus $O(m \cdot d)$ for the L-BFGS history vectors.
- **Performance:** The Moreau envelope makes the L1 norm differentiable, allowing BFGS to converge faster than first-order methods for smooth approximations, but the smoothing parameter $\sigma$ must be chosen carefully to balance accuracy and differentiability.
- **Strengths:** Handles non-smooth regularization (L1) effectively within a second-order framework. The Hann filter initialization provides a stable starting point.
- **Limitations:** Requires computation of the operator norm, which can be expensive for large operators. The smoothing parameter $\sigma$ is a hyperparameter that affects convergence and solution quality.
- **When to use:** Suitable for tomographic reconstruction and other inverse problems where L1 regularization on gradients is desired and second-order convergence is beneficial.
- **When not to use:** If the operator norm is too expensive to compute or if the problem size is extremely large, first-order methods might be preferred.

## Background
The method is based on the reference implementation from the `learned_gradient_tomography` repository [S1, unofficial]. It combines classical tomographic reconstruction techniques (FBP with Hann filter) with modern optimization methods (BFGS with Moreau envelope) to solve inverse problems. The use of the Moreau envelope to smooth the L1 norm is a standard technique in convex optimization to enable the use of gradient-based methods.

## Sources
[S1] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
