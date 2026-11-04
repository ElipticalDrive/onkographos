# Multi-Material Distance Fields

Also called: Augmented SDF signature, Color-Coded SDFs, Material-Augmented Signed Distance Function, SDF with Material Data, SDF with material ID

## Overview
Multi-Material Distance Fields extend standard Signed Distance Functions (SDFs) to return both a geometric distance and a material identifier, enabling per-object material assignment within Constructive Solid Geometry (CSG) scenes [S1, unofficial][S2, unofficial]. Unlike sibling operators like [SDF Smooth Union](1817-sdf-smooth-union.md) which focus on geometric blending, this method preserves the sharp boolean logic of CSG while propagating shading data, allowing the renderer to determine surface color based on the closest object's material ID rather than just its depth [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A position vector $p$ in world space [S1, unofficial].
*   **Output:** A pair $(d, m)$, where $d$ is the signed distance to the nearest surface and $m$ is the material identifier (e.g., an integer index or a normalized color value) associated with that surface [S1, unofficial][S2, unofficial].

**Data Structures:**
The scene is represented as a CSG tree [S1, unofficial]. Each leaf node contains:
1.  An SDF function $SDF_i(p)$ returning a scalar distance [S1, unofficial].
2.  A material identifier $M_i$ [S1, unofficial].

In shader implementations, the output is typically a `float2` (or `vec2`) [S2, unofficial]:
*   `x` component: Distance $d$ [S2, unofficial].
*   `y` component: Material data $m$ (e.g., color index or direct color value) [S2, unofficial].

**Algorithm:**
The evaluation is recursive [S1, unofficial]. For a leaf node, the SDF is evaluated directly [S1, unofficial]. For internal nodes (Union, Intersection, Difference), the results of the left and right children are combined [S1, unofficial]. The combination logic compares the distance components ($d$) to determine the closest surface, then selects the corresponding material component ($m$) from the winning child [S1, unofficial].

1.  **Leaf Evaluation:**
    For a leaf node with SDF function $f$ and material ID $m_{leaf}$:
    $$ (d, m) = (f(p), m_{leaf}) $$ [S1, unofficial].

2.  **Union Operation:**
    Given left result $(d_a, m_a)$ and right result $(d_b, m_b)$:
    *   If $d_a \le d_b$: Return $(d_a, m_a)$ [S1, unofficial].
    *   Else: Return $(d_b, m_b)$ [S1, unofficial].
    *   *Logic:* The union takes the minimum distance [S1, unofficial]. The material of the closest object is preserved [S1, unofficial][S2, unofficial].

3.  **Intersection Operation:**
    Given left result $(d_a, m_a)$ and right result $(d_b, m_b)$:
    *   If $d_a \ge d_b$: Return $(d_a, m_a)$ [S1, unofficial].
    *   Else: Return $(d_b, m_a)$ [S1, unofficial].
    *   *Logic:* The intersection takes the maximum distance (deepest inside both) [S1, unofficial]. The material is taken from the left child ($m_a$) if the left child is the limiting factor ($d_a \ge d_b$), otherwise it takes the right child's distance but *still* uses the left child's material [S1, unofficial]. *Note: Standard CSG intersection usually implies the material of the object being intersected into, but the provided source explicitly assigns $m_a$ in the else case* [S1, unofficial].

4.  **Difference Operation:**
    Given left result $(d_a, m_a)$ and right result $(d_b, m_b)$:
    *   If $d_a \ge -d_b$: Return $(d_a, m_a)$ [S1, unofficial].
    *   Else: Return $(-d_b, m_a)$ [S1, unofficial].
    *   *Logic:* Difference is $A \setminus B$ [S1, unofficial]. If the left object is closer to the surface (or outside the right object), keep left material [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_material_sdf(tree: dict, p: tuple) -> tuple:
    if tree['type'] == 'leaf':
        sdf_func = tree['sdf_func']
        material = tree['material']
        distance = sdf_func(p)
        return (distance, material)
    
    left_tree = tree['left']
    right_tree = tree['right']
    operation = tree['op']
    
    d_a, m_a = evaluate_material_sdf(left_tree, p)
    d_b, m_b = evaluate_material_sdf(right_tree, p)
    
    if operation == 'union':
        if d_a <= d_b:
            return (d_a, m_a)
        else:
            return (d_b, m_b)
    elif operation == 'intersection':
        if d_a >= d_b:
            return (d_a, m_a)
        else:
            return (d_b, m_a)
    elif operation == 'difference':
        if d_a >= -d_b:
            return (d_a, m_a)
        else:
            return (-d_b, m_a)
    else:
        raise ValueError(f"Unknown operation: {operation}")

def raymarch_material_scene(scene_tree: dict, ray_origin: tuple, ray_direction: tuple, color_ramp: list) -> tuple:
    max_steps = 64
    t = 0.0
    hit_material = None
    
    for i in range(max_steps):
        p = (
            ray_origin[0] + ray_direction[0] * t,
            ray_origin[1] + ray_direction[1] * t,
            ray_origin[2] + ray_direction[2] * t
        )
        
        d, m = evaluate_material_sdf(scene_tree, p)
        
        if d < 0.001:
            hit_material = m
            break
            
        t += d
        
    if hit_material is not None:
        if isinstance(hit_material, (int, float)) and hit_material < len(color_ramp):
            color = color_ramp[hit_material]
        else:
            color = hit_material if isinstance(hit_material, tuple) else (1.0, 1.0, 1.0)
        return (*color, 1.0)
    else:
        return (0.0, 0.0, 0.0, 1.0)
```

**Parameters:**
*   **ColorRamp:** A texture or array mapping material IDs (the $y$-component of the map output) to colors [S2, unofficial].
*   **Max Steps:** Typically 64 or more, depending on scene complexity and required precision [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Comparing floating-point distances $d_a$ and $d_b$ for equality can be unstable. Using a small epsilon for the hit condition ($d < 0.001$) is standard [S2, unofficial].
*   **Intersection/Difference Material Logic:** The specific material assignment in Intersection and Difference operations (always taking $m_a$ from the left child in the source code) is a design choice [S1, unofficial].

## Subtopics

### [Material Object Format](1981-material-object-format.md)
A data structure that holds surface shading information, including color, physical properties, and texture descriptors, accessible from both host and device code. It distinguishes itself by explicitly separating geometric displacement (via `heightmapTex`) from shading calculations and by managing texture projection modes and normal deflection scaling.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Enables complex multi-material scenes with CSG operations without requiring multiple render passes or complex shader branching for material selection. Shadows are straightforward to calculate using the depth buffer and the same SDF [S1, unofficial].
*   **Limitations:** Material boundaries are sharp (discrete). The material assignment logic in Intersection/Difference is fixed by the tree structure (left-child dominance in the provided implementation), which may not match all artistic requirements [S1, unofficial].

## Background
The technique was popularized in the context of real-time raymarching shaders, notably by Jamie Wong and Iñigo Quilez [S2, unofficial]. It allows for the creation of complex, multi-colored geometric scenes using simple SDF primitives and CSG operations [S2, unofficial]. The approach is detailed in tutorials on raymarching signed distance functions [S1, unofficial][S2, unofficial].

## Sources
[S1] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
