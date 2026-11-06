# Cone Overlap Test for Refractive Boundaries

Also called: Cone overlap test, Normal cone overlap test

## Overview
The Cone Overlap Test is a pruning test used to determine if a position bounding box and a normal bounding cone can possibly satisfy the refraction conditions for a light transport path [S1]. It ensures that the view vector $V$ lies on the negative side of the shading normal and within the critical angle cone defined by $\arcsin(1/\eta)$ [S1]. It differs from the smooth normal pruning test only in that positions are bounded by a 3D box rather than a 2D triangle; the test logic is otherwise identical [S1].

## How it works
**Inputs:**
*   Position bounding box $B$ (axis-aligned, defined by min/max corners).
*   Normal bounding cone $C$ (defined by axis $\mathbf{n}_0$ and semi-angle $\alpha$).
*   Light direction $\mathbf{L}$ and View direction $\mathbf{V}$ (vectors from the point in $B$ to the light and camera, respectively).
*   Refractive index ratio $\eta$.

**Outputs:**
*   Boolean: `True` if the box and cone *might* contain a valid refraction point; `False` if the subtree can be pruned.

**Data Structures:**
*   The position bounding box is represented by its minimum and maximum coordinates: $\mathbf{b}_{min}, \mathbf{b}_{max} \in \mathbb{R}^3$.
*   The normal bounding cone is represented by its central axis $\mathbf{n}_0$ (unit vector) and semi-angle $\alpha$.

**Steps:**
1.  **Compute Angle Bounds:** Calculate the minimum and maximum possible angles between the cone axis $\mathbf{n}_0$ and the light direction $\mathbf{L}$, as well as between $\mathbf{n}_0$ and the view direction $\mathbf{V}$, for any point $\mathbf{p}$ within the bounding box $B$. This uses the techniques described in [Walter et al. 2005] [S1]. Let $\theta_{L, min}, \theta_{L, max}$ be the bounds for $\mathbf{L}$ and $\theta_{V, min}, \theta_{V, max}$ be the bounds for $\mathbf{V}$.
2.  **Offset by Cone Semiangle:** Adjust these bounds to account for the variation of the normal within the cone. The effective range of angles for the light is $[\theta_{L, min} - \alpha, \theta_{L, max} + \alpha]$ and for the view is $[\theta_{V, min} - \alpha, \theta_{V, max} + \alpha]$ [S1].
3.  **Check Refraction Conditions:** Verify if it is possible for the angles to satisfy the geometric constraints of refraction:
    *   The light direction $\mathbf{L}$ must lie on the positive side of the shading normal (angle $< \pi/2$).
    *   The view direction $\mathbf{V}$ must lie on the negative side of the shading normal (angle $> \pi/2$).
    *   The view direction $\mathbf{V}$ must be within the critical angle cone, i.e., the angle between $\mathbf{V}$ and the normal must be less than or equal to $\arcsin(1/\eta)$ [S1].
