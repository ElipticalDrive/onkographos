# Bézier Curve Signed Distance Function

Also called: Distance to Generalized Cylinder (Bézier Curve), Offset surface distance, Skeletal model distance

## Overview
Computes the exact distance from a query point to the global offset surface of a cubic Bézier curve, effectively modeling a generalized cylinder with a fixed radius around the curve [S1]. It differs from standard primitive SDFs by requiring the solution of a higher-degree polynomial equation to find the closest point on the curve, rather than relying on closed-form geometric projections [S1].

## How it works
**Inputs:**
- Query point $x \in \mathbb{R}^3$.
- Cubic Bézier curve defined by control points $P_0, P_1, P_2, P_3 \in \mathbb{R}^3$.
- Radius $R$ of the generalized cylinder.

**Outputs:**
- Distance $d_{curve}$ from $x$ to the curve, and signed distance $d(x) = d_{curve} - R$ to the offset surface.

**Data Structures:**
- Control points $P_i$ stored as 3D vectors [G].
- Coefficients for the degree-5 polynomial derived from the orthogonality condition [G].

**Steps:**
1. **Relative Coordinate System:** To simplify the derivation, assume the query point $x$ is at the origin. Transform the control points by subtracting $x$ from each $P_i$, yielding $Q_i = P_i - x$ [S1].
2. **Parametric Definition:** Define the cubic Bézier curve $p(u)$ for $u \in [0, 1]$ using the Bernstein basis polynomials:
   $$ p(u) = (1-u)^3 Q_0 + 3(1-u)^2 u Q_1 + 3(1-u) u^2 Q_2 + u^3 Q_3 $$
   The derivative $p'(u)$ is:
   $$ p'(u) = 3(1-u)^2 (Q_1 - Q_0) + 6(1-u) u (Q_2 - Q_1) + 3u^2 (Q_3 - Q_2) $$
3. **Orthogonality Condition:** The closest point $p(u)$ on the curve to the origin satisfies the condition that the vector from the curve to the origin is orthogonal to the tangent vector:
   $$ (0 - p(u)) \cdot p'(u) = 0 \implies p(u) \cdot p'(u) = 0 $$
   This equation determines the parameter $u$ of the closest point [S1].
4. **Polynomial Conversion:** The equation $p(u) \cdot p'(u) = 0$ results in a polynomial of degree 5 in $u$. The source notes this can be converted into a degree-5 1D Bézier curve representation [S1].
5. **Root Finding:** Solve the degree-5 polynomial for real roots $u \in [0, 1]$ using efficient numerical techniques [S1].
6. **Candidate Selection:** Evaluate the distance $|p(u)|$ for all valid roots $u \in [0, 1]$. Also evaluate the distances to the endpoints $|Q_0|$ (at $u=0$) and $|Q_3|$ (at $u=1$). The minimum of these distances is the distance to the curve, denoted $d_{curve}$ [S1].
7. **Signed Distance Calculation:** The signed distance to the generalized cylinder surface is:
   $$ d(x) = d_{curve} - R $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bezier_sdf(x, P0, P1, P2, P3, R):
    # Step 1: Transform to local coordinates where query point is origin
    Q0 = P0 - x
    Q1 = P1 - x
    Q2 = P2 - x
    Q3 = P3 - x

    # Step 2: Convert to power basis for easier coefficient calculation
    # p(u) = V0 + V1*u + V2*u^2 + V3*u^3
    V0 = Q0
    V1 = 3 * (Q1 - Q0)
    V2 = 3 * (Q2 - 2*Q1 + Q0)
    V3 = Q3 - 3*Q2 + 3*Q1 - Q0

    # Step 3 & 4: Compute coefficients of D(u) = p(u) . p'(u) = a0 + a1*u + ... + a5*u^5
    # p'(u) = V1 + 2*V2*u + 3*V3*u^2
    a0 = dot(V0, V1)
    a1 = dot(V1, V1) + 2 * dot(V0, V2)
    a2 = 3 * dot(V1, V2) + 3 * dot(V0, V3)
    a3 = 4 * dot(V1, V3) + 2 * dot(V2, V2)
    a4 = 5 * dot(V2, V3)
    a5 = 3 * dot(V3, V3)

    # Step 5: Find roots in [0, 1]
    roots = find_roots_in_interval([a0, a1, a2, a3, a4, a5], 0.0, 1.0)

    # Step 6: Candidate selection
    min_dist_sq = float('inf')
    # Check endpoints
    min_dist_sq = min(min_dist_sq, dot(Q0, Q0))
    min_dist_sq = min(min_dist_sq, dot(Q3, Q3))
    # Check interior roots
    for u in roots:
        pu = V0 + u * (V1 + u * (V2 + u * V3))
        dist_sq = dot(pu, pu)
        if dist_sq < min_dist_sq:
            min_dist_sq = dist_sq

    d_curve = sqrt(min_dist_sq)
    return d_curve - R

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def sqrt(x):
    return x ** 0.5

