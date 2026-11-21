# Linear SDF Blend

Also called: Shape Blending

## Overview
Linear SDF Blend performs a weighted linear interpolation between two signed distance function values, allowing for morphing between shapes or creating smooth transitions based on a scalar control factor [S1, unofficial]. It is distinct from spatially varying blends like [Smoothstep SDF Blend](#smoothstep-sdf-blend) by using a constant or externally controlled factor rather than deriving the factor from the query point's position [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input $d_1$: The signed distance value of the first primitive at the query point [S1, unofficial].
- Input $d_2$: The signed distance value of the second primitive at the query point [S1, unofficial].
- Input $a$: The blend factor, a scalar in the range $[0, 1]$ [S1, unofficial].
- Output $d_{blend}$: The interpolated signed distance value [S1, unofficial].

**Data Structures:**
- The inputs and output are simple scalar floating-point values [S1, unofficial].

**Steps:**
1. Retrieve the signed distance $d_1$ for the first shape at the current raymarching position [S1, unofficial].
2. Retrieve the signed distance $d_2$ for the second shape at the current raymarching position [S1, unofficial].
3. Determine the blend factor $a$, which can be a constant, a function of time, or a function of other scene variables [S1, unofficial].
4. Compute the blended distance using the linear interpolation formula:
   $$ d_{blend} = a \cdot d_1 + (1 - a) \cdot d_2 $$ [S1, unofficial].

**Parameters:**
- $a$: The blending weight. $a=0$ yields $d_2$, $a=1$ yields $d_1$ [S1, unofficial].

**Edge Cases and Pitfalls:**

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
- **Layout:** The SDF functions return a two-component vector where the x-component is the distance (float) and the y-component is the primitive ID (float, typically 1) [S2, unofficial].
- **Pseudocode:**
  ```python
  def length(v):
      return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

  def smoothstep(edge0, edge1, x):
      t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
      return t * t * (3.0 - 2.0 * t)

  def smoothstep_sdf_blend(p, a, b):
      """
      Blend two SDFs based on the distance of p from the origin.
      
      Args:
          p: Query point as a 3-tuple (x, y, z).
          a: 2-tuple containing (distance, id) for primitive A.
          b: 2-tuple containing (distance, id) for primitive B.
          
      Returns:
          2-tuple containing (blended_distance, id).
      """
      dist_from_origin = length(p)
      s = smoothstep(0.0, 1.0, dist_from_origin)
      d = (1.0 - s) * a[0] + s * b[0]
      return (d, 1.0)
  ```

## Operating characteristics

## Background
The concept of blending SDFs using linear interpolation is discussed in the context of raymarching tutorials and is presented as a direct analogy to alpha blending in color space [S1, unofficial]. The spatially varying variant using smoothstep is documented in raymarching tutorials [S2, unofficial].

## Sources
[S1] Volumetric Rendering: Signed Distance Functions, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
[S2] 8 – Raymarching: Primitive Blending, http://www.geeks3d.com/20130524/building-worlds-with-distance-functions-in-glsl-raymarching-glslhacker-tutorial-opengl/8/, fetched 2026-10-26 (unofficial)