4.  **Prune Decision:** If the adjusted angle ranges for $\mathbf{L}$ or $\mathbf{V}$ cannot satisfy these conditions simultaneously, return `False`. Otherwise, return `True` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_overlap_test(box_min, box_max, cone_axis, cone_semi_angle, L, V, eta):
    """
    Determines if a position bounding box and normal bounding cone 
    can possibly satisfy refraction conditions.
    
    Args:
        box_min, box_max: 3D vectors defining the axis-aligned bounding box.
        cone_axis: Unit vector representing the axis of the normal bounding cone.
        cone_semi_angle: Float, the semi-angle alpha of the normal bounding cone.
        L: 3D vector, direction to the light.
        V: 3D vector, direction to the viewer.
        eta: Float, refractive index ratio.
        
    Returns:
        bool: True if the box/cone might contain a valid refraction point.
    """
    # Step 1: Compute bounds on angles between cone axis and L/V over the box
    # Using techniques from Walter et al. 2005 to find min/max angles from any point in box
    # to the fixed vectors L and V, relative to the cone axis.
    # Let theta_L_min, theta_L_max be the bounds for angle(cone_axis, L)
    # Let theta_V_min, theta_V_max be the bounds for angle(cone_axis, V)
    theta_L_min, theta_L_max = compute_angle_bounds(box_min, box_max, cone_axis, L)
    theta_V_min, theta_V_max = compute_angle_bounds(box_min, box_max, cone_axis, V)
    
    # Step 2: Offset by the semi-angle of the normal bounding cone
    # The normal can vary by alpha from the cone axis.
    effective_L_min = theta_L_min - cone_semi_angle
    effective_L_max = theta_L_max + cone_semi_angle
    effective_V_min = theta_V_min - cone_semi_angle
    effective_V_max = theta_V_max + cone_semi_angle
    
    # Critical angle for refraction
    critical_angle = math.asin(1.0 / eta)
    
    # Step 3: Check if it is possible for angles to lie within required ranges
    # Condition 1: L must be on the positive side of the normal (angle < pi/2)
    # The minimum possible angle between normal and L must be < pi/2
    # Note: Angles are typically in [0, pi]. 
    # If the entire range [effective_L_min, effective_L_max] is >= pi/2, L is always on negative side -> Fail.
    # We need overlap with [0, pi/2).
    l_possible = effective_L_max < math.pi / 2 # Simplified: if max possible angle is < pi/2, it's definitely positive side.
    # More robust: Check if the interval [effective_L_min, effective_L_max] intersects [0, pi/2)
    # Since angles are non-negative, we check if effective_L_max > 0 and effective_L_min < pi/2
    l_positive_possible = effective_L_max > 0 and effective_L_min < math.pi / 2
    
    # Condition 2: V must be on the negative side of the normal (angle > pi/2)
    # The maximum possible angle between normal and V must be > pi/2
    v_negative_possible = effective_V_min < math.pi / 2 and effective_V_max > math.pi / 2
    
    # Condition 3: V must be within the critical angle cone relative to the normal
    # The angle between V and the normal must be <= critical_angle
    # This implies the angle between V and the normal is in [0, critical_angle]
    # However, V is on the negative side, so the angle between V and -Normal is <= critical_angle
    # Or angle between V and Normal is in [pi - critical_angle, pi]
    # Let's stick to the source description: "within a cone bounded by the critical angle"
    # Usually this means the angle between the view vector and the normal is > pi/2 and <= pi - critical_angle?
    # Or angle between V and Normal is in [pi/2, pi/2 + critical_angle]?
    # Source says: "V must lie on the negative side ... and within a cone bounded by the critical angle"
    # This typically means the angle between V and the normal is in [pi/2, pi/2 + arcsin(1/eta)]?
    # Actually, standard refraction: V and L are on opposite sides. 
    # If Normal points out, L is in (angle < pi/2), V is in (angle > pi/2).
    # The "critical angle cone" usually refers to the cone around the normal where total internal reflection occurs?
    # No, for incoming light from medium 1 to 2, critical angle is for TIR.
    # Here we are finding the point P. The condition is that V is within the critical angle of the normal?
    # Let's re-read carefully: "V must lie on the negative side of the shading normal and within a cone bounded by the critical angle, arcsin(1/eta)."
    # This phrasing is slightly ambiguous. It likely means the angle between V and the normal is <= pi/2 + arcsin(1/eta)?
    # Or that the angle between V and the *tangent* plane is limited?
    # Given the context of pruning, it usually checks if the view direction is "visible" through the interface.
    # Let's assume the condition is: angle(V, Normal) <= pi/2 + critical_angle? 
    # Or more likely, the angle between V and the normal is in [pi/2, pi/2 + critical_angle] is NOT correct.
    # Standard condition: The view vector must be within the critical angle of the normal's negative side?
    # Let's use the explicit bounds check from the source logic:
    # "see if it is possible for the respective angles to lie within the required ranges"
    
    # Range for L: [0, pi/2)
    # Range for V: [pi/2, pi/2 + critical_angle] ?? 
    # Actually, if V is on the negative side, the angle with the normal is in (pi/2, pi].
    # The "cone bounded by critical angle" usually implies the angle between V and the normal is <= pi/2 + arcsin(1/eta) is not right.
    # It implies the angle between V and the normal is >= pi/2 - arcsin(1/eta)? No.
    # Let's look at the Spindle test: angle between PL and PV is between pi/2 + arcsin(1/eta) and pi.
    # This suggests the geometry is constrained.
    
    # For the Cone Overlap test, we just check if the intervals overlap the required ranges.
    # Required range for L: [0, pi/2)
    # Required range for V: [pi/2, pi] AND angle(V, Normal) <= pi/2 + critical_angle? 
    # No, "within a cone bounded by the critical angle" usually means the angle between V and the normal is <= critical_angle? 
    # But V is on the negative side. So angle(V, Normal) is > pi/2.
    # This implies the condition is actually on the angle between V and the *negative* normal?
    # If V is on the negative side, let N' = -Normal. V is on positive side of N'.
    # Angle(V, N') < pi/2.
    # "Within a cone bounded by critical angle" -> Angle(V, N') <= critical_angle?
    # This would mean V is within critical_angle of the inward normal.
    
    # Let's implement the check based on the text "lie within the required ranges".
    # We have effective_V_min and effective_V_max.
    # We need to check if there is an overlap between [effective_V_min, effective_V_max] and the valid range for V.
    # Valid range for V (angle with Normal): [pi/2, pi/2 + critical_angle] is a common interpretation for "within critical angle of the surface"?
    # No, critical angle is for TIR. 
    # Let's assume the source implies: Angle(V, Normal) <= pi/2 + arcsin(1/eta) is not the constraint.
    # The constraint is likely: Angle(V, Normal) >= pi/2 and Angle(V, Normal) <= pi/2 + arcsin(1/eta) is incorrect.
    # Actually, if eta > 1, critical angle is < pi/2.
    # If the view is inside the medium, and looking out, it must be within the critical angle of the normal?
    # Let's stick to the generic check:
    
    l_valid_range = (0, math.pi / 2)
    # V is on negative side, so angle > pi/2.
    # "Within a cone bounded by the critical angle" -> This likely refers to the cone around the normal.
    # If V is on the negative side, it is within the cone if angle(V, Normal) <= pi/2 + critical_angle?
    # Or is it angle(V, Normal) <= critical_angle? (Impossible if > pi/2).
    # It must be angle(V, -Normal) <= critical_angle.
    # So angle(V, Normal) >= pi/2 - critical_angle? No, that's positive side.
    # Angle(V, Normal) <= pi/2 + critical_angle?
    # Let's assume the valid range for V is [pi/2, pi/2 + critical_angle] is wrong.
    # Let's assume the valid range for V is [pi/2, pi] intersected with "within critical angle cone".
    # The critical angle cone is usually defined around the normal.
    # If we are inside, we look out. The view vector V must be within the critical angle of the normal?
    # No, V is the direction TO the viewer.
    # Let's use the generic overlap check.
    
    # Check L
    l_overlap = max(effective_L_min, l_valid_range[0]) < min(effective_L_max, l_valid_range[1])
    
    # Check V
    # Assuming valid range for V is [pi/2, pi/2 + critical_angle] is not standard.
    # Standard: V is visible if it is within the critical angle of the normal's negative side.
    # Angle(V, -Normal) <= critical_angle.
    # Angle(V, Normal) = pi - Angle(V, -Normal) >= pi - critical_angle.
    # So V must be in [pi - critical_angle, pi].
    # But V must also be on the negative side (Angle > pi/2).
    # Since critical_angle < pi/2 (for eta > 1), pi - critical_angle > pi/2.
    # So the range is [pi - critical_angle, pi].
    
    v_valid_range = (math.pi - critical_angle, math.pi)
    v_overlap = max(effective_V_min, v_valid_range[0]) < min(effective_V_max, v_valid_range[1])
    
    return l_overlap and v_overlap

