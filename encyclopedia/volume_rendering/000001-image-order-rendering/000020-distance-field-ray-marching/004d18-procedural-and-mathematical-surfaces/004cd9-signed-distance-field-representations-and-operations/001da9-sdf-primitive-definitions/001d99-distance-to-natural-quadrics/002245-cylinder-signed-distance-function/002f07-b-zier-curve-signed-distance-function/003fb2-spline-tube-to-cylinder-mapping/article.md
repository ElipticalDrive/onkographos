# Spline Tube to Cylinder Mapping
Also called: Cylinder Mapping, Spline Tube Deformation, Spline tube deformation mapping

## Overview
Spline Tube to Cylinder Mapping is a technique for mapping points from an arbitrary spline tube (defined by a spline or Bézier curve) to a canonical vertical cylinder, enabling the direct querying of pre-defined isotropic procedural density fields [S1, unofficial]. It solves the problem of inverse parameterization in volumetric rendering by providing a robust Signed Distance Function (SDF) to the curve, which yields the curvilinear abscissa required for texture coordinate calculation. It is distinguished from its siblings, such as Footvector Representation, by its reliance on the Frenet frame and explicit curvilinear abscissa to define the mapping, whereas footvector methods typically rely on local surface normals and implicit distance fields without explicit tube parameterization [S1, unofficial][8501-footvector-representation.md].

## How it works
**Inputs:**
- A point $P$ in 3D space.
- A spline curve $C(t)$ defining the axis of the tube.
- The radius function $r(l)$ of the tube at curvilinear abscissa $l$ (default constant $r=1$ if not specified).
- A reference cylinder defined with radius 1 and height 1, with its lower base centered at the origin.

**Outputs:**
- Mapped coordinates $M$ in the reference cylinder.
- The distance to the tube surface (derived from the SDF, if solvable).

**Data Structures:**
- Spline Curve $C(t)$: Defined by control points. For Bézier segments, this is a set of control points $A, B, C$ (quadratic) or $P_0, P_1, P_2, P_3$ (cubic).
- Frenet Frame at $O$: Consisting of the tangent $\mathbf{T}$, normal $\mathbf{N}$, and binormal $\mathbf{B}$ vectors at the closest point $O$ on the curve.

**Steps:**
1. **Find Closest Point:** Find the point $O$ on the spline curve $C$ which is closest to $P$. Let $t$ be the parameter such that $O = C(t)$. This step is critical; if solvable, it justifies the use of Sphere Marching for bounding box detection [S1, unofficial].
2. **Construct Frenet Frame:** Using the Frenet–Serret formulas, create the Frenet Frame associated with the spline curve at point $O$ [S1, unofficial].
   - Calculate the tangent $\mathbf{T} = C'(t) / \|C'(t)\|$.
   - Calculate the normal $\mathbf{N}$ and binormal $\mathbf{B}$ using the derivatives of $\mathbf{T}$ (specifically $\mathbf{N} = \mathbf{T}' / \|\mathbf{T}'\|$ and $\mathbf{B} = \mathbf{T} \times \mathbf{N}$) [S1, unofficial].
3. **Calculate Angle:** Calculate the angle $\alpha$ between the vector $\vec{OP}$ and the Frenet’s unit normal vector $\mathbf{N}$.
   - Let $\vec{V} = P - O$.
   - $\alpha = \text{atan2}(\vec{V} \cdot \mathbf{B}, \vec{V} \cdot \mathbf{N})$.
