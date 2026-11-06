# Linear SDF Blend

Also called: Shape Blending

## Overview
Linear SDF Blend is a method for interpolating between two signed distance functions (SDFs) to create a smooth morphing transition between shapes, rather than a boolean union or intersection. It solves the problem of creating organic transitions or time-varying geometry where the surface topology changes gradually. Unlike [SDF Smooth Union](1817-sdf-smooth-union.md) which creates a rounded union, Linear SDF Blend produces a linear interpolation of the distance values themselves. It is distinct from [Smooth Intersection](1839-smooth-intersection.md) and [Multi-Material Distance Fields](1943-multi-material-distance-fields.md) in that it does not attempt to preserve the Lipschitz continuity property (gradient magnitude $\le 1$) of the resulting field, making it suitable for visual morphing but potentially less robust for strict ray marching convergence if the blend factor varies spatially without care [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input 1: $d_1$, the signed distance from the query point to the first primitive.
- Input 2: $d_2$, the signed distance from the query point to the second primitive.
- Input 3: $a$, the blend factor, a scalar in the range $[0, 1]$.
- Output: $d_{blend}$, the interpolated signed distance.

**Data Structures:**
The operation is element-wise on scalar distances. If material IDs are carried (as in [Multi-Material Distance Fields](1943-multi-material-distance-fields.md)), the blend factor may also determine material selection, but the core distance calculation remains a scalar linear interpolation [G].

**Formula:**
$$d_{blend} = a \cdot d_1 + (1 - a) \cdot d_2$$

Where:
- $d_1$ and $d_2$ are the distances of the two shapes.
- $a \in [0, 1]$ is the blend factor.

**Parameters:**
- $a$: The blend factor. Typical usage involves a constant value for static blending or a time-varying value (e.g., derived from $\sin(time)$) for animation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_sdf_blend(d1: float, d2: float, a: float) -> float:
    """
    Linearly interpolate between two SDF values.
    
    Args:
        d1: Distance to the first primitive.
        d2: Distance to the second primitive.
        a: Blend factor in [0, 1].
        
    Returns:
        The blended distance.
    """
    return a * d1 + (1.0 - a) * d2
```

**Edge Cases and Pitfalls:**
- **Lipschitz Continuity Loss:** As noted in the parent [Object Combination Operators](1801-object-combination-operators.md), this operator does not preserve Lipschitz continuity. If the gradients of $d_1$ and $d_2$ are not orthogonal, the gradient magnitude of the result can exceed 1, leading to ray marching artifacts (overshooting or jittering) [S1, unofficial][S2, unofficial].
- **Parallel Surfaces:** If the surfaces are parallel and close, the blend can produce unnatural thickness or gaps [S1, unofficial][S2, unofficial].
- **Gradient Deviation:** Lipschitz continuity is lost if the gradients deviate by more than $30^\circ$ [S1, unofficial][S2, unofficial].

## Subtopics

### Smoothstep SDF Blend
A variant where the blend factor $s$ is not a constant but is determined by the spatial position of the query point, specifically its distance from the origin. This creates a spatially varying blend region.
- **Definition:** Blends two SDFs using a smoothstep interpolation factor based on the length of the query point $p$ [S2, unofficial].
- **Formula:**
  $$s = \text{smoothstep}(0.0, 1.0, \|p\|)$$
  $$d = (1 - s) \cdot d_1 + s \cdot d_2$$
  Where $\text{smoothstep}(edge0, edge1, x)$ clamps $x$ to $[edge0, edge1]$ and performs cubic Hermite interpolation [S2, unofficial].
- **Parameters:** The smoothstep range is fixed between 0.0 and 1.0, defining the spherical shell region where the blend occurs [S2, unofficial].
- **Data Layout:** The function typically returns a `vec2` where the x-component is the distance and the y-component is a primitive identifier (e.g., 1) [S2, unofficial].
- **Pseudocode:**
  ```python
  def smoothstep_sdf_blend(p: Vector3, d1: float, d2: float) -> tuple:
      """
      Blend two SDFs based on the distance of the query point from the origin.
      
      Args:
          p: Query point.
          d1: Distance to the first primitive (e.g., rounded box).
          d2: Distance to the second primitive (e.g., torus).
          
      Returns:
          Tuple of (blended_distance, primitive_id).
      """
      # Calculate spatial blend factor
      dist_from_origin = length(p)
      s = smoothstep(0.0, 1.0, dist_from_origin)
      
      # Interpolate distances
      blended_d = (1.0 - s) * d1 + s * d2
      
      # Return distance and a primitive ID (1)
      return (blended_d, 1.0)
  ```
- **When to choose:** Use when you want the transition between shapes to depend on the viewer's or object's position relative to a center, creating a "fading" effect between geometries [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point. It involves two multiplications and one addition.
- **Memory Complexity:** $O(1)$. No additional data structures are required beyond the input distances and blend factor.
- **Performance:** Extremely fast, suitable for real-time ray marching on the GPU.
- **Strengths:** Simple to implement; allows for smooth morphing animations; works with any pair of SDFs.
- **Limitations:** Does not preserve the Lipschitz continuity property, meaning it is not a true distance field in the mathematical sense after blending. This can cause ray marching to fail or require smaller step sizes if the blend factor varies rapidly in space [S1, unofficial][S2, unofficial].
- **When to use:** For visual effects, morphing animations, or when the exact distance metric is less critical than the visual smoothness of the transition.
- **When not to use:** When strict collision detection or robust ray marching convergence is required, or when the gradients of the two shapes are nearly parallel [S1, unofficial][S2, unofficial].

## Background
The concept of blending SDFs using linear interpolation was popularized in the context of GLSL raymarching tutorials. Alan Zucconi introduced the term "Shape Blending" and demonstrated its use in morphing between a sphere and a cube using a time-varying blend factor [S1, unofficial]. Geeks3D further explored spatial blending techniques using smoothstep functions to blend primitives like rounded boxes and tori based on their distance from the origin [S2, unofficial]. These techniques emerged as alternatives to boolean operators (min/max) to create more organic and less angular geometry in real-time graphics.

## Sources
[S1] Volumetric Rendering: Signed Distance Functions, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
[S2] 8 – Raymarching: Primitive Blending, http://www.geeks3d.com/20130524/building-worlds-with-distance-functions-in-glsl-raymarching-glslhacker-tutorial-opengl/8/, fetched 2026-10-26 (unofficial)
