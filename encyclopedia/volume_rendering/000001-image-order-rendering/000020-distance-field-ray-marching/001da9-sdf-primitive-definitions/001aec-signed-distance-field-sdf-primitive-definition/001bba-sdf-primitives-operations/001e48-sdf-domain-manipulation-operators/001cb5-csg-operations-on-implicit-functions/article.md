# CSG Operations on Implicit Functions

Also called: CSG implicit surfaces, CSG with SDFs, Constructive Solid Geometry (CSG) with SDFs, SDF Boolean Operations, SDF Difference, SDF Intersection, SDF Union, Union/Intersection via min/max

## Overview
CSG Operations on Implicit Functions is a technique for creating complex geometric shapes from simple Signed Distance Function (SDF) primitives using boolean operators [S1, S2, S3, S4, S5, S6, S7, S8, S9]. It maps the set-theoretic operations of union, intersection, and difference directly to arithmetic operations on the scalar distance values returned by the SDFs [S1, S2, S3, S4, S5, S6, S7, S8, S9]. Unlike explicit mesh-based CSG, which requires complex boolean intersection algorithms on polygonal geometry, SDF CSG reduces scene composition to simple `min` and `max` evaluations [S2, S8]. This allows for the rendering of near-tangent intersections and complex composite geometries with minimal computational overhead per sample [S4, S8].

## How it works
### Inputs and Outputs
- **Inputs**: Two (or more) SDF functions, $f_A(p)$ and $f_B(p)$, which return the signed distance from a point $p$ to the surface of object A or B. Negative values indicate the interior, positive values the exterior, and zero the surface.
- **Output**: A composite SDF function $f_{scene}(p)$ that represents the boolean combination of the input shapes.