def find_roots_in_interval(coeffs, a, b):
    # Standard numerical root finding for polynomials [G]
    roots = []
    def poly(u):
        return sum(c * (u ** i) for i, c in enumerate(coeffs))
    # Bisection on 1000 subintervals
    steps = 1000
    for i in range(steps):
        u1 = a + (b - a) * i / steps
        u2 = a + (b - a) * (i + 1) / steps
        y1 = poly(u1)
        y2 = poly(u2)
        if y1 == 0:
            roots.append(u1)
        elif y1 * y2 < 0:
            # Bisection to refine root
            lo, hi = u1, u2
            for _ in range(50):
                mid = (lo + hi) / 2
                ymid = poly(mid)
                if ymid == 0:
                    lo = hi = mid
                    break
                if y1 * ymid < 0:
                    hi = mid
                    y2 = ymid
                else:
                    lo = mid
                    y1 = ymid
            roots.append((lo + hi) / 2)
    return roots
```

**Parameters:**
- $R$: Radius of the tube. Typical values depend on scene scale.

**Edge Cases:**
- If the polynomial has no roots in $[0, 1]$, the closest point is one of the endpoints [S1].
- Precision issues may arise if the curve is very short or nearly straight, causing the polynomial to be ill-conditioned [G].

## Subtopics

### [Footvector Representation](8501-footvector-representation.md)
The Footvector Representation is a geometric generalization of Signed Distance Functions (SDFs) for curves and surfaces, defined by the vector pointing from a query point to its closest point (footpoint) on the shape. Unlike standard SDFs which only provide scalar distance, this representation retains directional information, enabling exact offset surface computation for Constructive Solid Geometry (CSG) operations via fast iterative methods. It is chosen when precise geometric invariants and exact offsetting are required, though it poses implementation challenges for GPU shaders due to higher-order function mappings.

### [Spline Tube to Cylinder Mapping](10061-spline-tube-to-cylinder-mapping.md)
Spline Tube to Cylinder Mapping is a technique for rendering procedural density fields inside deformed spline tubes by mapping points from the deformed space back to a canonical reference cylinder. It solves the problem of applying isotropic procedural noise or density functions to arbitrarily bent or twisted volumes without storing or transforming the density field itself. Unlike [Footvector Representation](8501-footvector-representation.md), which typically relies on local geometric properties for distance queries, this method explicitly constructs a coordinate transformation based on the Frenet frame to query a pre-defined cylinder density field.

## Operating characteristics
- **Time Complexity:** Solving a degree-5 polynomial is more expensive than the closed-form solutions for primitives like spheres or cylinders. The cost is dominated by root finding, which is typically $O(1)$ but with a higher constant factor [G].
- **Memory Complexity:** $O(1)$, storing only control points and coefficients [G].
- **Strengths:** Provides exact distance to smooth, curved tubes without tessellation.
- **Limitations:** Requires robust polynomial root finding. Performance is lower than linear primitives.
- **When to use:** When smooth, organic shapes defined by curves are needed and exact SDFs are preferred over approximations.

## Background
The method is based on the concept of generalized cylinders, specifically the fixed-radius global offset surface of a Bézier curve [S1]. The approach of converting the orthogonality condition into a degree-5 Bézier curve and solving it efficiently was described by Schneider (1990) and Rockwood et al. (1989) [S1].

## Sources
[S1] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
