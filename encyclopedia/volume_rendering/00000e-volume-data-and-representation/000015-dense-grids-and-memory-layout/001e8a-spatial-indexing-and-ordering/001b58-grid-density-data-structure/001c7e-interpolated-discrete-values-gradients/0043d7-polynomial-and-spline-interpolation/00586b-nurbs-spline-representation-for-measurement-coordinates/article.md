# NURBS Spline Representation for Measurement Coordinates

Also called: NURBS Spline Representation, NURBS surface for measurement coordinates, NURBS surface optimization for measurement coordinates, Rational B-spline optimization for view angles

## Overview
This method represents uncertain measurement coordinates (such as view angles, tilt angles, and detector locations) as a NURBS surface whose control points and weights are optimized alongside the volume reconstruction [S1, unofficial]. It differs from implicit neural representations by using fixed-degree spline basis functions with optimized control points, which allows for the explicit enforcement of physical consistency conditions (such as Radon transform symmetries) that neural networks approximate implicitly [S1, unofficial]. It is typically paired with a U-Net reconstruction network $G_\mu(\cdot)$ that maps the NURBS-interpolated coordinates to image space [S1, unofficial].

## How it works
**Inputs:**
*   Measurement coordinate vector $\mathbf{m} = [\theta', t']^T$ (e.g., view angle and detector location) [S1, unofficial].
*   Knot vectors for each dimension (non-learnable) [S1, unofficial].
*   Control point vectors $Q_{u,d_\Theta}$ and $Q_{v,d_T}$ (learnable) [S1, unofficial].
*   Weights $w_{u,v}$ (learnable) [S1, unofficial].
*   Degrees $d_\Theta, d_T \in \mathbb{Z}^+$ for each dimension [S1, unofficial].
*   Segment counts $J, K$ per dimension [S1, unofficial].

**Outputs:**
*   Optimized measurement coordinates or the basis function values used to map raw measurements to the reconstruction space [S1, unofficial].

**Data Structures:**
*   **Knot Vector:** For a dimension with $J$ segments and degree $d$, the knot vector has $J + d + 1$ elements.
    *   Elements $k_u$ for $0 \le u < d + 1$ are set to 0.
    *   Elements $k_u$ for $d + 1 \le u \le J$ are uniformly spaced between 0 and 1.
    *   Elements $k_u$ for $J < u \le J + d$ are set to 1 [S1, unofficial].
*   **Control Points:** A grid of vectors $Q_{u,v}$ of size $J \times K$ [S1, unofficial].
*   **Weights:** A grid of scalars $w_{u,v}$ of size $J \times K$ [S1, unofficial].

**Steps:**
1.  **Construct Knot Vectors:** Create the knot vector for the view angle dimension with $J + d_\Theta + 1$ elements and the detector location dimension with $J + d_T + 1$ elements (or $K + d_T + 1$ if segment counts differ) using the ascending order rule described above [S1, unofficial].
2.  **Compute 1D Basis Functions:** Calculate the B-spline basis functions $Q_{u,d_\Theta}(\theta')$ and $Q_{v,d_T}(t')$ using the Cox-de Boor recursion [S1, unofficial].
3.  **Form Tensor Product:** Compute the rational basis functions $b_{j,k}$ by taking the tensor product of the 1D basis functions weighted by $w_{u,v}$ and normalized by the sum of weighted basis functions [S1, unofficial].
4.  **Evaluate Surface:** The optimized coordinate is derived from the weighted sum of control points using the rational basis functions [S1, unofficial].

**Formulas:**
The rational basis function is defined as [S1, unofficial]:
$$ b_{j,k}([\theta', t']^T) = \frac{\sum_{u=0}^{J-1} \sum_{v=0}^{K-1} w_{u,v} Q_{u,d_\Theta}(\theta') Q_{v,d_T}(t')}{\sum_{u=0}^{J-1} \sum_{v=0}^{K-1} Q_{u,d_\Theta}(\theta') Q_{v,d_T}(t')} $$
where $w_{u,v}$ are learnable weights, $Q$ are B-spline basis functions, $d_\Theta, d_T$ are degrees, and $J, K$ are segment counts per dimension [S1, unofficial].

