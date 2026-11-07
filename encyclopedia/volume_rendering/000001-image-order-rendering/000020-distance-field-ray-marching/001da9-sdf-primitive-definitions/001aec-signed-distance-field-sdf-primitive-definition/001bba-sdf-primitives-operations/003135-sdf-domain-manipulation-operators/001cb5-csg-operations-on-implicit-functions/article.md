# CSG Operations on Implicit Functions

Also called: CSG Operations, Boolean SDF operators, SDF Boolean Operators, SDF Boolean Operations, CSG implicit surfaces, Constructive Solid Geometry with SDFs, CSG with SDFs, CSG SDF combination, CSG Operations for SDFs

## Overview
CSG Operations on Implicit Functions combine multiple Signed Distance Functions (SDFs) into a single implicit function using Boolean operations (Union, Intersection, Difference) to define complex volumetric shapes [S1][S3, S7, unofficial]. Unlike tree-based CSG evaluation which computes the final distance field value on-the-fly during rendering, this method often pre-computes the logical classification of points in a volume space using a Point-Classification Map (PCM) or directly composites voxel buffers [S1][S7, unofficial]. It sets itself apart from [Stack-Based SDF Editor](1696-stack-based-sdf-editor.md) by focusing on the spatial composition of volume data (voxels or classification maps) rather than a procedural stack of operations, and from [CSG Operations on Level Sets](1627-constructive-solid-geometry-csg-on-level-sets.md) by providing concrete implementation strategies for discrete volume representations.

## How it works
### Inputs and Outputs
**Inputs**:
- Two or more SDF primitives, each returning a signed distance value $d$ for a query point $\mathbf{p}$.
- A Boolean operator: Union, Intersection, or Difference.
- Optional smooth blending parameter $k$ for smooth variants.

**Outputs**:
- A composite SDF returning the signed distance to the resulting implicit surface.
- In discrete implementations: Updated material grids and edge data (Hermite data) representing the combined volume [S1].

### Data Structures
For procedural (on-the-fly) evaluation, the data structure is a binary tree [S2, S4, unofficial]:
- `Node`: Contains `operation` (Union, Intersection, Difference), `left` child, `right` child.
- `Leaf`: Contains `sdf` function pointer and `material` ID.

For discrete volume generation [S1]:
- `Material Grid`: 3D array of material indices. Initialized to "air".
- `Edge Arrays`: Pre-allocated arrays for starting grid point indices, intersection normals, and Zero Crossing positions. Size must accommodate the maximum potential edges to avoid resizing overhead [S1].

### Formulas
The core arithmetic operations map directly to set theory [S1][S3, unofficial][S8][S10, S13, unofficial]:

1.  **Union** ($A \cup B$): The point is inside if it is inside *either* shape. The distance is the minimum of the two distances.
    $$ d_{union}(\mathbf{p}) = \min(d_A(\mathbf{p}), d_B(\mathbf{p})) $$

2.  **Intersection** ($A \cap B$): The point is inside if it is inside *both* shapes. The distance is the maximum of the two distances.
    $$ d_{intersect}(\mathbf{p}) = \max(d_A(\mathbf{p}), d_B(\mathbf{p})) $$

3.  **Difference** ($A \setminus B$): The point is inside $A$ but *outside* $B$. This is equivalent to intersecting $A$ with the complement of $B$. The distance is the maximum of $A$'s distance and the negation of $B$'s distance.
    $$ d_{diff}(\mathbf{p}) = \max(d_A(\mathbf{p}), -d_B(\mathbf{p})) $$

4.  **XOR** ($A \oplus B$): The point is inside exactly one of the shapes.
    $$ d_{xor}(\mathbf{p}) = \max(\min(d_A(\mathbf{p}), d_B(\mathbf{p})), -\max(d_A(\mathbf{p}), d_B(\mathbf{p}))) $$
    [S10, unofficial]

5.  **Smooth Union** ($smin$): Blends the union with a fillet of radius $k$.
    $$ h = \text{clamp}\left(\frac{0.5 + 0.5 \frac{d_B - d_A}{k}}{1.0}, 0.0, 1.0\right) $$
    $$ d_{smooth\_union} = \text{mix}(d_B, d_A, h) - k \cdot h \cdot (1.0 - h) $$
    [S4, S12, unofficial]

