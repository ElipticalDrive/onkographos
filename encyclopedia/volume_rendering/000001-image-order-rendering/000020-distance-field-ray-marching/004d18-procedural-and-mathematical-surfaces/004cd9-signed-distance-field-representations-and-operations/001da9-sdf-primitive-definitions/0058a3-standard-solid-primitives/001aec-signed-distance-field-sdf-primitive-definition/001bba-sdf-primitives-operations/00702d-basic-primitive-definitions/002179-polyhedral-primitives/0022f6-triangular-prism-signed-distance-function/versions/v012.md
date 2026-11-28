# Triangular Prism Signed Distance Function

Also called: Signed Triangle Prism, Triangular Prism SDF, sdTriPrism

## Overview
Computes the Euclidean signed distance from a query point to the surface of an equilateral triangular prism aligned with the Z-axis. It differs from general polyhedral SDFs by exploiting the specific symmetry of the equilateral triangle to reduce the 3D problem to a 2D triangle distance calculation via coordinate folding, avoiding iterative projection or complex clipping logic.

## How it works
**Inputs and Outputs:**
- Input: Query point $p = (p_x, p_y, p_z)$ and parameter vector $h = (h_x, h_y)$.
- Output: Signed distance $d$.

**Data Structures:**
- $p$: 3D vector.
- $h$: 2D vector where $h_x$ controls the triangle size and $h_y$ controls the half-height.

**Parameters:**
- $h_x$: Half the side length of the equilateral triangle base? No, see formula derivation. The formula uses $h_x \cdot 0.5$. Let's analyze the formula.
- $h_y$: Half the height of the prism.

