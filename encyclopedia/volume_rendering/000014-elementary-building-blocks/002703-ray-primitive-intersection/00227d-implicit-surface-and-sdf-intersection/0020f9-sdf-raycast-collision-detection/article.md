# SDF Raycast Collision Detection

Also called: SDF Raycast, SDF Sign Change Check

## Overview
This method detects collisions between a ray and a terrain represented by a Signed Distance Field (SDF) by evaluating the distance at the ray's origin and end points [S1, unofficial]. It identifies collisions via sign changes in the distance values between these two endpoints [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $O$, ray direction $D$, ray distance $t_{max}$, and a distance evaluation function $E(P)$ that returns the signed distance at point $P$ [S1, unofficial].
- Output: Boolean indicating if a collision (sign change) occurred [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Steps:**
1. Normalize the direction vector $D$ [S1, unofficial].
2. Calculate the end position of the ray: $P_{end} = O + D \cdot t_{max}$ [S1, unofficial].
3. Evaluate the signed distance at the end position: $d_1 = E(P_{end})$ [S1, unofficial].
4. Evaluate the signed distance at the origin: $d_2 = E(O)$ [S1, unofficial].
5. Check for a sign change between $d_1$ and $d_2$ [S1, unofficial].
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
    if (d1 > 0 and d2 < 0) or (d1 < 0 and d2 > 0):
        return True
    
    # Step 6: Return false if no sign change
    return False
```

**Parameters:**
- `distance`: The maximum length of the ray to test [S1, unofficial].
- `evaluate_distance`: A function that queries the SDF [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-normalized Direction:** The method requires the direction vector to be normalized before calculating the end position [S1, unofficial].

## Subtopics

### [Terrain SDF Collision Detection and Resolution](3049-terrain-sdf-collision-detection-and-resolution.md)
Detects collisions by evaluating distance at an object's previous and current positions, estimates surface normals for resolution, and relies on discrete time-step checks rather than continuous collision detection.

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray, as it requires exactly two SDF evaluations [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory [S1, unofficial].

## Background
The method originates from an open-source implementation for marching cubes terrain generation [S1, unofficial].

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
