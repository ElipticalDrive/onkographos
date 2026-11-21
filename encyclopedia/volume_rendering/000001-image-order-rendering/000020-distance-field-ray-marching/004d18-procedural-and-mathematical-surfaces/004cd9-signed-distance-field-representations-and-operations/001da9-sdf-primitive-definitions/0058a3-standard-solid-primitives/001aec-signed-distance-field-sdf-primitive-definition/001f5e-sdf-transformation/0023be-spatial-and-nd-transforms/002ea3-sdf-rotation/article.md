# SDF Rotation

Also called: Rotating SDFs, SDF Rotate

## Overview
SDF Rotation applies a rotational transformation to a signed distance field primitive by rotating the sampling point before evaluating the base distance function [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Query point $p$ in world space, rotation representation (quaternion $q$ or axis-angle).
- Output: Signed distance value $d$ from the base primitive at the rotated location.

**Data Structures:**
- Rotation representation: Stores the orientation as a quaternion [S1, unofficial].
- Query point: 3D vector $p = (x, y, z)$ [S2, unofficial].

**Steps:**
1. Retrieve the stored rotation representation (quaternion $q$) [S1, unofficial].
2. Compute the inverse rotation of $q$. For a unit quaternion, the inverse is the conjugate: $q^{-1} = (w, -x, -y, -z)$ [G].
3. Apply the inverse rotation to the query point $p$ to obtain the rotated point $p'$ [S1, S2, unofficial].
   - If using a quaternion $q^{-1}$: $p' = q^{-1} \otimes (0, p) \otimes q$ (where $\otimes$ is quaternion multiplication) [G].
   - If using axis-angle: Rotate $p$ around the specified axis by the stored angle [S2, unofficial].
4. Evaluate the base SDF function using the rotated point $p'$: $d = \text{BaseSDF}(p')$ [S1, S2, unofficial].
5. Return $d$ [S2, unofficial].

**Parameters:**
- Quaternion $q$: The rotation to apply. Stored as the second field of the Rotation struct. It is inverted at query time [S1, unofficial].
- Axis and angle: Alternative representation where the axis is a 3D vector and the angle is in radians [S2, unofficial].

**Edge Cases and Pitfalls:**
- Non-unit quaternions: If the stored quaternion is not normalized, the inverse rotation will scale the point incorrectly. Normalization is required [G].
- Precision errors: Repeated rotations can accumulate floating-point errors. Renormalizing the quaternion periodically is recommended [G].
- Degenerate input: An axis of zero length in axis-angle representation is undefined [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Vec3:
    x: float
    y: float
    z: float

@dataclass
class Quaternion:
    w: float
    x: float
    y: float
    z: float

def rotate_sdf(p: Vec3, q: Quaternion, base_sdf: Callable[[Vec3], float]) -> float:
    # Step 2: Compute inverse quaternion (conjugate for unit quaternion)
    q_inv = Quaternion(q.w, -q.x, -q.y, -q.z)
    
    # Step 3: Apply inverse rotation to p
    # Convert point to quaternion (w=0)
    p_q = Quaternion(0.0, p.x, p.y, p.z)
    
    # Quaternion multiplication: q_inv * p_q * q
    temp = quaternion_multiply(q_inv, p_q)
    p_rotated_q = quaternion_multiply(temp, q)
    
    # Extract rotated point
    p_rotated = Vec3(p_rotated_q.x, p_rotated_q.y, p_rotated_q.z)
    
    # Step 4: Evaluate base SDF
    distance = base_sdf(p_rotated)
    
    # Step 5: Return distance
    return distance

def quaternion_multiply(a: Quaternion, b: Quaternion) -> Quaternion:
    """Standard Hamilton product for quaternion multiplication."""
    w = a.w * b.w - a.x * b.x - a.y * b.y - a.z * b.z
    x = a.w * b.x + a.x * b.w + a.y * b.z - a.z * b.y
    y = a.w * b.y - a.x * b.z + a.y * b.w + a.z * b.x
    z = a.w * b.z + a.x * b.y - a.y * b.x + a.z * b.w
    return Quaternion(w, x, y, z)
```

## Subtopics

### [SDF Translation](5820-sdf-translation.md)
Positions a primitive by transforming the sampling point via vector subtraction. Choose it when only placement is needed, as it avoids rotation overhead.

## Operating characteristics
- Time complexity: $O(1)$ per query, dominated by quaternion multiplication [G].
- Memory complexity: $O(1)$ additional memory for the inverse quaternion calculation [G].
- Strengths: Efficient orientation of primitives without modifying the base SDF code; supports arbitrary rotations via quaternions [G].
- Limitations: Requires careful handling of quaternion normalization; slightly more expensive than translation due to quaternion multiplication overhead [G].
- When to use: When primitives need to be oriented at arbitrary angles in the scene [G].
- When not to use: When performance is critical and only axis-aligned rotations are needed (matrix multiplication might be faster on some hardware) [G].

## Background
Not given in the sources.

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S2] Painting with Math: A Gentle Study of Raymarching, https://blog.maximeheckel.com/posts/painting-with-math-a-gentle-study-of-raymarching/, fetched 2026-10-27 (unofficial)