4. **Find Curvilinear Abscissa:** Find the curvilinear abscissa $l$ of $O$, defined as the arc length of the spline curve from the beginning of the tube to $O$.
5. **Calculate Mapped Coordinates:** Calculate the mapped coordinates $M$ in the original cylinder.
   - The curvilinear abscissa $l$ represents the height.
   - The angle $\alpha$ deduces the unit direction $\text{dir}$ from the center of the disk at height $l$ toward $M$.
   - The length of $\vec{OP}$ calculates the radial distance.
   - Formula:
     $$ M = (\cos(\alpha) \cdot \|\vec{OP}\| \cdot \frac{1}{r(l)} ; l ; \sin(\alpha) \cdot \|\vec{OP}\| \cdot \frac{1}{r(l)}) $$
   - Note: If stretching is desired, replace $l$ with $t = l / L_{total}$, where $L_{total}$ is the total length of the spline curve [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def spline_to_cylinder_mapping(P, spline_curve, use_stretching=False):
    """
    Maps point P in a spline tube to a canonical cylinder.
    
    Args:
        P: numpy array, point in 3D space.
        spline_curve: Object with methods get_point(t), get_tangent(t), 
                      get_normal(t), get_arc_length(t), get_total_length().
        use_stretching: bool, if True, normalize height by total length.
        
    Returns:
        M: numpy array, mapped coordinates in cylinder [x, y, z].
    """
    # Step 1: Find closest point O on the spline curve to P
    # This requires solving for t that minimizes ||C(t) - P||^2.
    # For a cubic Bézier, this is a polynomial root finding problem.
    # For quadratic, it is also solvable.
    # [G] In practice, this is often done via Newton-Raphson or by 
    # finding roots of the derivative of the distance function.
    t_closest = find_closest_parameter(P, spline_curve)
    O = spline_curve.get_point(t_closest)
    
    # Step 2: Construct Frenet Frame at O
    T = spline_curve.get_tangent(t_closest)
    T = T / np.linalg.norm(T)
    N = spline_curve.get_normal(t_closest)
    B = np.cross(T, N)
    
    # Step 3: Calculate angle alpha
    V = P - O
    alpha = math.atan2(np.dot(V, B), np.dot(V, N))
    
    # Step 4: Find curvilinear abscissa l
    l = spline_curve.get_arc_length(t_closest)
    
    # Step 5: Calculate mapped coordinates M
    # Radius at this point
    r = spline_curve.get_radius(l) # Default 1 if not varying
    
    dist_OP = np.linalg.norm(V)
    
    # Normalized radius for mapping to unit cylinder
    normalized_radius = dist_OP / r
    
    # Height calculation
    if use_stretching:
        L_total = spline_curve.get_total_length()
        height = l / L_total
    else:
        height = l
        
    # Map to cylinder of radius 1, height 1
    # x = cos(alpha) * normalized_radius
    # y = height
    # z = sin(alpha) * normalized_radius
    M = np.array([
        math.cos(alpha) * normalized_radius,
        height,
        math.sin(alpha) * normalized_radius
    ])
    
    return M

def find_closest_parameter(P, spline_curve):
    """
    Finds the parameter t on the spline curve closest to P.
    [G] This is a non-trivial optimization problem. For quadratic Bézier,
    it involves solving a cubic equation. For general splines, iterative
    methods are used.
    """
    # Placeholder for the specific solver implementation
    # For quadratic Bézier with control points A, B, C:
    # C(t) = (1-t)^2 A + 2(1-t)t B + t^2 C
    # We minimize f(t) = ||C(t) - P||^2
    # f'(t) = 0 leads to a cubic equation in t.
    # Roots in [0, 1] are candidates.
    pass
```

**Parameters:**
- Reference Cylinder: Vertical cylinder of radius 1 and height 1, with its lower base centered at the origin [S1, unofficial].
- Stretching: Optional boolean flag. If true, height is normalized by total curve length [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Stretching vs. Bending:** Making a longer spline tube without stretching does not stretch the volume; it looks like a longer cylinder was bent. Stretching is required to simulate expansion [S1, unofficial].
- **Varying Radius:** If the radius varies, the surface normal is no longer perpendicular to the curve axis. This makes the SDF function problematic and can create bugs where boundaries are not detected properly during sphere marching [S1, unofficial].
- **Closest Point Solvability:** If the closest point $O$ is not solvable (e.g., due to numerical instability or complex curve geometry), the SDF function cannot be reliably derived, and sphere marching may fail [S1, unofficial].

## Subtopics

### [Quadratic Bezier Tube Deformation](10138-quadratic-bezier-tube-deformation.md)
Quadratic Bezier Tube Deformation is a technique for mapping points from an arbitrary spline tube defined by quadratic Bezier segments to a canonical vertical cylinder, enabling the use of pre-defined density fields. It solves the problem of inverse parameterization in volumetric rendering by providing a robust Signed Distance Function (SDF) to the curve, which yields the curvilinear abscissa required for texture coordinate calculation. It is distinguished from cubic spline approaches by its computational simplicity in finding the closest point and curvilinear abscissa, though it imposes stricter constraints on segment concatenation.

### Varying Radius Spline Tube Mapping
The radius of the Bezier/spline curve is defined as a function of the curvilinear abscissa, allowing shapes like cones or DNA-like tubes. Varying radius makes the surface normal no longer perpendicular to the curve axis, which is problematic for the SDF function and sphere marching. It can create bugs where boundaries are not detected properly [S1, unofficial].

### Twisting Spline Tube Mapping
Altering the angle used for mapping to induce a twisting effect. The angle can depend on the curvilinear abscissa or time [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The dominant cost is finding the closest point on the spline curve to $P$. For a quadratic Bézier, this involves solving a cubic equation, which is $O(1)$ but involves complex arithmetic. For general splines, iterative methods may be required, making it $O(k)$ where $k$ is the number of iterations. The Frenet frame construction and mapping are $O(1)$.
- **Memory Complexity:** $O(1)$ per point query, assuming the spline curve is stored efficiently.
- **Measured Performance:** Not given in the sources.
- **Scaling:** The method scales with the complexity of the spline curve. More segments or higher-degree curves increase the cost of finding the closest point.
- **Strengths:** Allows for complex, bent, and twisted volumetric shapes using simple procedural density fields defined on a cylinder. The SDF derivation justifies efficient bounding box culling via sphere marching [S1, unofficial].
- **Limitations:** Varying radius complicates the SDF and boundary detection. The method assumes the spline curve is well-behaved enough to find a unique closest point.
- **When to use:** When rendering procedural volumes inside bent or twisted tubes, such as smoke trails, solar flares, or DNA strands.
- **When not to use:** When the radius varies significantly and accurate boundary detection is critical, as the SDF may become unreliable [S1, unofficial].

## Background
The method was introduced in the context of real-time rendering of interstellar dust clouds and solar flares. It allows for the deformation of procedural density fields without storing and transforming the density field itself. The technique relies on the Frenet–Serret formulas, originally developed by Jean Frédéric Frenet (1847–1851) and Joseph Alfred Serret [S1, unofficial]. The specific application to Bézier curves and spline tubes was detailed in a paper on sculpting procedural noise for volumetric rendering [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
