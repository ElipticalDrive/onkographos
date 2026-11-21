# Bézier Curve Signed Distance Function

Also called: Distance to Generalized Cylinder (Bézier Curve), Offset surface distance, Skeletal model distance

## Overview
Computes the exact signed distance from a query point to the global offset surface of a cubic Bézier space curve, effectively modeling a generalized cylinder with a fixed radius. It differs from standard primitive SDFs by solving for the closest point on a parametric curve via root finding of the orthogonality condition, rather than using closed-form geometric projections.

## How it works
**Inputs:**
- Query point $x \in \mathbb{R}^3$.
- Cubic Bézier curve defined by control points $P_0, P_1, P_2, P_3 \in \mathbb{R}^3$.
- Radius $R$ of the generalized cylinder.

**Outputs:**
- Signed distance $d(x)$ from $x$ to the offset surface.

**Data Structures:**
- Control points $P_i$ stored as 3D vectors.
- Coefficients for the degree-5 polynomial derived from the orthogonality condition.

**Steps:**
1. **Relative Coordinates:** Translate the problem so the query point is at the origin. Let $q = -x$. The distance from $x$ to the curve is the same as the distance from the origin to the curve shifted by $-x$. [S1]
2. **Orthogonality Condition:** The closest point $p(u)$ on the curve to the origin satisfies the condition that the vector from the curve to the origin is orthogonal to the curve's tangent:
   $$ (p(u) - 0) \cdot p'(u) = 0 \implies p(u) \cdot p'(u) = 0 $$
   where $p(u)$ is the cubic Bézier curve and $p'(u)$ is its derivative. [S1]
3. **Polynomial Conversion:** The equation $p(u) \cdot p'(u) = 0$ for a cubic Bézier curve results in a polynomial of degree 5 in $u$. This polynomial can be represented as a degree-5 1D Bézier curve. [S1]
4. **Root Finding:** Solve for the roots of this degree-5 polynomial within the valid parameter range $u \in [0, 1]$. Efficient techniques for solving 1D Bézier curves are used. [S1]
5. **Distance Calculation:** For each valid root $u^*$, calculate the distance $r = \|p(u^*)\|$. The minimum such $r$ is the distance to the curve. The signed distance to the offset surface is $d(x) = r - R$. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bezier_sdf(x, P0, P1, P2, P3, R):
    # Step 1: Relative coordinates (query at origin)
    # p(u) is the curve relative to x
    # p(u) = (1-u)^3*P0 + 3*(1-u)^2*u*P1 + 3*(1-u)*u^2*P2 + u^3*P3
    # But since we shifted x to origin, the control points are relative to -x
    # Let Q0 = P0 - x, Q1 = P1 - x, Q2 = P2 - x, Q3 = P3 - x
    Q0 = P0 - x
    Q1 = P1 - x
    Q2 = P2 - x
    Q3 = P3 - x

    # Step 2 & 3: Construct the degree-5 polynomial coefficients for p(u) . p'(u) = 0
    # p(u) = sum(B_i(u) * Qi)
    # p'(u) = sum(B'_i(u) * Qi)
    # The dot product p(u) . p'(u) is a polynomial of degree 5.
    # We need the coefficients of this polynomial in Bernstein basis or power basis.
    # Schneider (1990) describes converting this to a 1D Bézier curve.
    # Let the polynomial be f(u). We find roots of f(u) in [0, 1].
    
    # Implementation detail: Calculate coefficients of f(u) = p(u) . p'(u)
    # p(u) = C0*(1-u)^3 + C1*3*(1-u)^2*u + C2*3*(1-u)*u^2 + C3*u^3
    # p'(u) = 3*(C1-C0)*(1-u)^2 + 6*(C2-2*C1+C0)*(1-u)*u + 3*(C3-3*C2+3*C1-C0)*u^2
    # This is complex to expand manually. A robust way is to evaluate the polynomial
    # at many points and use subdivision, or explicitly compute the 6 coefficients
    # of the degree 5 Bernstein polynomial.
    
    # For the sake of a complete implementation from text, we use a standard root-finding
    # approach for the polynomial derived from the orthogonality condition.
    
    # Coefficients for the polynomial f(u) = p(u) . p'(u)
    # This requires expanding the dot product.
    # Let's define helper functions for Bernstein basis derivatives.
    
    # Alternative: Use Newton-Raphson or subdivision on the function g(u) = p(u) . p'(u)
    # Since it's a degree 5 polynomial, it can have up to 5 roots.
    
    roots = find_roots_degree5_bezier(Q0, Q1, Q2, Q3)
    
    min_dist_sq = infinity
    
    for u in roots:
        if 0 <= u <= 1:
            p_u = bezier_point(Q0, Q1, Q2, Q3, u)
            dist_sq = dot(p_u, p_u)
            if dist_sq < min_dist_sq:
                min_dist_sq = dist_sq
                
    # Also check endpoints if not covered by roots (though roots should cover extrema)
    # The closest point is either at a root of the derivative of distance squared
    # or at the boundary. The orthogonality condition finds the interior extrema.
    # We must also check u=0 and u=1.
    
    p_0 = Q0
    d0_sq = dot(p_0, p_0)
    if d0_sq < min_dist_sq:
        min_dist_sq = d0_sq
        
    p_1 = Q3
    d1_sq = dot(p_1, p_1)
    if d1_sq < min_dist_sq:
        min_dist_sq = d1_sq
        
    dist = sqrt(min_dist_sq)
    return dist - R

