# Local Lipschitz Bound Computation for Hierarchical Implicit Models

Also called: Hierarchical Lipschitz Bound Computation, Tree-based Lipschitz Bound

## Overview
This method computes the local Lipschitz bound $\lambda(\mathbf{e})$ for a segment $\mathbf{e}$ within a hierarchical construction tree of implicit primitives and operators [S1]. It solves the problem of determining safe stepping distances in enhanced sphere tracing (Segment Tracing) by providing a tight bound on the gradient magnitude, which allows for larger, safe steps along the ray compared to conservative global bounds [S1]. It is distinguished from its parent [Lipschitz Constant Estimation](1572-lipschitz-constant-estimation.md) by operating on the specific geometric structure of the implicit model (the construction tree) rather than sampling the bounding box, and from its children by handling general hierarchical operators (Boolean, blending, affine, deformation) rather than specific primitive types [S1].

## How it works
**Inputs:**
- A hierarchical construction tree where leaves are compactly supported skeletal primitives and nodes are operators (Boolean, blending, affine, deformation) [S1].
- A ray segment $\mathbf{e}$ defined by parameter interval $[t, t+\epsilon]$, with center $c$ and radius $r = \|\mathbf{e}\|/2$ [S1].
- The ray direction unit vector $\mathbf{u}$ [S1].

**Outputs:**
- The local Lipschitz bound $\lambda(\mathbf{e})$ for the segment $\mathbf{e}$ [S1].

**Data Structures:**
- **Hierarchical Construction Tree:** A tree structure. Leaves store primitive data (e.g., center, radius, skeleton points). Internal nodes store operator type and pointers to child nodes [S1].

**Method:**
The computation traverses the tree recursively. For a node, the bound is computed based on the operator type. If the primitive or deformation is complex, coarser bounds may be used [S1].

