# SDF Raycast Collision Detection

Also called: SDF Raycast, SDF Sign Change Check

## Overview
This method detects collisions between a ray and a terrain represented by a Signed Distance Field (SDF) by evaluating the distance at the ray's origin and end points [S1, unofficial]. It identifies collisions via sign changes in the distance values, indicating the ray has crossed the zero-level set (the surface) [S1, unofficial]. It distinguishes itself from continuous collision detection by relying on discrete time-step checks, which is simpler to implement but prone to tunneling at high velocities [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $O$, ray direction $D$ (assumed normalized), ray distance $t_{max}$, and a distance evaluation function $E(P)$ that returns the signed distance at point $P$.
- Output: Boolean indicating if a collision (sign change) occurred.

**Data Structures:**
- No complex data structures are required beyond the SDF evaluation function. The method operates on scalar distance values.

**Steps:**
1. Normalize the direction vector $D$ if it is not already normalized [S1, unofficial].
2. Calculate the end position of the ray: $P_{end} = O + D \cdot t_{max}$ [S1, unofficial].
3. Evaluate the signed distance at the end position: $d_1 = E(P_{end})$ [S1, unofficial].
4. Evaluate the signed distance at the origin: $d_2 = E(O)$ [S1, unofficial].
5. Check for a sign change between $d_1$ and $d_2$. A collision is detected if one value is positive and the other is negative [S1, unofficial].
6. Return `true` if a sign change is detected, `false` otherwise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_raycast_collision(origin, direction, distance, evaluate_distance):
    # Step 1: Normalize direction
    direction = normalize(direction)
    
    # Step 2: Calculate end position
    end_position = origin + (direction * distance)
    
    # Step 3: Evaluate distance at end position
    d1 = evaluate_distance(end_position)
    
    # Step 4: Evaluate distance at origin
    d2 = evaluate_distance(origin)
    
    # Step 5: Check for sign change
    # Collision if signs are different (one positive, one negative)
    if (d1 > 0 and d2 < 0) or (d1 < 0 and d2 > 0):
        return True
    
    # Step 6: Return false if no sign change
    return False
```

**Parameters:**
- `distance`: The maximum length of the ray to test. Typical values depend on the scene scale and physics timestep.
- `evaluate_distance`: A function or method that queries the SDF. Implementation details vary by SDF representation (e.g., voxel grid, procedural function).

**Edge Cases and Pitfalls:**
- **Tunneling:** At high velocities or large timesteps, the ray may pass completely through a thin surface without the endpoints straddling the zero-level set, resulting in a missed collision [S1, unofficial].
- **Zero Distance:** If the ray starts or ends exactly on the surface ($d=0$), the sign change check might fail depending on implementation. A robust implementation should handle $d=0$ as a collision or use a small epsilon.
- **Non-normalized Direction:** If the direction vector is not normalized, the calculated end position will be incorrect relative to the intended distance.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray, as it requires exactly two SDF evaluations.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Extremely fast and simple to implement. No iterative search or complex data structure traversal is needed.
- **Limitations:** Prone to tunneling at high speeds or with thin geometry. Does not provide the exact intersection point or normal, only a boolean collision flag.
- **When to Use:** Suitable for simple collision detection in games or simulations where performance is critical and objects move slowly relative to the SDF resolution.
- **When Not to Use:** Unsuitable for continuous collision detection, high-speed physics, or when precise intersection points are required.

## Background
This method is a basic technique used in terrain physics and simple collision detection systems involving SDFs. It is often found in game development tutorials and open-source projects dealing with voxel-based terrain [S1, unofficial]. It serves as a foundational concept for more advanced SDF-based collision detection methods.

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
