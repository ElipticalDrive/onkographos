# Eigenvalue wheel

Also called: Cubic characteristic polynomial wheel, Nickalls wheel

## Overview
The eigenvalue wheel is a geometric visualization method that maps the three real eigenvalues of a symmetric tensor to the projections of equally spaced spokes on a circle. It solves the problem of visualizing tensor shape by encoding the mean, variance, and skewness of the eigenvalues into the circle's center, radius, and spoke orientation, respectively. Unlike the Mohr circle diagram which uses nested circles to convey pairwise differences, or trilinear coordinate systems that map eigenvalues to a single point in a triangular space, the eigenvalue wheel uses continuous geometric parameters of a single circle to display the full shape information [S1, unofficial].

## How it works
**Inputs:** The three eigenvalues of a symmetric tensor, $\lambda_1, \lambda_2, \lambda_3$.
**Outputs:** A geometric representation defined by a center $\mu_1$, a radius $R$, and an orientation angle $\Theta$.

**Data Structures:**
The method computes three scalar invariants from the eigenvalues:
1.  **Mean ($\mu_1$):** The first central moment.
2.  **Variance-related Radius ($R$):** Derived from the second central moment $\mu_2$.
3.  **Skewness-related Angle ($\Theta$):** Derived from the third central moment $\mu_3$ and the standard deviation.

**Formulas and Definitions:**
Let the eigenvalues be sorted such that $\lambda_1 \ge \lambda_2 \ge \lambda_3$.

1.  **Mean ($\mu_1$):**
    $$ \mu_1 = \frac{\lambda_1 + \lambda_2 + \lambda_3}{3} = \frac{J_1}{3} $$
    where $J_1$ is the first principal invariant (trace).

2.  **Second Central Moment ($\mu_2$) and Radius ($R$):**
    $$ \mu_2 = \frac{1}{3} \sum_{i=1}^3 (\lambda_i - \mu_1)^2 $$
    The radius of the wheel is:
    $$ R = 2\sqrt{Q} = \sqrt{2\mu_2} = \sigma\sqrt{2} $$
    where $\sigma = \sqrt{\mu_2}$ is the standard deviation of the eigenvalues, and $Q$ is the second principal invariant of the deviatoric tensor.

3.  **Third Central Moment ($\mu_3$) and Skewness ($\alpha_3$):**
    $$ \mu_3 = \frac{1}{3} \sum_{i=1}^3 (\lambda_i - \mu_1)^3 $$
    $$ \alpha_3 = \frac{\mu_3}{\sigma^3} $$

4.  **Spoke Orientation ($\Theta$):**
    $$ \Theta = \frac{1}{3} \cos^{-1}(\sqrt{2} \alpha_3) $$
    Note: The argument to $\cos^{-1}$ must be clamped to $[-1, 1]$ to handle numerical precision errors.

**Numbered Steps:**
1.  Compute the mean $\mu_1$ from the input eigenvalues.
2.  Compute the standard deviation $\sigma$ and the skewness $\alpha_3$.
3.  Calculate the radius $R = \sigma\sqrt{2}$.
4.  Calculate the orientation angle $\Theta = \frac{1}{3} \cos^{-1}(\sqrt{2} \alpha_3)$.
5.  Define three spokes at angles $\Theta$, $\Theta - 2\pi/3$, and $\Theta + 2\pi/3$ relative to the horizontal axis.
6.  The eigenvalues are the projections of the spoke ends onto the horizontal axis passing through $\mu_1$. The projections are given by:
    $$ \lambda_1 = \mu_1 + R \cos(\Theta) $$
    $$ \lambda_2 = \mu_1 + R \cos(\Theta - 2\pi/3) $$
    $$ \lambda_3 = \mu_1 + R \cos(\Theta + 2\pi/3) $$
    These formulas inherently produce sorted eigenvalues $\lambda_1 \ge \lambda_2 \ge \lambda_3$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def eigenvalue_wheel(lambda1, lambda2, lambda3):
    """
    Computes the geometric parameters of the eigenvalue wheel.
    
    Args:
        lambda1, lambda2, lambda3: The three eigenvalues of a symmetric tensor.
        
    Returns:
        mu1: Center of the wheel (mean).
        R: Radius of the wheel.
        Theta: Orientation angle of the spokes.
    """
    # Step 1: Compute Mean
    mu1 = (lambda1 + lambda2 + lambda3) / 3.0
    
    # Step 2: Compute Standard Deviation (sigma)
    # mu2 is the variance
    mu2 = ((lambda1 - mu1)**2 + (lambda2 - mu1)**2 + (lambda3 - mu1)**2) / 3.0
    sigma = math.sqrt(mu2)
    
    # Edge case: If sigma is 0, all eigenvalues are equal.
    # The wheel collapses to a point. Theta is undefined but irrelevant.
    if sigma == 0:
        return mu1, 0.0, 0.0
        
    # Step 3: Compute Skewness (alpha3)
    mu3 = ((lambda1 - mu1)**3 + (lambda2 - mu1)**3 + (lambda3 - mu1)**3) / 3.0
    alpha3 = mu3 / (sigma**3)
    
    # Step 4: Compute Orientation Angle (Theta)
    # Clamp argument to [-1, 1] to handle floating point errors
    arg = math.sqrt(2.0) * alpha3
    arg = max(-1.0, min(1.0, arg))
    Theta = (1.0 / 3.0) * math.acos(arg)
    
    # Radius
    R = sigma * math.sqrt(2.0)
    
    return mu1, R, Theta