1.  **Base Case (Leaf Primitive):**
    Compute $\lambda(\mathbf{e})$ for the skeletal primitive using the product of two terms:
    $$ \lambda(\mathbf{e}) \leq \|g' \circ d \circ \delta(\mathbf{e})\| \|\nabla d \circ \delta(\mathbf{e}) \cdot \mathbf{u}\| $$
    where:
    - $g$ is the falloff filter function [S1].
    - $d$ is the Euclidean distance to the skeleton [S1].
    - $\delta(\mathbf{e})$ represents the ray parameterization over the segment [S1].
    - $\mathbf{u}$ is the ray direction unit vector [S1].

    **Falloff Bound ($\|g' \circ d \circ \delta(\mathbf{e})\|$):**
    - Compute the interval image of the distance $d \circ \delta(\mathbf{e})$.
    - Approximate the distance interval using the 1-Lipschitz property of distance functions:
      $$ d \circ \delta(\mathbf{e}) \subset [d(c) - r, d(c) + r] $$
      where $c$ is the segment center and $r = \|\mathbf{e}\|/2$ [S1].
    - Bound $g'$ using Interval Analysis on this distance interval [S1].

    **Gradient Bound ($\|\nabla d \circ \delta(\mathbf{e}) \cdot \mathbf{u}\|$):**
    - **General Case:** Use the bound 1, since $\|\nabla d\| = 1$ for Euclidean distance to a skeleton (1-Lipschitz) and $\|\mathbf{u}\|=1$ [S1]:
      $$ \|\nabla d \circ \delta(\mathbf{e}) \cdot \mathbf{u}\| \leq 1 $$
    - **Tighter Bound (Points, Spheres, Line Segments):** Compute the closed-form image of $\nabla d \circ \delta \cdot \mathbf{u}$ over the segment $\mathbf{e}$ [S1]. For a point primitive with center $c$, $\nabla d(p) = (p-c)/\|p-c\|$. The image is the interval $[|\nabla d \circ \delta(t) \cdot \mathbf{u}|, |\nabla d \circ \delta(t+\epsilon) \cdot \mathbf{u}|]$ [S1].

2.  **Coarser Bounds (Complex Primitives/Deformations):**
    If exact computation is difficult, use global bounds for the supporting ray $\Delta$ or bounding sphere $S \supset \mathbf{e}$:
    $$ \lambda(\mathbf{e}) \leq \lambda(\Delta) \quad \text{and} \quad \lambda(\mathbf{e}) \leq \lambda(S) $$
    where $\lambda(\Delta)$ and $\lambda(S)$ are the Lipschitz bounds of $f$ over the entire ray and sphere respectively [S1].

3.  **Internal Nodes (Operators):**
    Compute bounds for children and combine based on operator type [S1]:
    - **Blending (Additive):** $f = f_A + f_B$
      $$ \lambda(\mathbf{e}) = \lambda_A(\mathbf{e}) + \lambda_B(\mathbf{e}) $$
    - **Boolean Union/Intersection (Max/Min):** $f_U = \max(f_A, f_B)$, $f_I = \min(f_A, f_B)$
      $$ \lambda(\mathbf{e}) = \max(\lambda_A(\mathbf{e}), \lambda_B(\mathbf{e})) $$
    - **Pasko Union:** $f_U = f_A + f_B + \sqrt{f_A^2 + f_B^2}$
      $$ \lambda(\mathbf{e}) = 2(\lambda_A(\mathbf{e}) + \lambda_B(\mathbf{e})) $$

4.  **Affine Transformations:**
    Let $f_D = f_N \circ a$, where $a$ is the affine transformation [S1].
    - Transform segment $\mathbf{e}$ to $\mathbf{e}' = a^{-1}(\mathbf{e})$ [S1].
    - For translation/rotation (1-Lipschitz):
      $$ \lambda_D(\mathbf{e}) = \lambda_N(\mathbf{e}') $$
    - For scaling by factor $\alpha$:
      $$ \lambda_D(\mathbf{e}) = \frac{1}{\alpha} \lambda_N(\mathbf{e}') $$

5.  **Deformations:**
    Let $f_W = f_N \circ \omega^{-1}$, where $\omega$ is the warping function [S1].
    - The gradient is $\nabla f_W = \nabla f_N \circ \omega^{-1} \cdot \mathbf{J}_{\omega^{-1}}$, where $\mathbf{J}_{\omega^{-1}}$ is the Jacobian matrix of the inverse warping function [S1].
    - Bound the Lipschitz constant by bounding the gradient norm and Jacobian norm:
      $$ \|\nabla f_W\| \leq \|\nabla f_N \circ \omega^{-1}\| \|\mathbf{J}_{\omega^{-1}}\| $$
    - Compute two local bounds for the transformed segment $\mathbf{e}' = \omega^{-1}(\mathbf{e})$:
      1. Bound of the norm of the gradient $\|\nabla f_N \circ \omega^{-1}(\mathbf{e}')\|$ [S1].
      2. Bound of the norm of the Jacobian $\|\mathbf{J}_{\omega^{-1}}(\mathbf{e}')\|$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_lipschitz_bound(node, segment_e, ray_direction_u):
    """
    Computes the local Lipschitz bound lambda(e) for a segment e in a hierarchical tree.
    
    Args:
        node: The current node in the hierarchical construction tree.
        segment_e: A segment object with attributes:
                   - center: 3D point c
                   - radius: float r = ||e||/2
                   - length: float ||e||
                   - start_t: float t
                   - end_t: float t + epsilon
        ray_direction_u: 3D unit vector u
        
    Returns:
        float: The local Lipschitz bound lambda(e)
    """
    if node.is_leaf():
        primitive = node.primitive
        return compute_primitive_lipschitz(primitive, segment_e, ray_direction_u)
    
    operator_type = node.operator_type
    
    # Recursively compute bounds for children
    lambda_left = compute_lipschitz_bound(node.left_child, segment_e, ray_direction_u)
    lambda_right = compute_lipschitz_bound(node.right_child, segment_e, ray_direction_u)
    
    if operator_type == "BLEND":
        return lambda_left + lambda_right
    elif operator_type in ["BOOLEAN_UNION", "BOOLEAN_INTERSECTION"]:
        return max(lambda_left, lambda_right)
    elif operator_type == "PASKO_UNION":
        return 2 * (lambda_left + lambda_right)
    elif operator_type == "AFFINE":
        # Transform segment to inverse space
        segment_e_prime = node.affine_transform.inverse_transform(segment_e)
        return compute_lipschitz_bound(node.child, segment_e_prime, ray_direction_u)
    elif operator_type == "DEFORMATION":
        # Transform segment to inverse warping space
        segment_e_prime = node.warping_function.inverse_transform(segment_e)
        
        # Compute gradient bound of transformed segment
        lambda_grad = compute_lipschitz_bound(node.child, segment_e_prime, ray_direction_u)
        
        # Compute Jacobian norm bound for the deformation
        lambda_jacobian = compute_jacobian_norm_bound(node.warping_function, segment_e_prime)
        
        return lambda_grad * lambda_jacobian
    else:
        # Fallback to coarser bounds if operator type is not explicitly handled
        # Use global bound for supporting ray or bounding sphere
        return compute_coarser_bound(node, segment_e)

def compute_primitive_lipschitz(primitive, segment_e, ray_direction_u):
    """
    Computes the Lipschitz bound for a leaf skeletal primitive.
    lambda(e) <= ||g' o d o delta(e)|| * ||grad(d) o delta(e) . u||
    """
    # 1. Falloff Bound: ||g' o d o delta(e)||
    # Approximate distance interval using 1-Lipschitz property
    d_center = primitive.distance(segment_e.center)
    r = segment_e.radius
    distance_interval_min = d_center - r
    distance_interval_max = d_center + r
    
    # Compute bound of g' using Interval Analysis on [distance_interval_min, distance_interval_max]
    # This requires knowing the specific falloff function g (e.g., Wyvill C1 sextic)
    # For implementation, assume a function get_falloff_derivative_bound(interval) exists
    falloff_bound = get_falloff_derivative_bound(distance_interval_min, distance_interval_max)
    
    # 2. Gradient Bound: ||grad(d) o delta(e) . u||
    # Try to compute tighter bound for specific primitives
    if primitive.type == "POINT":
        # For point primitive, compute closed form image of grad(d) . u
        # grad(d)(p) = (p - c) / ||p - c||
        # Image is interval [|grad(d)(delta(t)) . u|, |grad(d)(delta(t+epsilon)) . u|]
        t_start = segment_e.start_t
        t_end = segment_e.end_t
        
        p_start = segment_e.center - segment_e.radius * ray_direction_u # Approximation for interval endpoints
        p_end = segment_e.center + segment_e.radius * ray_direction_u
        
        # More precisely, delta(t) = origin + t * u. Need origin from segment context.
        # Assuming segment_e has origin attribute or can be derived.
        # Let's assume segment_e provides start_point and end_point for clarity.
        p_start = segment_e.start_point
        p_end = segment_e.end_point
        
        grad_dot_u_start = abs(dot(primitive.gradient(p_start), ray_direction_u))
        grad_dot_u_end = abs(dot(primitive.gradient(p_end), ray_direction_u))
        
        gradient_bound = max(grad_dot_u_start, grad_dot_u_end)
    else:
        # General case: bound is 1
        gradient_bound = 1.0
        
    return falloff_bound * gradient_bound

def compute_jacobian_norm_bound(warping_function, segment_e):
    """
    Computes the bound of the norm of the Jacobian matrix of the inverse warping function.
    This is a helper for deformation nodes.
    """
    # This requires specific implementation based on the warping function omega.
    # Typically involves evaluating the Jacobian at segment endpoints or using interval arithmetic.
    # Not given in the sources. [G] Placeholder for specific deformation Jacobian computation.
    return 1.0 # Placeholder

def compute_coarser_bound(node, segment_e):
    """
    Fallback to coarser bounds: lambda(e) <= lambda(ray) or lambda(sphere).
    """
    # Not given in the sources. [G] Placeholder for global bound retrieval.
    return 1.0 # Placeholder
```

**Parameters:**
- Typical falloff functions: Wyvill's $C^1$ sextic, Wyvill's $C^2$ sextic, $C^1$ quartic [S1].
- Scaling factor $\alpha$ for affine transformations [S1].

**Edge Cases and Pitfalls:**
- **Complex Primitives/Deformations:** Exact computation of $\lambda(\mathbf{e})$ may be very complex. Use coarser bounds $\lambda(\mathbf{e}) \leq \lambda(\Delta)$ and $\lambda(\mathbf{e}) \leq \lambda(S)$ to ensure correctness and efficiency [S1].
- **Distance Interval Approximation:** The approximation $d \circ \delta(\mathbf{e}) \subset [d(c) - r, d(c) + r]$ relies on the 1-Lipschitz property of distance. This is valid but may be loose for complex skeletons [S1].
- **Gradient Bound Tightness:** The general bound of 1 for $\|\nabla d \circ \delta \cdot \mathbf{u}\|$ is not accurate. Compute closed-form expressions for points, spheres, or line segments to obtain tighter bounds, especially when the ray is orthogonal to the gradient [S1].

## Subtopics

### [Lipschitz Bound Computation for Skeletal Primitives](3966-lipschitz-bound-computation-for-skeletal-primitives.md)
This method computes the local Lipschitz bound $\lambda$ for skeletal primitives by evaluating the maximum absolute value of the derivative of the falloff filter function $g(x)$ over a specific interval $[a, b]$. It solves the problem of determining safe stepping distances in enhanced sphere tracing by providing a tight bound on the gradient magnitude, which allows for larger, safe steps along the ray compared to conservative global bounds. It is distinguished from generic interval arithmetic strategies by relying on closed-form analytical solutions for specific standard falloff functions, avoiding numerical approximation errors.

## Operating characteristics
- **Time Complexity:** Depends on the tree traversal depth and the complexity of the primitives/operators. For complex primitives/deformations, coarser bounds are used to reduce computation [S1].
- **Memory Complexity:** Proportional to the size of the hierarchical construction tree [S1].
- **Measured Performance:** Reduces the number of field function queries (#f) and accelerates Segment Tracing compared to Sphere Tracing, which necessitates many smaller steps [S1].
- **Strengths:** Local bounds are significantly smaller than global bounds, leading to fewer queries and faster rendering [S1]. Handles hierarchical models efficiently via bounding volume hierarchy pruning [S1].
- **Limitations:** Exact computation can be complex for some primitives or deformation operators. Requires implementation of interval analysis for falloff functions [S1].
- **When to Use:** When rendering hierarchical implicit models with skeletal primitives and deformations, and when Segment Tracing is employed for faster ray marching.
- **When Not to Use:** For simple implicit models where global bounds are sufficient, or when the overhead of tree traversal outweighs the benefits of tighter bounds.

## Background
- **Origin:** The method is described in "Segment Tracing Using Local Lipschitz Bounds" [S1]. It adapts to hierarchical function representations such as procedurally defined scalar fields [RMD11] or function representations [PASS95].
- **History:** The work builds on earlier implicit surface modeling techniques, including blending operators [Har96] and Pasko's Boolean operations [PASS95]. It extends the concept of Lipschitz bounds from global estimates to local, tree-based computations.
- **Usage:** Used in computer graphics for rendering complex implicit surfaces, particularly those defined by skeletal primitives and deformations, using Segment Tracing for efficient ray marching.

## Sources
[S1] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