### Parameters
- **$k$**: Smooth blending factor. Typical values range from 0.1 to 1.0 [S4, unofficial]. Controls the radius of the smooth fillet.
- **$n$**: Grid resolution for discrete generation [S1].
- **$s$**: Cell size for discrete generation [S1].

### Step-by-Step Procedure (Procedural/Tree Evaluation)
1.  **Evaluate Children**: Recursively evaluate the left child $L$ and right child $R$ at the query point $\mathbf{p}$ to obtain distances $d_L$ and $d_R$.
2.  **Apply Operator**:
    - If Union: Return $\min(d_L, d_R)$.
    - If Intersection: Return $\max(d_L, d_R)$.
    - If Difference: Return $\max(d_L, -d_R)$.
3.  **Return Result**: The resulting distance is the signed distance to the composite surface.

### Step-by-Step Procedure (Discrete Volume Generation)
1.  **Initialization**: Create a blank set of Hermite data with no edge data and a material grid where all indices are set to "air" [S1].
2.  **AABB Culling**: Limit processing to grid points inside the SDF's Axis-Aligned Bounding Box (AABB). For Union and Difference, unaffected grid points are ignored. For Intersection, all solid material indices outside the AABB must be set to air [S1].
3.  **World Position Calculation**: For each grid point $(x,y,z)$, calculate the local offset:
    $$ \vec{offset} = \left( \frac{x \cdot s}{n}, \frac{y \cdot s}{n}, \frac{z \cdot s}{n} \right) $$
    Add the offset to the world octant's min position to get the world position $\mathbf{p}_{world}$ [S1].
4.  **SDF Sampling**: Sample the SDF at $\mathbf{p}_{world}$. If the distance indicates the point is inside the surface, set the material index to solid; otherwise, keep it as air [S1].
5.  **Edge Generation**: Process edges along X, Y, and Z axes separately. Determine ending index $b$ from starting index $a$ using fixed offsets (e.g., for Y-axis: $b = a + (n+1)$) [S1]. Check if material indices at $a$ and $b$ differ. If they do, use Zero Crossing approximation to obtain surface intersection data (normal and position) [S1].
6.  **Combination**:
    - **Union**: Non-air materials of the new set override existing materials. Edges of the new set override existing edges if their Zero Crossing position is closer to the air grid point. Discard existing edges that no longer exhibit a material change [S1].
    - **Difference**: Non-air materials of the new set result in air. Edges of the new set override existing edges only if they connect different materials in the existing set and their Zero Crossing position is closer to the non-air grid point. Invert normals of adopted edges from the new set [S1].
    - **Intersection**: Set all materials of the existing set to air except those solid in both sets (choose new set's material). Edges of the new set override existing edges if they exhibit a material change in the existing set and their Zero Crossing position is closer to the non-air grid point. Discard existing edges that no longer exhibit a material change [S1].
7.  **Edge Combination Algorithm**: Iterate over generated edges (sorted by starting index). For each generated edge with a material change, enter an inner loop to process existing edges up to the current generated edge. If an existing edge has the same starting index, resolve conflict based on CSG type. Collect remaining existing edges [S1].

### Pseudocode (Procedural)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_csg(node, p):
    """
    Recursively evaluate a CSG tree at point p.
    node: Tree node (Leaf or Node)
    p: 3D vector query point
    """
    if node is Leaf:
        return node.sdf(p)
    
    # Evaluate children
    d_left = evaluate_csg(node.left, p)
    d_right = evaluate_csg(node.right, p)
    
    if node.operation == Union:
        return min(d_left, d_right)
    elif node.operation == Intersection:
        return max(d_left, d_right)
    elif node.operation == Difference:
        return max(d_left, -d_right)
    else:
        raise ValueError("Unknown CSG operation")

def smooth_union(a, b, k):
    """
    Smooth union of two distances.
    a, b: signed distances
    k: smoothness factor
    """
    h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0)
    return mix(b, a, h) - k * h * (1.0 - h)
