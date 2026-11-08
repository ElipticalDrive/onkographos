# Bézier Curve Signed Distance Function

Also called: Distance to Generalized Cylinder (Bézier Curve), Offset surface distance, Skeletal model distance

## Overview
Computes the exact signed distance from a query point to the global offset surface of a cubic Bézier curve, effectively modeling a generalized cylinder with a fixed radius around the curve [S1]. It differs from standard primitive SDFs by requiring the solution of a higher-degree polynomial equation to find the closest point on the curve, rather than relying on closed-form geometric projections [S1].

## How it works
**Inputs:**
- Query point $x \in \mathbb{R}^3$.
- Cubic Bézier curve defined by control points $P_0, P_1, P_2, P_3 \in \mathbb{R}^3$.
- Radius $R$ of the generalized cylinder.

**Outputs:**
- Signed distance $d(x)$ from $x$ to the surface of the tube.

**Data Structures:**
- Control points $P_i$ stored as 3D vectors.
- Coefficients for the degree-5 polynomial derived from the orthogonality condition.

**Steps:**
1.  **Relative Coordinate System:** To simplify the derivation, assume the query point $x$ is at the origin. If $x$ is not the origin, transform the control points $P_i$ by subtracting $x$ from each $P_i$. Let the transformed control points be $Q_i = P_i - x$. The distance calculation is then performed relative to the origin [S1].
2.  **Parametric Definition:** Define the cubic Bézier curve $p(u)$ for $u \in [0, 1]$ using the Bernstein basis polynomials:
    $$ p(u) = (1-u)^3 Q_0 + 3(1-u)^2 u Q_1 + 3(1-u) u^2 Q_2 + u^3 Q_3 $$
    The derivative $p'(u)$ is:
    $$ p'(u) = 3(1-u)^2 (Q_1 - Q_0) + 6(1-u) u (Q_2 - Q_1) + 3u^2 (Q_3 - Q_2) $$
3.  **Orthogonality Condition:** The closest point $p(u)$ on the curve to the origin satisfies the condition that the vector from the curve to the origin is orthogonal to the tangent vector:
    $$ (0 - p(u)) \cdot p'(u) = 0 \implies p(u) \cdot p'(u) = 0 $$
    This equation determines the parameter $u$ of the closest point [S1].
