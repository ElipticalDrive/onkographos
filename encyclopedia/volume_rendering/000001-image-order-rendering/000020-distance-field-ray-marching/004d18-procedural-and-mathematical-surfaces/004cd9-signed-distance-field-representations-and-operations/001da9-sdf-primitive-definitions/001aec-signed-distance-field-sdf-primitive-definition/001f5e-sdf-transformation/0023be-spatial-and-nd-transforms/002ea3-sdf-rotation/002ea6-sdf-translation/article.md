# SDF Translation

Also called: Moving SDFs, Positioning SDFs, SDF Move

## Overview
SDF Translation positions a signed distance field primitive at a specific location in world space by transforming the query point inversely before evaluating the base primitive. Unlike [SDF Rotation](5817-sdf-rotation.md), which applies an inverse rotation matrix to the query point, translation uses a simple vector subtraction. This method allows for the static definition of primitive shapes (e.g., a unit sphere) while dynamically placing them within a scene without modifying the primitive's internal geometry.

## How it works
**Inputs:**
- `p`: The query point in world space (vec3).
- `t`: The translation vector representing the desired position of the SDF object in world space (vec3) [S1, S2, unofficial].
- `base_sdf`: The function evaluating the untransformed primitive (e.g., `sdSphere`, `sdBox`) [S1, S2, unofficial].

**Outputs:**
- `d`: The signed distance from `p` to the translated primitive surface [S1, S2, unofficial].

**Data Structures:**
The translation vector `t` is typically stored as a uniform variable or a field within a scene descriptor struct. In the reference implementation, it is part of a `Translation` struct, stored as the second field [S1, unofficial].

**Steps:**
1. Compute the inverse-transformed query point `p'` by subtracting the translation vector `t` from the world-space query point `p`:
   $$ p' = p - t $$
2. Evaluate the base SDF function at `p'`:
   $$ d = \text{base\_sdf}(p') $$
3. Return `d`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def translate_sdf(p: Vec3, t: Vec3, base_sdf: Callable[[Vec3], float]) -> float:
    """
    Translates an SDF primitive by vector t.
    
    Args:
        p: World-space query point.
        t: Desired position of the primitive's origin in world space.
        base_sdf: Function evaluating the untransformed primitive distance.
        
    Returns:
        Signed distance from p to the translated primitive.
    """
    # Inverse transform: move the sampling point opposite to the object's movement
    p_transformed = p - t
    
    # Evaluate base primitive at transformed point
    distance = base_sdf(p_transformed)
    
    return distance
```

**Parameters:**
- `t` (Vec3): The 3D offset vector defining the object's position. Typical values depend on scene scale; for a unit sphere centered at $(0, 1, 0)$, $t = (0, 1, 0)$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Intuition Trap:** A common error is adding the translation vector to the query point ($p + t$) or modifying the primitive's internal coordinates directly. The correct approach is moving the sampling point in the *opposite* direction of the desired object movement [S2, unofficial]. If you move the object right, you must move the sampling point left to maintain the same relative distance [S2, unofficial].
- **Precision:** Standard floating-point precision is sufficient for most scene scales. However, for very large translation vectors, precision loss in the subtraction $p - t$ may affect distance accuracy near the surface [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, as it involves only one vector subtraction and one call to the base SDF.
- **Memory Complexity:** $O(1)$, requiring storage for the translation vector `t` [S1, unofficial].
- **Strengths:** Computationally inexpensive; allows for easy animation by updating `t` per frame; composable with other transforms (rotation, scale) via function composition.
- **Limitations:** None specific to translation itself; limitations are inherited from the base SDF (e.g., smoothness, analytic continuity).

## Background
SDF translation is a fundamental technique in procedural raymarching, popularized by blogs and tutorials on the subject. It relies on the principle that transforming the query point inversely is equivalent to transforming the object in the forward direction. This concept is often introduced alongside rotation and scaling in introductory raymarching guides [S2, unofficial]. The implementation details, such as storing translation vectors in uniform buffers for GPU rendering, are demonstrated in open-source raymarcher projects [S1, unofficial].

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S2] Painting with Math: A Gentle Study of Raymarching, https://blog.maximeheckel.com/posts/painting-with-math-a-gentle-study-of-raymarching/, fetched 2026-10-27 (unofficial)
