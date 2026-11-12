# Linear SDF Blend

Also called: Shape Blending

## Overview
Linear SDF Blend performs a weighted linear interpolation between two signed distance function values, allowing for morphing between shapes or creating smooth transitions based on a scalar control factor [S1, unofficial]. Unlike [SDF Smooth Union](1817-sdf-smooth-union.md) which creates a rounded union edge, or [Smooth Intersection](1839-smooth-intersection.md) which creates a rounded intersection, Linear Blend does not inherently produce a geometrically correct distance field for the combined shape; rather, it produces a visual blend or a morph [S1, unofficial]. It is distinct from spatially varying blends like [Smoothstep SDF Blend](#smoothstep-sdf-blend) by using a constant or externally controlled factor rather than deriving the factor from the query point's position.

## How it works
**Inputs and Outputs:**
- Input $d_1$: The signed distance value of the first primitive at the query point.
- Input $d_2$: The signed distance value of the second primitive at the query point.
- Input $a$: The blend factor, a scalar in the range $[0, 1]$.
- Output $d_{blend}$: The interpolated signed distance value.

**Data Structures:**
- The inputs and output are simple scalar floating-point values. No complex structures are required for the basic operation.

**Steps:**
1. Retrieve the signed distance $d_1$ for the first shape at the current raymarching position.
2. Retrieve the signed distance $d_2$ for the second shape at the current raymarching position.
3. Determine the blend factor $a$. This can be a constant, a function of time, or a function of other scene variables, but it is not derived from the spatial distance to the origin in the basic definition [S1, unofficial].
4. Compute the blended distance using the linear interpolation formula:
   $$ d_{blend} = a \cdot d_1 + (1 - a) \cdot d_2 $$

**Parameters:**
- $a$: The blending weight. $a=0$ yields $d_2$, $a=1$ yields $d_1$. Typical values depend on the desired visual effect or morphing state [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Lipschitz Continuity:** As with all combination operators, the resulting function preserves the Lipschitz condition (gradient magnitude $\le 1$) only if the gradients of the two input SDFs are orthogonal. If the angle between gradients deviates significantly (e.g., $>30^\circ$) or surfaces are parallel, the blended result may violate the Lipschitz condition, leading to ray marching artifacts such as overshooting or jittering [S1, S2, unofficial].
- **Geometric Accuracy:** The result is not a true distance field for the boolean combination of the shapes. It is a visual blend. Using this for precise collision detection or accurate raymarching of the combined geometry is generally incorrect unless the specific visual artifact is acceptable [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_sdf_blend(d1: float, d2: float, a: float) -> float:
    """
    Perform linear interpolation between two SDF values.
    
    Args:
        d1: Distance of the first primitive.
        d2: Distance of the second primitive.
        a: Blend factor in [0, 1].
        
    Returns:
        The blended distance value.
    """
    # Ensure a is clamped to [0, 1] if not guaranteed by caller
    a = max(0.0, min(1.0, a))
    return a * d1 + (1.0 - a) * d2
```

## Subtopics

### Smoothstep SDF Blend
A variant where the blend factor is derived spatially from the query point's distance to the origin using a smoothstep function [S2, unofficial].
- **Definition:** Blends between two SDFs based on the distance of the query point from the origin, creating a spatially varying transition zone [S2, unofficial].
- **Formula:**
  $$ s = \text{smoothstep}(0.0, 1.0, \|p\|) $$
  $$ d = (1 - s) \cdot a_x + s \cdot b_x $$
  where $p$ is the query point, $\|p\|$ is its length, and $a_x, b_x$ are the distance components of the input SDFs [S2, unofficial].
- **Parameters:** The smoothstep range is fixed between 0.0 and 1.0, defining the radial region where the blend occurs [S2, unofficial].
- **Layout:** The SDF functions return a `vec2` where the x-component is the distance (float) and the y-component is the primitive ID (float, typically 1) [S2, unofficial].
- **Pseudocode:**
  ```python
  def smoothstep_sdf_blend(p: Vector3, a: Vector2, b: Vector2) -> Vector2:
      """
      Blend two SDFs based on the distance of p from the origin.
      
      Args:
          p: Query point.
          a: vec2 containing (distance, id) for primitive A.
          b: vec2 containing (distance, id) for primitive B.
          
      Returns:
          vec2 containing (blended_distance, id).
      """
      # Calculate spatial blend factor based on distance from origin
      dist_from_origin = length(p)
      s = smoothstep(0.0, 1.0, dist_from_origin)
      
      # Interpolate distances
      d = (1.0 - s) * a.x + s * b.x
      
      # Return blended distance and primitive ID (assuming both are ID 1 or similar)
      return Vector2(d, 1.0)
  ```

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, as it involves only basic arithmetic operations.
- **Memory Complexity:** $O(1)$, requiring no additional storage beyond the input values.
- **Strengths:** Extremely fast; simple to implement; useful for morphing animations or visual blending where geometric precision is not critical.
- **Limitations:** Does not produce a valid distance field for the combined shape; prone to ray marching artifacts if the Lipschitz condition is violated; not suitable for precise boolean operations or collision detection [S1, S2, unofficial].
- **When to use:** Visual effects, morphing shapes, or when a simple visual transition is desired without the complexity of smooth boolean operators.
- **When not to use:** When a geometrically correct distance field for the union/intersection is required, or when high-precision ray marching is needed.

## Background
The concept of blending SDFs using linear interpolation is discussed in the context of volumetric rendering and raymarching tutorials. It is presented as a direct analogy to alpha blending in color space [S1, unofficial]. The technique is attributed to Alan Zucconi in his work on Signed Distance Functions [S1, unofficial]. The spatially varying variant using smoothstep is documented in raymarching tutorials by Geeks3D [S2, unofficial].

## Sources
[S1] Volumetric Rendering: Signed Distance Functions, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
[S2] 8 – Raymarching: Primitive Blending, http://www.geeks3d.com/20130524/building-worlds-with-distance-functions-in-glsl-raymarching-glslhacker-tutorial-opengl/8/, fetched 2026-10-26 (unofficial)