def bezier_point(Q0, Q1, Q2, Q3, u):
    u2 = u * u
    u3 = u2 * u
    u1 = 1 - u
    u12 = u1 * u1
    u13 = u12 * u1
    return (u13 * Q0 + 
            3 * u12 * u * Q1 + 
            3 * u1 * u2 * Q2 + 
            u3 * Q3)

def find_roots_degree5_bezier(Q0, Q1, Q2, Q3):
    # This function computes the coefficients of the degree 5 polynomial
    # f(u) = p(u) . p'(u) and finds its roots in [0,1].
    # p(u) = sum_{i=0}^3 B_{i,3}(u) Q_i
    # p'(u) = sum_{i=0}^2 B_{i,2}(u) 3(Q_{i+1} - Q_i)
    # The dot product is a polynomial of degree 5.
    # We can compute the 6 coefficients of the Bernstein basis representation
    # of f(u) and then convert to power basis or use a root finder for Bernstein polynomials.
    
    # Coefficients of p(u) in Bernstein basis: C0=Q0, C1=Q1, C2=Q2, C3=Q3
    # Coefficients of p'(u) in Bernstein basis (degree 2):
    # D0 = 3(Q1 - Q0)
    # D1 = 3(Q2 - Q1)
    # D2 = 3(Q3 - Q2)
    
    D0 = 3 * (Q1 - Q0)
    D1 = 3 * (Q2 - Q1)
    D2 = 3 * (Q3 - Q2)
    
    # f(u) = p(u) . p'(u)
    # We need the Bernstein coefficients of the product of two polynomials.
    # p(u) is degree 3, p'(u) is degree 2. Product is degree 5.
    # Bernstein coefficient k of product of degree n and m polynomials:
    # C_k = sum_{i=0}^k (B_{i,n} * B_{k-i,m} / B_{k,n+m}) * A_i * B_{k-i}
    # where A are coeffs of p, B are coeffs of p'.
    # Here A = [Q0, Q1, Q2, Q3], B = [D0, D1, D2].
    
    # This is vector arithmetic. The result is a set of 6 vectors (coeffs of degree 5).
    # Then we solve for u where sum(B_{i,5}(u) * V_i) = 0.
    # This is a scalar equation if we project, but it's a vector equation?
    # No, f(u) is a scalar (dot product).
    # So we compute the scalar coefficients c_k = V_k . (some direction)?
    # No, the coefficients V_k are vectors. The equation is sum(B_{i,5}(u) * V_i) = 0 vector?
    # No, f(u) = p(u) . p'(u) is a scalar.
    # The Bernstein coefficients of the scalar polynomial f(u) are:
    # c_k = sum_{i=0}^k (binom(k,i) * binom(n+m-k, n-i) / binom(n+m, n)) * (A_i . B_{k-i})
    # Wait, the formula for product of Bernstein polynomials is:
    # If P(u) = sum A_i B_{i,n}(u) and Q(u) = sum B_j B_{j,m}(u)
    # Then R(u) = P(u)Q(u) = sum C_k B_{k,n+m}(u)
    # where C_k = sum_{i=0}^k (binom(n,i) binom(m, k-i) / binom(n+m, k)) A_i B_{k-i}
    
    n = 3
    m = 2
    N = 5
    
    coeffs = []
    for k in range(N + 1):
        c_k = 0.0
        for i in range(k + 1):
            if i <= n and (k - i) <= m:
                binom_n_i = binom(n, i)
                binom_m_ki = binom(m, k - i)
                binom_N_k = binom(N, k)
                factor = (binom_n_i * binom_m_ki) / binom_N_k
                # A_i is Q_i, B_{k-i} is D_{k-i}
                # Dot product
                if i <= 3:
                    A_i = [Q0, Q1, Q2, Q3][i]
                else:
                    continue
                if (k - i) <= 2:
                    B_ki = [D0, D1, D2][k - i]
                else:
                    continue
                c_k += factor * dot(A_i, B_ki)
        coeffs.append(c_k)
        
    # Now find roots of sum(coeffs[k] * B_{k,5}(u)) = 0
    # Use a standard root finder for Bernstein polynomials (e.g., subdivision)
    return bernstein_roots(coeffs, 5)