### Data Structures
The scene can be represented as a binary tree where nodes are operations and leaves are primitives [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Tree:
    class Node:
        operation: Op  # Union, Intersection, Difference
        left: Tree
        right: Tree
    
    class Leaf:
        sdf: SDF       # Function returning distance
        material: MaterialID

class Op:
    Union = 0
    Intersection = 1
    Difference = 2
```

### Formulas
The core arithmetic mappings for standard CSG operations are [S2, S5, S6, S7, S8, S9]:

1.  **Union**: The distance to the union is the minimum of the individual distances. A point is inside the union if it is inside *at least one* of the shapes.
    $$ f_{union}(p) = \min(f_A(p), f_B(p)) $$

2.  **Intersection**: The distance to the intersection is the maximum of the individual distances. A point is inside the intersection only if it is inside *both* shapes.
    $$ f_{intersect}(p) = \max(f_A(p), f_B(p)) $$

3.  **Difference (Subtraction)**: The difference $A \setminus B$ is computed by intersecting $A$ with the complement of $B$. The complement of an SDF is obtained by negating its distance values.
    $$ f_{sub}(p) = \max(f_A(p), -f_B(p)) $$

4.  **XOR**: Exclusive OR can be constructed from the above operations.
    $$ f_{xor}(p) = \max(\min(f_A(p), f_B(p)), -\max(f_A(p), f_B(p))) $$

### Numbered Steps
1.  **Leaf Evaluation**: For a leaf node, evaluate the primitive SDF function at the query point $p$ to get distance $d$ and material ID [S1].
2.  **Recursive Evaluation**: For an internal node, recursively evaluate the left child $a$ and right child $b$ at point $p$ [S1].
3.  **Operation Application**:
    -   If `Op.Union`: Return $\min(a.d, b.d)$. If $a.d \le b.d$, propagate $a$'s material; otherwise propagate $b$'s material [S1].
    -   If `Op.Intersection`: Return $\max(a.d, b.d)$. Propagate the material of the shape that determined the maximum distance (the "tighter" bound) [S1].
    -   If `Op.Difference`: Return $\max(a.d, -b.d)$. If $a.d \ge -b.d$, propagate $a$'s material; otherwise propagate $b$'s material (note: the distance value is negated in the return, but the material source is determined by which term was larger) [S1].

### Pseudocode
The following Python-like pseudocode implements the recursive evaluation of a CSG tree, including material propagation [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_csg(node: Tree, pos: vec3) -> tuple[float, int]:
    """
    Evaluates the CSG tree at position pos.
    Returns (distance, material_id).
    """
    if isinstance(node, Tree.Leaf):
        d = node.sdf(pos)
        return (d, node.material)

    # Recursive evaluation of children
    a_dist, a_mat = evaluate_csg(node.left, pos)
    b_dist, b_mat = evaluate_csg(node.right, pos)

    if node.operation == Op.Union:
        if a_dist <= b_dist:
            return (a_dist, a_mat)
        else:
            return (b_dist, b_mat)
            
    elif node.operation == Op.Intersection:
        if a_dist >= b_dist:
            return (a_dist, a_mat)
        else:
            return (b_dist, b_mat)
            
    elif node.operation == Op.Difference:
        # Difference is max(a, -b)
        # We compare a_dist and -b_dist to decide material
        if a_dist >= -b_dist:
            return (a_dist, a_mat)
        else:
            return (-b_dist, b_mat)
            
    else:
        raise ValueError("Unknown operation")
```

### Parameters and Variants
-   **Smooth Union**: Uses a polynomial blend function $smin(a, b, k)$ with a smoothness parameter $k \in [0, 2]$ to create rounded fillets instead of sharp edges [S7, S8].
-   **Stairs Operations**: `Union Stairs`, `Difference Stairs`, and `Intersection Stairs` create stepped transitions between shapes, controlled by step width $r$ and number of steps $n$ [S7].
-   **Noise**: Adds Perlin noise to the signed distance, controlled by a noise proportion parameter $noiseProp \in [0, 0.5]$ [S7].
-   **Onion**: Creates a hollow object by subtracting a smaller inner sphere, controlled by a `thickness` parameter [S7].

### Edge Cases and Pitfalls
-   **Scaling**: Scaling an SDF requires scaling the input coordinates by the inverse of the scale factor and scaling the output distance by the scale factor to maintain correct distance values [S2].
-   **DeMorgan’s Theorem**: DeMorgan’s theorem ($A \cap B = \overline{\overline{A} \cup \overline{B}}$) does not work directly with min/max operators for distance bounds because the minimum operator is not complemented properly. Using `max` for intersection ensures the bound holds [S5].
-   **Material Boundaries**: The polygonizer does not consider materials, leading to drastic color changes between vertices at CSG boundaries. Material changes must be treated as surface boundaries (inside/outside changes) to separate colors correctly [S9].
-   **Distance Bounds**: For intersection, the resulting function is a *bound* on the true distance, not necessarily the exact distance. Specifically, in Case II (where $d_A < 0$ and $d_B > 0$), the closest point in $A$ to $p$ may not be in the intersection, but no point in the intersection can be closer than $d_B$ [S5].

## Subtopics

### [Stack-Based SDF Editor](1696-stack-based-sdf-editor.md)
An interactive editor that constructs SDFs from primitives and operations using a stack-based interface similar to Reverse Polish Notation (RPN), implicitly representing a tree with postorder traversal. It differs from explicit tree editors by deferring structure resolution until compilation, allowing users to build complex scenes by pushing operations onto a linear stack where only the top item is visible until combined.

### Constructive Solid Geometry (CSG) on Level Sets
A voxel-based implementation of CSG operations on discrete level set data. Operations are performed element-wise on voxel buffers [S10].
-   **Union**: $A_{new}(i,j,k) = \min(A(i,j,k), B(i,j,k))$ [S10].
-   **Intersection**: $A_{new}(i,j,k) = \max(A(i,j,k), B(i,j,k))$ [S10].
-   **Difference**: $A_{new}(i,j,k) = \max(A(i,j,k), -B(i,j,k))$ [S10].
In user interface terms, intersection corresponds to a copy operation, union to a paste operation, and difference to a cut operation [S10].

## Operating characteristics
-   **Time Complexity**: $O(N)$ per sample, where $N$ is the depth of the CSG tree. Each node requires one arithmetic operation and one comparison.
-   **Memory Complexity**: $O(N)$ for the tree structure. No additional memory is required for the scene representation compared to storing individual primitives.
-   **Strengths**:
    -   Simplicity: Boolean operations are reduced to `min`/`max` [S2, S8].
    -   No Mesh Booleaning: Avoids the computational complexity and numerical instability of polygonal boolean operations [S8].
    -   Free Normals: Surface normals are derived from the gradient of the composite SDF, which is consistent with the geometry [S8].
-   **Limitations**:
    -   Approximate Distance: The resulting SDF is a bound, not the exact distance to the CSG boundary, which can affect sphere tracing step sizes (though still valid for rendering) [S5].
    -   Sharp Edges: Standard CSG produces sharp, non-smooth edges unless smooth variants are used [S7].
    -   Material Handling: Requires explicit tracking of material IDs through the tree to avoid rendering artifacts at boundaries [S1, S9].

## Background
CSG was introduced by Ian Buckland and others in the 1970s for solid modeling [S2, S3]. The application of CSG to implicit surfaces using min/max operators was formalized by Ricci in 1974 [S5]. The technique became widely popular in real-time graphics with the advent of ray marching and SDFs, notably demonstrated by Iñigo Quilez and Jamie Wong in the 2010s [S1, S2, S8]. It is used in raymarched renderers, voxel editors, and differentiable rendering pipelines [S4, S7, S10].

## Sources
[S1] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S2] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S3] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S4] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S5] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S6] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S7] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27
[S8] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S9] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S10] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