```

### Edge Cases and Pitfalls
- **Scaling**: Scaling an SDF requires scaling the input coordinates by the inverse of the scale factor and scaling the output distance by the scale factor to maintain correct distance values [S3, unofficial].
- **Subtraction Artefacts**: The difference formula $\max(-a, b)$ is exact only when the surface of $A$ is entirely contained within $B$. If $A$ pokes through the boundary of $B$, the resulting SDF is not Lipschitz-1, and sphere marching may skip surfaces. Add a small EPSILON correction or tighten the step size in those regions [S12, unofficial].
- **DeMorgan’s Theorem**: DeMorgan’s theorem (intersection = complement of union of complements) does not work directly with min/max operators for distance bounds because the minimum operator is not complemented properly. Using max ensures the bound holds [S8].
- **Material Coloring**: Standard polygonizers do not consider materials, leading to drastic color changes between vertices at CSG boundaries. Material changes must be treated as surface boundaries (inside/outside changes) to separate colors correctly [S14, unofficial].

## Subtopics

### [Constructive Solid Geometry (CSG) on Level Sets](1627-constructive-solid-geometry-csg-on-level-sets.md)
Combines multiple SDFs into a single implicit function using Boolean operations. Unlike tree-based CSG evaluation, this method often pre-computes the logical classification of points in a volume space using a Point-Classification Map (PCM) or directly composites voxel buffers. It sets itself apart from [Stack-Based SDF Editor](1696-stack-based-sdf-editor.md) by focusing on the spatial composition of volume data (voxels or classification maps) rather than a procedural stack of operations, and from [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) by providing concrete implementation strategies for discrete volume representations.

### [Stack-Based SDF Editor](1696-stack-based-sdf-editor.md)
An interactive editor that constructs Signed Distance Fields (SDFs) from primitives and operations using a stack-based interface similar to Reverse Polish Notation (RPN), implicitly representing a tree with postorder traversal. It differs from explicit tree editors by deferring structure resolution until compilation, allowing users to build complex scenes by pushing operations onto a linear stack where only the top item is visible until combined.

## Operating characteristics
- **Time Complexity**:
    - Procedural evaluation: $O(1)$ per sample (depth of tree).
    - Discrete generation: Material index generation and combination is $O(n^3)$, where $n$ is the resolution. Edge generation is $O(3 \times n^3)$. Edge combination is $O(n)$ [S1].
- **Memory Complexity**:
    - Procedural: $O(D)$ where $D$ is the depth of the CSG tree.
    - Discrete: $O(n^3)$ for the material grid. Edge arrays must be pre-allocated to maximum potential size, which can be significant [S1].
- **Performance**:
    - Discrete generation is costly but can be optimized by limiting work to grid points inside the SDF's AABB [S1].
    - Procedural evaluation is efficient for ray marching but requires careful handling of smooth blending parameters to avoid artifacts.
- **Strengths**:
    - Simple arithmetic operations for complex shapes.
    - Analytic normals can be computed via gradient of the composite SDF.
    - Supports smooth blending for organic transitions.
- **Limitations**:
    - Subtraction can produce non-Lipschitz-1 surfaces if shapes intersect improperly, causing marching artifacts [S12, unofficial].
    - Discrete methods require significant memory for high-resolution grids.
    - Smooth operations introduce approximation errors and require tuning of the $k$ parameter.
- **When to Use**:
    - Use for rendering complex geometries composed of multiple primitives, such as near-tangent intersections of two spheres [S7, unofficial].
    - Use discrete methods when pre-computing volume data for terrain or voxel-based rendering is required [S1].
- **When Not to Use**:
    - Avoid for scenes with highly dynamic CSG operations where tree traversal overhead is prohibitive.
    - Avoid discrete methods for real-time applications with limited memory.

## Background
CSG Operations on Implicit Functions originated from the application of Constructive Solid Geometry (CSG) to implicit surfaces. The technique leverages the fact that SDFs are scalar fields, allowing Boolean operations to be reduced to arithmetic operations on the distance values [S13, unofficial]. The union operation corresponds to the minimum of distances, intersection to the maximum, and difference to the maximum of the first distance and the negated second distance [S1][S3, unofficial][S8][S10, S13, unofficial]. This approach was popularized in the context of ray marching and SDF rendering by developers like Iñigo Quilez and Jamie Wong, who demonstrated its efficiency and elegance in creating complex scenes from simple primitives [S3, S12, S13, unofficial]. The method is widely used in real-time graphics, procedural generation, and implicit surface rendering [S7, S12, unofficial].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S2] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S3] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S4] skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, https://github.com/skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S5] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S6] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S7] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S8] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S9] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S10] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S11] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S12] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S13] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S14] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