**Formula Derivation and Steps:**
The implementation folds the point into the fundamental domain of the equilateral triangle in the XY plane.
1.  Take the absolute value of the query point: $q = |p|$. This exploits the symmetry of the prism across the XZ and YZ planes (and the triangle's symmetries).
2.  Compute the distance to the triangular cross-section. The triangle is equilateral. The standard approach for an equilateral triangle centered at the origin with a specific orientation involves projecting onto the triangle's axes.
3.  The provided source [S2, unofficial] uses a specific analytical approximation or exact formula for an equilateral triangle oriented with a flat side or vertex up?
    The formula is: $d = \max(q_z - h_y, \max(q_x \cdot 0.866025 + q_y \cdot 0.5, -q_y) - h_x \cdot 0.5)$.
    
    Let's break down the triangle part: $d_{2D} = \max(q_x \cdot \frac{\sqrt{3}}{2} + q_y \cdot \frac{1}{2}, -q_y) - h_x \cdot 0.5$.
    This looks like the distance to an equilateral triangle with a vertex pointing in the $-y$ direction (or similar).
    Actually, let's look at the standard "sdTriPrism" from Inigo Quilez or similar libraries.
    The formula `max(q.x*0.866025+p.y*0.5,-p.y)` suggests a specific orientation.
    
    Let's define the steps clearly based on [S2, unofficial]:
    1.  $q = |p|$
    2.  $d_{cap} = q_z - h_y$
    3.  $d_{side} = \max(q_x \cdot \frac{\sqrt{3}}{2} + q_y \cdot \frac{1}{2}, -q_y) - h_x \cdot \frac{1}{2}$
    4.  $d = \max(d_{cap}, d_{side})$
    
    Wait, the standard SDF for a prism combines the 2D distance and the 1D height distance using the extrusion formula mentioned in the parent article:
    $d = \min(\max(d_{2D}, d_{cap}), 0.0) + \sqrt{\max(d_{2D}, 0.0)^2 + \max(d_{cap}, 0.0)^2}$.
    
    However, the source [S2, unofficial] returns `max(q.z-h.y, max(...)-h.x*0.5)`. This is **not** the extrusion formula. This is a simpler, potentially incorrect or specific-case formula?
    Let's re-read [S2, unofficial] carefully.
    `return max(q.z-h.y,max(q.x*0.866025+p.y*0.5,-p.y)-h.x*0.5);`
    
    This formula returns the maximum of the height distance and the 2D distance. This is the definition of an $L_\infty$ norm combination, not the Euclidean distance.
    However, many SDF primitives in these libraries (like `sdBox`) use the Euclidean formula. `sdBox` uses `length(max(abs(p)-b,0.0))`.
    `sdTriPrism` in [S2, unofficial] uses `max`. This is a known simplification or error in some online repositories, OR it defines a different shape (a prism with rounded corners? No, max implies sharp corners in the combined metric).
    
    Actually, looking at [S1, unofficial], it imports `sdTriPrism` from the same library.
    Let's check if there is a more correct version. The prompt notes say: "Computes the signed distance...".
    If the source [S2, unofficial] is the primary source, I must document what it does.
    However, the parent article mentions the extrusion formula.
    The prompt says: "The core mechanism involves ... using the extrusion formula".
    But the source [S2, unofficial] does NOT use the extrusion formula.
    
    Conflict: The parent article describes a general mechanism (extrusion), but the specific source for Triangular Prism [S2, unofficial] uses a max-based combination.
    I must report what the source says. I will note that the source uses a max combination, which is an approximation or a specific variant, distinct from the exact Euclidean extrusion.
    
    Wait, let's look at the formula again.
    $d_{2D} = \max(q_x \frac{\sqrt{3}}{2} + q_y \frac{1}{2}, -q_y) - h_x \frac{1}{2}$.
    This calculates the distance to the triangle boundary.
    If $d_{2D} < 0$ (inside) and $d_{cap} < 0$ (inside), the max is negative.
    If one is positive, the max is positive.
    This does not produce the correct Euclidean distance at the edges/corners.
    
    However, I must follow the source. I will describe the method as implemented in [S2, unofficial].
    
    **Pseudocode:**
    ```python
    def sd_tri_prism(p, h):
        """
        Computes the signed distance to a triangular prism.
        
        Args:
            p: vec3, query point (x, y, z).
            h: vec2, parameters. h[0] relates to triangle size, h[1] is half-height.
            
        Returns:
            float, signed distance.
        """
        # Step 1: Fold into fundamental domain using absolute values
        q_x = abs(p.x)
        q_y = abs(p.y)
        q_z = abs(p.z)
        
        # Step 2: Calculate distance to the triangular cross-section in XY plane
        # The triangle is equilateral. The formula assumes a specific orientation.
        # Constant sqrt(3)/2 approx 0.866025
        cos_pi_6 = 0.866025
        sin_pi_6 = 0.5
        
        # Distance to the triangle edges
        # Term 1: Projection onto one axis
        term1 = q_x * cos_pi_6 + q_y * sin_pi_6
        # Term 2: Projection onto another axis
        term2 = -q_y
        
        # The 2D distance to the triangle boundary
        # Note: The source subtracts h[0] * 0.5. 
        # If h[0] is the side length, the distance from center to edge is side * sqrt(3)/6.
        # This parameterization is specific to the source.
        d_2d = max(term1, term2) - h[0] * 0.5
        
        # Step 3: Calculate distance to the caps (top/bottom)
        d_cap = q_z - h[1]
        
        # Step 4: Combine distances
        # The source uses max(d_cap, d_2d). This is an L-infinity combination, 
        # not the standard Euclidean extrusion formula.
        d = max(d_cap, d_2d)
        
        return d
    ```
    
    **Edge Cases:**
    - $h_x = 0$: Degenerate triangle (line or point).
    - $h_y = 0$: Degenerate prism (flat triangle).
    - Precision: The constant $0.866025$ is an approximation of $\frac{\sqrt{3}}{2}$. For high-precision requirements, use `sqrt(3)/2`.

## Operating characteristics
- **Time Complexity:** $O(1)$. Constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$. No auxiliary data structures.
- **Strengths:** Extremely fast evaluation; suitable for real-time ray marching.
- **Limitations:** The implementation in [S2, unofficial] uses a `max` combination rather than the Euclidean extrusion formula, which may result in incorrect distances at the prism's edges and corners (the distance field will not be perfectly smooth/Euclidean there). [S1, unofficial] and [S2, unofficial] are identical in this regard.

## Background
The Signed Distance Function for a triangular prism is a standard primitive in procedural geometry and ray tracing. The specific implementation using the coordinate folding and `max` combination is found in online GLSL SDF libraries.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] marklundin/glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
