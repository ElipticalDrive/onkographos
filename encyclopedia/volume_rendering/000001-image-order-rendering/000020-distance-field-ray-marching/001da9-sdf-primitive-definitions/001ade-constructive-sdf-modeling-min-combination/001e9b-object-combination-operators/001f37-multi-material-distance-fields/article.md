# Multi-Material Distance Fields

Also called: Augmented SDF signature, Color-Coded SDFs, Material-Augmented Signed Distance Function, MaterialSDF, SDF with Material Data, SDF with material ID

## Overview
Multi-Material Distance Fields extend standard Signed Distance Functions (SDFs) by returning both a distance value and a material identifier (or color index) for every query point, enabling per-object material assignment within Constructive Solid Geometry (CSG) scenes [S1, unofficial][S2, unofficial]. Unlike sibling operators like [SDF Smooth Union](1817-sdf-smooth-union.md) which focus on geometric blending, this method preserves the sharp boolean logic of CSG operators while propagating material attributes through the scene graph, allowing for distinct shading of individual primitives without requiring complex texture atlases or multiple render passes [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $p$ in world space; a CSG tree structure where each leaf node contains an SDF primitive and a material ID.
*   **Output:** A tuple $(d, m)$ where $d$ is the signed distance to the nearest surface and $m$ is the material identifier (e.g., an integer index or a normalized color value) associated with that surface [S1, unofficial][S2, unofficial].

**Data Structures:**
The core data structure is an augmented vector type. In GLSL-style shading languages, this is typically `float2` or `vec2` [S2, unofficial]. In functional implementations, it is a tuple [S1, unofficial]:
*   **Distance Field Vector:**
    *   $x$-component: The signed distance value $d$.
    *   $y$-component: The material identifier $m$ (e.g., color index, material ID) [S1, unofficial][S2, unofficial].
*   **CSG Leaf Node:** Contains the primitive SDF function and the static material ID assigned to that primitive [S1, unofficial].

**Numbered Steps:**
1.  **Leaf Evaluation:** For a leaf node in the CSG tree, evaluate the primitive's SDF at point $p$ to get distance $d_{prim}$, and retrieve the leaf's associated material ID $m_{leaf}$. Return the vector $(d_{prim}, m_{leaf})$ [S1, unofficial][S2, unofficial].
2.  **Recursive Combination:** For internal nodes, recursively evaluate the left child $a$ and right child $b$ to obtain vectors $(d_a, m_a)$ and $(d_b, m_b)$ [S1, unofficial].
3.  **Operator Application:** Apply the specific CSG operator (Union, Intersection, or Difference) to the distance components $d_a$ and $d_b$ to compute the resulting distance $d_{res}$. Simultaneously, select the appropriate material ID $m_{res}$ based on the operator's logic [S1, unofficial][S2, unofficial].
4.  **Return Result:** Return the combined vector $(d_{res}, m_{res})$ [S1, unofficial].
5.  **Shading:** In the raymarching loop, use the returned $d$ for marching steps. Upon hit ($d \approx 0$), use $m$ to sample a color ramp or material property buffer to determine the final color [S2, unofficial].

**Formulas:**
Let $a = (d_a, m_a)$ and $b = (d_b, m_b)$ be the results from the left and right children.

*   **Union:** Returns the closer object.
    $$ d_{res} = \min(d_a, d_b) $$
    $$ m_{res} = \begin{cases} m_a & \text{if } d_a \le d_b \\ m_b & \text{otherwise} \end{cases} $$
    [S1, unofficial][S2, unofficial]

*   **Intersection:** Returns the object defining the boundary (the one with the larger distance, i.e., the "inner" boundary).
    $$ d_{res} = \max(d_a, d_b) $$
    $$ m_{res} = \begin{cases} m_a & \text{if } d_a \ge d_b \\ m_b & \text{otherwise} \end{cases} $$
    [S1, unofficial]

*   **Difference (Subtraction):** Removes the right child from the left.
    $$ d_{res} = \max(d_a, -d_b) $$
    $$ m_{res} = \begin{cases} m_a & \text{if } d_a \ge -d_b \\ m_a & \text{otherwise (using negated } b \text{ distance)} \end{cases} $$
    *Note: In Difference, the material ID always comes from the left child ($m_a$) because the resulting surface is part of the left object, even if the distance is determined by the negated right object [S1, unofficial].*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Type definition: MaterialSDF is a function returning (distance: float, material_id: float)

def evaluate_material_sdf(node, point):
    """
    Recursively evaluates the CSG tree at a given point.
    node: CSG tree node (Leaf or Internal)
    point: 3D vector (x, y, z)
    Returns: (distance, material_id)
    """
    if node.is_leaf:
        # Leaf evaluation: return primitive distance and its material ID
        dist = node.sdf_function(point)
        mat_id = node.material_id
        return (dist, mat_id)
    
    else:
        # Internal node: evaluate children
        a = evaluate_material_sdf(node.left, point)
        b = evaluate_material_sdf(node.right, point)
        
        d_a, m_a = a
        d_b, m_b = b
        
        op = node.operation
        
        if op == "Union":
            # Return the closer object (min distance)
            if d_a <= d_b:
                return (d_a, m_a)
            else:
                return (d_b, m_b)
                
        elif op == "Intersection":
            # Return the object defining the inner boundary (max distance)
            if d_a >= d_b:
                return (d_a, m_a)
            else:
                return (d_b, m_b)
                
        elif op == "Difference":
            # Subtract right from left: max(d_a, -d_b)
            # Material always comes from the left object (m_a)
            if d_a >= -d_b:
                return (d_a, m_a)
            else:
                return (-d_b, m_a)
                
        else:
            raise ValueError(f"Unknown operation: {op}")

def raymarch_material_scene(origin, direction, scene_root, color_ramp_texture):
    """
    Raymarching loop using Multi-Material SDF.
    """
    t = 0.0
    max_steps = 64
    hit_material_id = None
    
    for i in range(max_steps):
        p = origin + direction * t
        dist, mat_id = evaluate_material_sdf(scene_root, p)
        
        # Hit condition
        if dist < 0.001:
            hit_material_id = mat_id
            break
            
        t += dist
        
    if hit_material_id is not None:
        # Sample color ramp using the material ID
        # Assuming color_ramp_texture maps normalized material IDs to colors
        color = sample_color_ramp(color_ramp_texture, hit_material_id)
        return color
    else:
        return background_color
```

**Parameters:**
*   **ColorRamp:** A 1D texture (sampler2D) used to map the continuous or discrete material ID (the $y$-component of the map output) to a final RGB color [S2, unofficial].
*   **Max Steps:** Standard raymarching limit (e.g., 64) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** When $d_a \approx d_b$ in Union or Intersection, floating-point precision may cause flickering between materials. [G] A small epsilon threshold or consistent tie-breaking rule (e.g., always prefer left child) is recommended.
*   **Material ID Range:** Ensure material IDs are normalized to $[0, 1]$ if using a texture lookup, or mapped to integer indices if using a texture array [S2, unofficial].
*   **Difference Material:** A common pitfall is assigning the material of the subtracted object (right child) in Difference operations. The surface belongs to the left object, so the material ID must always be $m_a$ [S1, unofficial].

## Subtopics

### [Material Object Format](1981-material-object-format.md)
The Material Object Format is a data structure that holds surface shading information, including color, physical properties, and texture descriptors, accessible from both host and device code. It distinguishes itself by explicitly separating geometric displacement (via `heightmapTex`) from shading calculations and by managing texture projection modes and normal deflection scaling.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the depth of the CSG tree, per raymarch step. The material propagation adds negligible overhead compared to distance calculation, as it involves simple comparisons and tuple assignments [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(1)$ per ray. No additional global data structures are required beyond the scene graph and the color ramp texture [S1, unofficial][S2, unofficial].
*   **Strengths:** Enables complex, multi-material scenes with sharp boolean edges without requiring multiple render passes or complex texture atlases. Shadows are straightforward to calculate using the depth buffer and sphere tracing [S1, unofficial].
*   **Limitations:** Like all CSG-based SDF methods, it suffers from the Lipschitz continuity issues described in [Object Combination Operators](1801-object-combination-operators.md) if gradients are not orthogonal, leading to ray marching artifacts. It is not suitable for smooth, organic blending of materials without additional techniques [S1, unofficial][S2, unofficial].
*   **When to use:** Ideal for hard-surface modeling, architectural visualization, and scenes with distinct, non-blending materials composed via CSG.
*   **When not to use:** When smooth material transitions or organic shapes are required; use [SDF Smooth Union](1817-sdf-smooth-union.md) or other blending techniques instead.

## Background
The technique was popularized in the context of real-time raymarching shaders by Iñigo Quilez and further detailed by Jamie Wong and others in the demoscene and indie graphics communities [S1, unofficial][S2, unofficial]. It builds upon the foundational work on Signed Distance Functions by Kevin Upson and Todd Munz, and the raymarching techniques by John Cates and others. The specific implementation of returning a tuple for material ID is a direct extension of the scalar SDF map function, allowing for efficient per-pixel material identification in fragment shaders [S1, unofficial][S2, unofficial].

## Sources
[S1] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
