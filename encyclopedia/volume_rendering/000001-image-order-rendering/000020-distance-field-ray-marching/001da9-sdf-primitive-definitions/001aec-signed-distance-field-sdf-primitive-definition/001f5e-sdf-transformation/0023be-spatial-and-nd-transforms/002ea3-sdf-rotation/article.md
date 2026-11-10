# SDF Rotation

Also called: SDF Rotate
## Overview
SDF Rotation transforms a query point by applying the inverse of a stored quaternion before querying the base SDF [S1, unofficial]. It differs from [SDF Translation](3106-spatial-and-nd-transforms.md) by using a quaternion-based rotation rather than a vector offset [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: a query point $P$ in world space and a stored rotation quaternion $Q$ [S1, unofficial].
- Output: the distance value returned by the base SDF evaluated at the rotated point [S1, unofficial].

**Data Structures:**
The rotation is stored as the second field of a `Rotation` struct [S1, unofficial].

**Steps:**
1. Let $Q$ be the stored rotation quaternion [S1, unofficial].
2. Compute the inverse quaternion $Q^{-1}$ [S1, unofficial].
3. For a unit quaternion, $Q^{-1} = Q^*$ (the conjugate) [G].
4. Rotate the query point $P$ by $Q^{-1}$ to obtain the local point $P_{local}$ [S1, unofficial].
5. The rotation is applied as $P_{local} = Q^{-1} \otimes P \otimes Q$, where $\otimes$ denotes quaternion multiplication and $P$ is treated as a pure quaternion $(0, P_x, P_y, P_z)$ [G].
6. Pass $P_{local}$ to the base SDF function to retrieve the distance [S1, unofficial].

**Parameters:**
- $Q$ (Quaternion): The rotation to apply [S1, unofficial].
- It is stored as the second field of the `Rotation` struct and inverted at query time [S1, unofficial].

**Edge Cases and Pitfalls:**
- Non-unit quaternions require normalization before inversion or division by the squared norm [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rotate_and_query(query_point, stored_quaternion, base_sdf):
    # 1. Compute inverse quaternion
    w, x, y, z = stored_quaternion
    # For unit quaternions, inverse is conjugate. Non-unit handling not given in sources. [G]
    inv_w, inv_x, inv_y, inv_z = w, -x, -y, -z
    norm_sq = inv_w**2 + inv_x**2 + inv_y**2 + inv_z**2
    inv_w /= norm_sq; inv_x /= norm_sq; inv_y /= norm_sq; inv_z /= norm_sq
    
    # 2. Rotate query point by inverse quaternion
    # P_local = Q_inv * P * Q
    px, py, pz = query_point
    def qmul(a, b):
        return (a[0]*b[0] - a[1]*b[1] - a[2]*b[2] - a[3]*b[3],
                a[0]*b[1] + a[1]*b[0] + a[2]*b[3] - a[3]*b[2],
                a[0]*b[2] - a[1]*b[3] + a[2]*b[0] + a[3]*b[1],
                a[0]*b[3] + a[1]*b[2] - a[2]*b[1] + a[3]*b[0])
    
    p_quat = (0.0, px, py, pz)
    rotated = qmul((inv_w, inv_x, inv_y, inv_z), p_quat)
    rotated = qmul(rotated, (w, x, y, z))
    
    local_point = (rotated[1], rotated[2], rotated[3])
    
    # 3. Query base SDF
    return base_sdf.distance_from(local_point)
```
## Subtopics

### SDF Translation
Translates the query point by a 3D vector before querying the base SDF [S1, unofficial]. It subtracts the stored translation vector from the query point and passes the result to the base SDF's distance_from function [S1, unofficial]. The parameter is a `Vec3` representing the 3D offset, stored as the second field of the `Translation` struct [S1, unofficial].

## Operating characteristics
Time and memory complexity are Not given in the sources.
## Background
The method is implemented in the `unfinishedprogram/ray-marcher` repository using a quaternion stored in a struct and inverted at query time [S1, unofficial].
## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