```

**Parameters:**
- $R$: Radius of the generalized cylinder. Typical values depend on scene scale.

**Edge Cases:**
- If the query point is inside the offset surface, the distance is negative.
- Precision issues may arise if the curve is nearly linear or if the query point is very close to the curve.
- The degree-5 polynomial may have multiple roots; the global minimum must be selected.

## Subtopics

### [Footvector Representation](8501-footvector-representation.md)
The Footvector Representation is a geometric generalization of Signed Distance Functions (SDFs) for curves and surfaces, defined by the vector pointing from a query point to its closest point (footpoint) on the shape [G]. Unlike standard SDFs which only provide scalar distance, this representation retains directional information, enabling exact offset surface computation for Constructive Solid Geometry (CSG) operations via fast iterative methods. It builds on the [Bézier Curve Signed Distance Function](5929-b-zier-curve-signed-distance-function.md) and differs from the [Spline Tube to Cylinder Mapping](10061-spline-tube-to-cylinder-mapping.md) by explicitly tracking the footpoint vector rather than mapping to a cylinder tube. It is chosen when precise geometric invariants and exact offsetting are required, though it poses implementation challenges for GPU shaders due to higher-order function mappings.

### [Spline Tube to Cylinder Mapping](10061-spline-tube-to-cylinder-mapping.md)
Spline Tube to Cylinder Mapping is a technique for mapping points from an arbitrary spline tube to a canonical vertical cylinder, enabling the direct querying of pre-defined isotropic procedural density fields. It solves the problem of inverse parameterization in volumetric rendering by providing a robust Signed Distance Function (SDF) to the curve, which yields the curvilinear abscissa required for texture coordinate calculation.

## Operating characteristics
- **Time Complexity:** Solving a degree-5 polynomial is constant time but involves iterative root-finding or subdivision, which is more expensive than closed-form primitives.
- **Memory Complexity:** Minimal, only storing control points and radius.
- **Strengths:** Exact distance to complex curved tubes; no discretization artifacts.
- **Limitations:** Root finding can be unstable; multiple roots require checking all candidates.
- **When to use:** When rendering smooth, organic tubes or skeletal models with fixed radius.
- **When not to use:** When performance is critical and the curve can be approximated by linear segments or simpler primitives.

## Background
Originated in the context of implicit modeling and ray tracing of generalized cylinders. The method leverages the property that the distance to a curve's offset surface is determined by the closest point on the curve, found via the orthogonality condition. The conversion to a degree-5 Bézier curve allows for efficient root finding using established techniques.

## Sources
[S1] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