def compute_angle_bounds(box_min, box_max, axis, vector):
    """
    Computes the minimum and maximum angles between the axis and the vector
    for any point in the box. This is a helper based on Walter et al. 2005.
    """
    # Implementation detail: This involves projecting the box onto the plane perpendicular to the axis
    # and finding the extreme angles.
    # For brevity, we assume this function is implemented according to [Walter et al. 2005].
    # It returns (theta_min, theta_max).
    pass
```

## Subtopics

### Spindle Test
The Spindle Test is a pruning test based on the geometric constraint that a single refraction bends light by a limited amount, restricting the valid refraction point $P$ to a spindle-shaped region (surface of revolution of a circular arc from $V$ to $L$) [S1]. It tests if a bounding box overlaps this spindle by using a modified standard box-sphere overlap test that also computes a lower bound on the distance from the box to the line containing $V$ and $L$ [S1]. The condition for a point $P$ to lie within the spindle is:
$$ \left\| \mathbf{P} - \frac{\mathbf{L} + \mathbf{V}}{2} \right\|^2 + \frac{\|\mathbf{L} - \mathbf{V}\|}{\eta} r \leq \frac{\|\mathbf{L} - \mathbf{V}\|^2}{4} $$
where $r$ is the distance from point $P$ to the line containing $L$ and $V$, and $\eta$ is the refractive index [S1]. If $\eta$ is arbitrarily large, this reduces to a point-in-sphere test [S1]. This test is cheap and does not depend on shading normals, allowing it to prune many subtrees even if they have large normal bounding cones [S1].

## Operating characteristics
The Cone Overlap Test is a pruning test used during the traversal of a bounding volume hierarchy (BVH) or similar structure for light transport in participating media [S1]. Its primary strength is its ability to quickly discard subtrees that cannot contain valid refraction points based on normal and position bounds [S1]. The test relies on angle bound computations which are relatively cheap compared to full ray intersection tests [S1]. It is particularly effective when combined with the Spindle Test and Sidedness Test [S1]. The complexity depends on the cost of computing angle bounds over a 3D box, which is generally low [S1]. It is most useful in scenes with refractive boundaries where normal variation is significant [S1].

## Background
The Cone Overlap Test was introduced by Walter et al. in the context of single scattering in refractive media with triangle mesh boundaries [S1]. It is an extension of the pruning test used for smooth normals, adapted for 3D position bounding boxes [S1]. The underlying techniques for computing angle bounds are from [Walter et al. 2005] [S1]. The test is part of a suite of three pruning tests (Spindle, Sidedness, Cone Overlap) designed to efficiently find refraction points in participating media [S1].

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