4.  **Polynomial Conversion:** The equation $p(u) \cdot p'(u) = 0$ results in a polynomial of degree 5 in $u$. This degree-5 polynomial can be converted into a 1D Bézier curve representation (degree-5 Bernstein form) [S1].
5.  **Root Finding:** Solve the degree-5 polynomial for roots $u \in [0, 1]$. Efficient techniques for solving such polynomials, such as those described by Rockwood et al., are used to find all real roots within the valid parameter range [S1].
6.  **Candidate Selection:** Evaluate the distance $|p(u)|$ for all valid roots $u \in [0, 1]$. Also evaluate the distances to the endpoints $|Q_0|$ (at $u=0$) and $|Q_3|$ (at $u=1$). The minimum of these distances is the distance to the curve, denoted $d_{curve}$ [S1].
7.  **Signed Distance Calculation:** The signed distance to the generalized cylinder surface is:
    $$ d(x) = d_{curve} - R $$
    If $d(x) < 0$, the point is inside the tube; if $d(x) > 0$, it is outside [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bezier_sdf(x, P0, P1, P2, P3, R):
    # Step 1: Transform to local coordinates where query point is origin
    Q0 = P0 - x
    Q1 = P1 - x
    Q2 = P2 - x
    Q3 = P3 - x

    # Step 2 & 3: Define p(u) and p'(u) implicitly via the dot product polynomial
    # p(u) . p'(u) = 0
    # This expands to a degree 5 polynomial in u.
    # Let the polynomial be coeff[0] + coeff[1]*u + ... + coeff[5]*u^5 = 0
    
    # Calculate coefficients for the degree-5 polynomial derived from (p(u) . p'(u))
    # p(u) = sum(Bernstein_3,i(u) * Qi)
    # p'(u) = sum(Bernstein_3,i'(u) * Qi)
    # The dot product is a scalar polynomial.
    
    # Helper to get Bernstein basis and derivative coefficients for cubic
    # Bernstein_3,0 = (1-u)^3, Bernstein_3,1 = 3u(1-u)^2, etc.
    # Derivatives are scaled by 3.
    
    # We need the coefficients of the polynomial f(u) = p(u) . p'(u)
    # Let's compute the vector coefficients for p(u) and p'(u) in Bernstein form
    # p(u) = C0 * (1-u)^3 + C1 * 3u(1-u)^2 + C2 * 3u^2(1-u) + C3 * u^3
    # where C0=Q0, C1=Q1, C2=Q2, C3=Q3
    
    # p'(u) = 3*(C1-C0)*(1-u)^2 + 6*(C2-C1)*u(1-u) + 3*(C3-C2)*u^2
    
    # To find roots of p(u) . p'(u) = 0, we expand this dot product into a standard polynomial a0 + a1*u + ... + a5*u^5
    # Or convert directly to degree-5 Bernstein form.
    
    # For implementation, converting to standard polynomial coefficients is often easier for root finding.
    
    # Vector coefficients for p(u) in power basis (u^0 to u^3)
    # p(u) = Q0*(1-3u+3u^2-u^3) + Q1*(3u-6u^2+3u^3) + Q2*(3u^2-3u^3) + Q3*(u^3)
    # p(u) = Q0 + u*(-3Q0 + 3Q1) + u^2*(3Q0 - 6Q1 + 3Q2) + u^3*(-Q0 + 3Q1 - 3Q2 + Q3)
    
    V0 = Q0
    V1 = -3*Q0 + 3*Q1
    V2 = 3*Q0 - 6*Q1 + 3*Q2
    V3 = -Q0 + 3*Q1 - 3*Q2 + Q3
    
    # p'(u) = V1 + 2*V2*u + 3*V3*u^2
    # Note: The derivative of the power basis form is consistent.
    
    # Dot product D(u) = p(u) . p'(u)
    # D(u) = (V0 + V1*u + V2*u^2 + V3*u^3) . (V1 + 2*V2*u + 3*V3*u^2)
    # Expand to get scalar coefficients a0..a5
    
    a0 = dot(V0, V1)
    a1 = dot(V0, 2*V2) + dot(V1, V1)
    a2 = dot(V0, 3*V3) + dot(V1, 2*V2) + dot(V2, V1)
    a3 = dot(V1, 3*V3) + dot(V2, 2*V2)
    a4 = dot(V2, 3*V3)
    a5 = dot(V3, 3*V3)
    
    # Solve a0 + a1*u + a2*u^2 + a3*u^3 + a4*u^4 + a5*u^5 = 0 for u in [0, 1]
    roots = solve_polynomial_roots([a0, a1, a2, a3, a4, a5], min_val=0.0, max_val=1.0)
    
    min_dist_sq = infinity
    
    # Check endpoints
    dist_sq_0 = dot(Q0, Q0)
    dist_sq_3 = dot(Q3, Q3)
    min_dist_sq = min(dist_sq_0, dist_sq_3)
    
    # Check interior roots
    for u in roots:
        # Evaluate p(u)
        # Using Horner's method for stability
        # p(u) = V0 + u*(V1 + u*(V2 + u*V3))
        pu = V0 + u * (V1 + u * (V2 + u * V3))
        dist_sq = dot(pu, pu)
        if dist_sq < min_dist_sq:
            min_dist_sq = dist_sq
            
    d_curve = sqrt(min_dist_sq)
    return d_curve - R
```

**Parameters:**
- $R$: Radius of the tube. Typical values depend on scene scale.

**Edge Cases:**
- If the polynomial has no roots in $[0, 1]$, the closest point is one of the endpoints.
- Precision issues may arise if the curve is very short or nearly straight, causing the polynomial to be ill-conditioned.

## Subtopics

### [Footvector Representation](8501-footvector-representation.md)
The Footvector Representation is a geometric generalization of Signed Distance Functions (SDFs) for curves and surfaces, defined by the vector pointing from a query point to its closest point (footpoint) on the shape. Unlike standard SDFs which only provide scalar distance, this representation retains directional information, enabling exact offset surface computation for Constructive Solid Geometry (CSG) operations via fast iterative methods. It is chosen when precise geometric invariants and exact offsetting are required, though it poses implementation challenges for GPU shaders due to higher-order function mappings.

### [Spline Tube to Cylinder Mapping](10061-spline-tube-to-cylinder-mapping.md)
Spline Tube to Cylinder Mapping is a technique for rendering procedural density fields inside deformed spline tubes by mapping points from the deformed space back to a canonical reference cylinder. It solves the problem of applying isotropic procedural noise or density functions to arbitrarily bent or twisted volumes without storing or transforming the density field itself. Unlike [Footvector Representation](8501-footvector-representation.md), which typically relies on local geometric properties for distance queries, this method explicitly constructs a coordinate transformation based on the Frenet frame to query a pre-defined cylinder density field.

## Operating characteristics
- **Time Complexity:** Solving a degree-5 polynomial is more expensive than the closed-form solutions for primitives like spheres or cylinders. The cost is dominated by root finding, which is typically $O(1)$ but with a higher constant factor.
- **Memory Complexity:** $O(1)$, storing only control points and coefficients.
- **Strengths:** Provides exact distance to smooth, curved tubes without tessellation.
- **Limitations:** Requires robust polynomial root finding. Performance is lower than linear primitives.
- **When to use:** When smooth, organic shapes defined by curves are needed and exact SDFs are preferred over approximations.

## Background
The method is based on the concept of generalized cylinders, specifically the fixed-radius global offset surface of a Bézier curve [S1]. The approach of converting the orthogonality condition into a degree-5 Bézier curve and solving it efficiently was described by Schneider (1990) and Rockwood et al. (1989) [S1]. This technique is part of the broader field of ray tracing implicit surfaces, as detailed in the source paper [S1].

## Sources
[S1] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