def reconstruct_eigenvalues(mu1, R, Theta):
    """
    Reconstructs the sorted eigenvalues from the wheel parameters.
    """
    l1 = mu1 + R * math.cos(Theta)
    l2 = mu1 + R * math.cos(Theta - 2.0 * math.pi / 3.0)
    l3 = mu1 + R * math.cos(Theta + 2.0 * math.pi / 3.0)
    return l1, l2, l3
```

**Parameters:**
-   $\mu_1$: Center location on the real axis.
-   $R$: Radius, proportional to the standard deviation.
-   $\Theta$: Angle in radians, typically in $[0, \pi/3]$ due to symmetry and sorting.

**Edge Cases and Pitfalls:**
-   **Complex Plane:** The circle does not lie on the complex plane. The symmetry of the tensor $\mathbf{D}$ ensures three real eigenvalues, avoiding the case of one real and two complex-conjugate eigenvalues [S1, unofficial].
-   **Degenerate Input:** If all eigenvalues are equal ($\sigma = 0$), the radius is 0 and the wheel collapses to a single point. The angle $\Theta$ is undefined but does not affect the result.
-   **Numerical Precision:** The argument to $\cos^{-1}$ must be clamped to $[-1, 1]$ to prevent domain errors due to floating-point inaccuracies when $\alpha_3$ is close to $\pm 1/\sqrt{2}$.

## Subtopics

### [Shape Space Scaling](8892-shape-space-scaling.md)
Normalizes the shape space coordinates.

### [Mohr Diagram](8899-mohr-diagram.md)
Uses nested circles to convey eigenvalues and pairwise differences.

## Operating characteristics
-   **Time Complexity:** $O(1)$ for computing the invariants and geometric parameters from three eigenvalues.
-   **Memory Complexity:** $O(1)$, storing only three scalar values ($\mu_1, R, \Theta$).
-   **Strengths:** Provides an intuitive geometric interpretation of tensor shape using continuous parameters. It clearly separates the mean (position), variance (size), and skewness (orientation) of the eigenvalues.
-   **Limitations:** It is a visualization aid rather than a direct data structure. It requires the eigenvalues to be computed first.
-   **When to use:** When visualizing the shape of symmetric tensors (e.g., in Diffusion Tensor Imaging) and needing to distinguish between linear, planar, and spherical shapes based on the spoke orientation.
-   **When not to use:** When pairwise differences between eigenvalues are the primary focus (use Mohr diagram instead) or when a compact point-based representation is needed (use trilinear coordinates).

## Background
The eigenvalue wheel was introduced by Kindlmann in his 2004 PhD thesis "Visualization and Analysis of Diffusion Tensor Fields" [S1, unofficial]. It is based on a geometric analogy for solving cubic characteristic polynomials, originally presented by Nickalls in the context of general cubic equations [S1, unofficial]. Kindlmann adapted this construction specifically for visualizing the shape of diffusion tensors, noting its superficial similarity to the Mohr circle diagram from continuum mechanics but highlighting its distinct use of a single circle with oriented spokes.

## Sources
[S1] Visualization and Analysis of Diffusion Tensor Fields Abstract, http://people.cs.uchicago.edu/~glk/pubs/pdf/Kindlmann-VisualizationAndAnalysisOfDiffusionTensorFields-PhD-2004.pdf, fetched 2026-10-30 (unofficial)
