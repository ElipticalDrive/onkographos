# CSG Operations on Implicit Functions

Also called: CSG Operations, CSG implicit surfaces, SDF Boolean Operators, SDF Boolean Operations, CSG with SDFs, Constructive Solid Geometry with SDFs

## Overview
CSG Operations on Implicit Functions is a method of creating complex geometric shapes from simple Signed Distance Function (SDF) primitives via boolean operations: union, intersection, and difference [S1, unofficial][S3, unofficial][S4, unofficial][S5][S7, unofficial][S10, unofficial]. It combines scalar fields using binary node operations to construct complex implicit surfaces [S2, unofficial]. Unlike explicit mesh-based CSG, which requires complex boolean intersection algorithms on polygonal geometry, SDF CSG reduces scene composition to simple arithmetic on distance values [S2, unofficial][S8, unofficial]. This allows for the rendering of near-tangent intersections and complex composite geometries with minimal computational overhead per sample [S4, unofficial][S8, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: Two SDF functions, $f_A(p)$ and $f_B(p)$, which return the signed distance from a query point $p$ to the surface of object A or B [S2, unofficial][S7, unofficial].
- **Output**: A composite SDF function $f_{scene}(p)$ that represents the boolean combination of the input shapes [S2, unofficial][S4, unofficial].

### Data Structures
The scene is represented as a node hierarchy where each node holds pointers to two child `ScalarField` objects [S2, unofficial]. Nodes are evaluated recursively or iteratively to compute the final distance [S2, unofficial].

### Formulas
The core arithmetic mappings for standard CSG operations are [S2, unofficial][S5][S7, unofficial][S8, unofficial][S9, unofficial]:
1. **Union**: The distance to the union is the minimum of the individual distances. A point is inside the union if it is inside at least one of the shapes [S2, unofficial][S5][S8, unofficial][S9, unofficial].
   $$ f_{union}(p) = \min(f_A(p), f_B(p)) $$
2. **Intersection**: The distance to the intersection is the maximum of the individual distances. A point is inside the intersection only if it is inside both shapes [S2, unofficial][S5][S8, unofficial][S9, unofficial].
   $$ f_{intersect}(p) = \max(f_A(p), f_B(p)) $$
3. **Difference (Subtraction)**: The difference $A \setminus B$ is computed by intersecting $A$ with the complement of $B$. The complement of an SDF is obtained by negating its distance values [S2, unofficial][S5].
   $$ f_{sub}(p) = \max(f_A(p), -f_B(p)) $$
4. **XOR**: Exclusive OR can be constructed from the above operations [S6, unofficial].
   $$ f_{xor}(p) = \max(\min(f_A(p), f_B(p)), -\max(f_A(p), f_B(p))) $$

### Numbered Steps
1. Instantiate nodes with pointers to child SDFs: `new UnionNode(a, b)`, `new IntersectionNode(a, b)`, `new DifferenceNode(a, b)` [S2, unofficial].
2. Chain nodes by passing output nodes as inputs to subsequent nodes to build the scene graph [S2, unofficial].
3. Evaluate the final root node's `Value(p)` method at the query point to get the combined SDF distance [S2, unofficial].

### Pseudocode
The following Python-like pseudocode implements the recursive evaluation of a CSG tree [S2, unofficial][S7, unofficial].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ScalarField:
    def __init__(self, func):
        self.func = func
    def evaluate(self, pos):
        return self.func(pos)

class Node:
    def __init__(self, op, left, right):
        self.operation = op
        self.left = left
        self.right = right
    def evaluate(self, pos):
        a_dist = self.left.evaluate(pos)
        b_dist = self.right.evaluate(pos)
        if self.operation == "Union":
            return min(a_dist, b_dist)
        elif self.operation == "Intersection":
            return max(a_dist, b_dist)
        elif self.operation == "Difference":
            return max(a_dist, -b_dist)
        else:
            raise ValueError("Unknown operation")

def evaluate_csg(node, pos):
    """Evaluates the CSG tree at position pos. Returns the combined signed distance."""
    if isinstance(node, ScalarField):
        return node.evaluate(pos)
    return node.evaluate(pos)
```

### Parameters and Variants
- **Smooth Union**: Alternative Union with smooth transition [S7, unofficial]. Takes additional float $k$ (0 to 2) determining smoothness, typically 0.1 to 1.0 [S7, unofficial][S2, unofficial]. Implementation: `h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0); return mix(b, a, h) - k * h * (1.0 - h)` [S7, unofficial][S8, unofficial].
- **Stairs Operations**: `Union Stairs`, `Difference Stairs`, and `Intersection Stairs` create steps between edges [S7, unofficial]. Takes floats $r$ (step width) and $n$ (number of steps) [S7, unofficial].
- **Noise**: Unary operation adding Perlin noise to signed distance. Takes float `noiseProp` (0 to 0.5). Uses GLSL mix function [S7, unofficial].
- **Onion**: Unary operation creating a hole inside a shape. Takes float `thickness` (smaller value = larger hole) [S7, unofficial].

### Edge Cases and Pitfalls
- **Scaling**: Scaling an SDF requires scaling the input coordinates by the inverse of the scale factor and scaling the output distance by the scale factor to maintain correct distance values [S1, unofficial][S8, unofficial].
- **DeMorgan’s Theorem**: DeMorgan’s theorem ($A \cap B = \overline{\overline{A} \cup \overline{B}}$) does not work directly with min/max operators for distance bounds because the minimum operator is not complemented properly. Using `max` ensures the bound holds [S5].
- **Subtraction Artefact**: `max(-a, b)` is exact only when the surface of A is entirely contained within B. If A pokes through the boundary of B, the resulting SDF is not Lipschitz-1 and sphere marching may skip surfaces — add a small EPSILON correction or tighten the step size in those regions [S8, unofficial].
- **Material Boundaries**: The polygonizer does not consider materials, leading to drastic color changes between vertices at CSG boundaries. To separate colors, material changes must be treated as surface boundaries (inside/outside changes) [S10, unofficial].
- **Distance Bounds**: For intersection, the resulting function is a bound on the true distance, not necessarily the exact distance. Specifically, in Case II ($d_A < 0, d_B > 0$), the closest point in A to p may not be in the intersection, but no point in the intersection can be closer than $d_B$ [S5].

## Subtopics

### [Stack-Based SDF Editor](1696-stack-based-sdf-editor.md)
An interactive editor that constructs SDFs from primitives and operations using a stack-based interface similar to Reverse Polish Notation (RPN), implicitly representing a tree with postorder traversal. It differs from explicit tree editors by deferring structure resolution until compilation, allowing users to build complex scenes by pushing operations onto a linear stack where only the top item is visible until combined.

### Constructive Solid Geometry (CSG) on Level Sets
A voxel-based implementation of CSG operations on discrete level set data [S11, unofficial]. Operations are performed element-wise on voxel buffers [S11, unofficial].
- **Union**: $A_{new}(i,j,k) = \min(A(i,j,k), B(i,j,k))$ [S11, unofficial].
- **Intersection**: $A_{new}(i,j,k) = \max(A(i,j,k), B(i,j,k))$ [S11, unofficial].
- **Difference**: $A_{new}(i,j,k) = \max(A(i,j,k), -B(i,j,k))$ [S11, unofficial].
In user interface terms, intersection corresponds to a copy operation, union to a paste operation, and difference to a cut operation [S11, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources. [G]
- **Memory Complexity**: Not given in the sources. [G]
- **Strengths**:
    - Simplicity: Boolean operations are reduced to `min`/`max` [S2, unofficial][S8, unofficial].
    - No Mesh Booleaning: Avoids the computational complexity and numerical instability of polygonal boolean operations [S8, unofficial].
    - Free Normals: Surface normals are derived from the gradient of the composite SDF, which is consistent with the geometry [S8, unofficial].
- **Limitations**:
    - Approximate Distance: The resulting SDF is a bound, not the exact distance to the CSG boundary, which can affect sphere tracing step sizes (though still valid for rendering) [S5].
    - Sharp Edges: Standard CSG produces sharp, non-smooth edges unless smooth variants are used [S7, unofficial].
    - Material Handling: The polygonizer does not consider materials, leading to drastic color changes between vertices at CSG boundaries [S10, unofficial].

## Background
CSG was introduced in the 1970s for solid modeling [S2, unofficial][S3, unofficial]. The application of CSG to implicit surfaces using min/max operators was formalized by Ricci in 1974 [S5]. The technique became widely popular in real-time graphics with the advent of ray marching and SDFs, notably demonstrated by Iñigo Quilez and Jamie Wong in the 2010s [S1, unofficial][S8, unofficial]. It is used in voxel editors and differentiable rendering pipelines [S4, unofficial][S10, unofficial][S11, unofficial].

## Sources
[S1] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S2] skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, https://github.com/skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S3] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S4] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S5] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S6] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S7] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S8] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S9] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S10] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S11] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