The B-spline basis function $Q_{u,d}(\cdot)$ is obtained using the Cox-de Boor recursion [S1, unofficial]:
$$ Q_{u,0}(k) = \begin{cases} 1 & \text{if } k_u \le k < k_{u+1} \\ 0 & \text{otherwise} \end{cases} $$
$$ Q_{u,d}(k) = \frac{k - k_u}{k_{u+d} - k_u} Q_{u,d-1}(k) + \frac{k_{u+d+1} - k}{k_{u+d+1} - k_{u+1}} Q_{u+1,d-1}(k) $$
where $k_u$ is the $u$-th element of the knot vector [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_knot_vector(num_segments, degree):
    """
    Constructs a knot vector for NURBS.
    num_segments: J or K (number of intervals)
    degree: d (degree of the spline)
    Returns: list of length num_segments + degree + 1
    """
    total_length = num_segments + degree + 1
    knot_vector = [0.0] * total_length
    
    # First degree + 1 elements are 0
    # Middle elements (degree + 1 to num_segments) are uniformly spaced
    # Last degree elements are 1
    
    # Implementation of uniform spacing between index 'degree' and 'num_segments'
    # Note: The source specifies indices 0 <= u < d+1 are 0, d+1 <= u <= J are uniform, J < u <= J+d are 1.
    # This implies the uniform part starts at index d+1 (0-based) which is the (d+2)-th element?
    # Let's map strictly to source:
    # u < d+1 -> 0
    # d+1 <= u <= J -> uniform
    # u > J -> 1
    
    # Re-reading source: "0 <= u < d_theta + 1" -> indices 0 to d_theta are 0.
    # "d_theta + 1 <= u <= J" -> indices d_theta+1 to J are uniform.
    # "J < u <= J + d_theta" -> indices J+1 to J+d_theta are 1.
    
    for u in range(total_length):
        if u < degree + 1:
            knot_vector[u] = 0.0
        elif u > num_segments:
            knot_vector[u] = 1.0
        else:
            # Uniformly spaced between 0 and 1
            # The range of indices is from degree+1 to num_segments inclusive.
            # Number of points = num_segments - (degree + 1) + 1 = num_segments - degree
            # We need to map these to [0, 1].
            # Standard clamped uniform knot vector usually has:
            # 0...0, 0, 1/(N-d), ..., 1, 1...1
            # Let's assume standard uniform spacing for the internal knots.
            # Index u maps to fraction (u - degree) / (num_segments - degree) ?
            # Let's check bounds.
            # If u = degree + 1, val = 1 / (num_segments - degree)
            # If u = num_segments, val = (num_segments - degree) / (num_segments - degree) = 1.
            # This matches the boundary conditions.
            knot_vector[u] = (u - degree) / (num_segments - degree)
            
    return knot_vector

def cox_de_boor(u, k, degree, knot_vector):
    """
    Computes the B-spline basis function Q_{u,d}(k) using Cox-de Boor recursion.
    u: index of the basis function
    k: the parameter value (coordinate)
    degree: d
    knot_vector: list of knots
    """
    if degree == 0:
        # Q_{u,0}(k) = 1 if k_u <= k < k_{u+1}, else 0
        k_u = knot_vector[u]
        k_u_plus_1 = knot_vector[u + 1]
        if k_u <= k < k_u_plus_1:
            return 1.0
        else:
            return 0.0
    
    # Recursive step
    # Term 1: (k - k_u) / (k_{u+d} - k_u) * Q_{u,d-1}(k)
    denom1 = knot_vector[u + degree] - knot_vector[u]
    term1 = 0.0
    if denom1 != 0:
        term1 = ((k - knot_vector[u]) / denom1) * cox_de_boor(u, k, degree - 1, knot_vector)
    else:
        # If denominator is zero, term is zero [S1, unofficial]
        term1 = 0.0
        
    # Term 2: (k_{u+d+1} - k) / (k_{u+d+1} - k_{u+1}) * Q_{u+1,d-1}(k)
    denom2 = knot_vector[u + degree + 1] - knot_vector[u + 1]
    term2 = 0.0
    if denom2 != 0:
        term2 = ((knot_vector[u + degree + 1] - k) / denom2) * cox_de_boor(u + 1, k, degree - 1, knot_vector)
    else:
        # If denominator is zero, term is zero [S1, unofficial]
        term2 = 0.0
        
    return term1 + term2

def nurbs_surface_basis(theta_prime, t_prime, J, K, d_theta, d_t, weights, knots_theta, knots_t):
    """
    Computes the rational basis function value for a given coordinate.
    """
    numerator = 0.0
    denominator = 0.0
    
    for u in range(J):
        for v in range(K):
            # Compute 1D basis functions
            q_u_theta = cox_de_boor(u, theta_prime, d_theta, knots_theta)
            q_v_t = cox_de_boor(v, t_prime, d_t, knots_t)
            
            w_uv = weights[u][v]
            
            weighted_basis = w_uv * q_u_theta * q_v_t
            numerator += weighted_basis
            denominator += q_u_theta * q_v_t
            
    if denominator == 0:
        return 0.0 # Guard against division by zero, though unlikely with valid knots
    return numerator / denominator
```

**Parameters:**
*   $d_\Theta, d_T$: Degrees of the spline for view angle and detector location. Typical values are not explicitly given, but Appendix C mentions degree zero NURBS uses piecewise constant basis functions [S1, unofficial].
*   $J, K$: Segment counts per dimension. In the example, there are 120 view angles, implying $J=120$ or similar scaling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Denominators in Cox-de Boor:** If the denominator in any term of the Cox-de Boor recursion is zero, that term is taken to be zero [S1, unofficial].
*   **Boundary Conditions:** The knot vector construction ensures clamped boundaries (0 and 1) [S1, unofficial].

## Subtopics

### [NURBS-based Uncalibrated Imaging Reconstruction](16412-nurbs-based-uncalibrated-imaging-reconstruction.md)
NURBS-based Uncalibrated Imaging Reconstruction is a method for reconstructing volumetric data (specifically CT) from projections with uncertain or uncalibrated measurement coordinates (view angles, tilt angles, detector positions) by modeling these coordinates as a NURBS surface. It differs from implicit neural representations by using fixed-degree spline basis functions with optimized control points, which allows for the explicit enforcement of physical consistency conditions (such as Radon transform symmetries) that neural networks approximate implicitly. It is typically paired with a U-Net reconstruction network $G_\mu(\cdot)$ that maps the NURBS-interpolated coordinates to image space.

## Operating characteristics
*   **Time Complexity:** Evaluation of the NURBS surface requires computing $J \times K$ basis functions. Each basis function evaluation via Cox-de Boor recursion is $O(d^2)$ or $O(d)$ depending on implementation. Thus, evaluation is $O(J \cdot K \cdot d^2)$.
*   **Memory Complexity:** Storage of control points ($J \times K \times \text{dim}$) and weights ($J \times K$).
*   **Performance:** The method reduces angle error as optimization progresses, demonstrating that it learns more accurate tilt angles than assumed ones [S1, unofficial]. In experiments with 120 view angles, it showed competitive SNR compared to pseudoinverse, baseline, and implicit neural methods under measurement noise [S1, unofficial].
*   **Strengths:** Explicit enforcement of physical consistency; differentiable; fixed-degree basis functions provide stability compared to neural networks.
*   **Limitations:** Requires optimization of control points and weights; less flexible than neural networks for highly irregular errors.

## Background
The method originates from "Differentiable Uncalibrated Imaging" (2022) [S1, unofficial]. It builds on the principles of NURBS (Non-Uniform Rational B-Splines) and extends the PyTorch source code from the NURBS-Diff module [S1, unofficial]. It is used in the context of CT reconstruction with uncalibrated measurement coordinates.

## Sources
[S1] Differentiable Uncalibrated Imaging, https://doi.org/10.48550/arXiv.2211.10525, fetched 2026-10-26 (unofficial)
