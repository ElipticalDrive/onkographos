# Line Capsule Primitive
Also called: Geometry Primitives and Mesh Shading, Mesh Lighting and Materials

## Overview
The Line Capsule Primitive is a screen-space rendering technique for visualizing line-based volumetric data, specifically designed to replace exact volumetric primitives (such as the closed-form convolution of a segment with a 3D Gaussian) with a computationally cheaper alternative that maintains visual fidelity [S1, unofficial]. It solves the performance bottleneck of rendering high-density line structures by approximating the 3D Gaussian footprint with a 2D rotationally symmetric profile of the point-to-segment distance in screen space [S1, unofficial]. It is distinguished from sibling methods like [EWA Splatting](1069-ewa-splatting.md) by its specific focus on line segments rather than point clouds, and from [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by its use of a simplified 2D geometric profile instead of full 3D covariance matrices for line primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of line segments defined by endpoints in 3D space, associated with a Gaussian-like density profile.
*   **Output:** A 2D image where lines are rendered as capsules with smooth, rotationally symmetric intensity falloff.

**Data Structures:**
The primitive is defined by the screen-space projection of the line segment. The core data required for the fragment shader includes:
1.  **Segment Endpoints:** The 2D screen coordinates of the segment endpoints, $\mathbf{e}_1$ and $\mathbf{e}_2$.
2.  **Width Parameter:** The effective on-screen width of the line, $w$, which determines the radius of the capsule.
3.  **Joints:** The endpoints themselves serve as the joints where adjacent capsules connect.

**Numbered Steps:**

1.  **Projection:** Transform the 3D line segment endpoints into screen space using the Model-View-Projection (MVP) matrix. Let the resulting 2D coordinates be $\mathbf{e}_1 = (x_1, y_1)$ and $\mathbf{e}_2 = (x_2, y_2)$ [S1, unofficial].

2.  **Distance Calculation:** For each pixel fragment at screen position $\mathbf{p} = (x, y)$, calculate the perpendicular distance $d$ from $\mathbf{p}$ to the line segment defined by $\mathbf{e}_1$ and $\mathbf{e}_2$. This is the 2D point-to-segment distance [S1, unofficial].
    *   Let $\mathbf{v} = \mathbf{e}_2 - \mathbf{e}_1$ be the segment vector.
    *   Let $\mathbf{w} = \mathbf{p} - \mathbf{e}_1$ be the vector from the first endpoint to the fragment.
    *   Compute the projection parameter $t$:
        $$ t = \frac{\mathbf{w} \cdot \mathbf{v}}{\mathbf{v} \cdot \mathbf{v}} $$
    *   Clamp $t$ to the range $[0, 1]$ to ensure the distance is calculated to the segment, not the infinite line:
        $$ t_{clamped} = \max(0, \min(1, t)) $$
    *   Calculate the closest point on the segment $\mathbf{c}$:
        $$ \mathbf{c} = \mathbf{e}_1 + t_{clamped} \mathbf{v} $$
    *   Calculate the distance $d$:
        $$ d = \| \mathbf{p} - \mathbf{c} \| $$

3.  **Intensity Profile:** Compute the intensity $I$ based on the distance $d$ and the line width $w$. The profile is rotationally symmetric around the segment. A common analytic radial profile for Gaussian-like splats is used:
    $$ I = \exp\left(-\frac{d^2}{2\sigma^2}\right) $$
    where $\sigma$ is related to the width $w$ (typically $w \approx 2\sqrt{2\ln 2}\sigma$ or similar, depending on the specific kernel definition). The source specifies a "rotationally symmetric profile of 2D point-to-segment distance" [S1, unofficial].

4.  **Joint Composition (Deficit Rule):** At the joints (endpoints $\mathbf{e}_1$ and $\mathbf{e}_2$), adjacent capsules meet. To prevent double brightness (overlapping contributions) or gaps (underlapping contributions), a deficit rule is applied. This ensures that the sum of intensities from adjacent segments at the joint equals the intensity of a single continuous segment. The exact mathematical formulation of the deficit rule is not provided in the sources, but it operates by adjusting the intensity contribution at the endpoints based on the angle between connected segments [S1, unofficial].

5.  **Size Policy Selection:** A per-node size policy determines whether to use the Line Capsule Primitive or a screen-space Quad Primitive.
    *   Calculate the **Effective Segment Load**:
        $$ \text{Load} = N_{segments} \times W_{widest} $$
        where $N_{segments}$ is the authored segment count and $W_{widest}$ is the on-screen width of the widest segment at the current framing [S1, unofficial].
    *   If $\text{Load} \geq 2,000,000$, use the screen-space Quad Primitive instead.
    *   Otherwise, use the Line Capsule Primitive [S1, unofficial].
    *   This policy can be pinned to either primitive if desired [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_line_capsule_fragment(p, e1, e2, width):
    """
    Calculate intensity for a line capsule fragment.
    
    Args:
        p: (x, y) screen position of the fragment.
        e1: (x, y) screen position of the first endpoint.
        e2: (x, y) screen position of the second endpoint.
        width: Effective on-screen width of the line (radius parameter).
        
    Returns:
        float: Intensity value [0, 1].
    """
    # Vector from e1 to e2
    vx = e2[0] - e1[0]
    vy = e2[1] - e1[1]
    
    # Vector from e1 to p
    wx = p[0] - e1[0]
    wy = p[1] - e1[1]
    
    # Dot products
    dot_vv = vx * vx + vy * vy
    dot_wv = wx * vx + wy * vy
    
    # Avoid division by zero for zero-length segments
    if dot_vv == 0:
        # Segment is a point, distance is to e1
        dx = p[0] - e1[0]
        dy = p[1] - e1[1]
        dist = (dx*dx + dy*dy)**0.5
    else:
        # Projection parameter t
        t = dot_wv / dot_vv
        
        # Clamp t to [0, 1]
        if t < 0.0:
            t = 0.0
        elif t > 1.0:
            t = 1.0
            
        # Closest point c on segment
        cx = e1[0] + t * vx
        cy = e1[1] + t * vy
        
        # Distance from p to c
        dx = p[0] - cx
        dy = p[1] - cy
        dist = (dx*dx + dy*dy)**0.5
        
    # Intensity profile (Gaussian-like)
    # sigma is derived from width. Assuming width corresponds to 2*sigma or similar.
    # The source does not specify the exact mapping, but uses a rotationally symmetric profile.
    # Using a standard Gaussian falloff:
    sigma = width / 2.0  # Placeholder for exact sigma-width relationship
    if sigma == 0:
        return 0.0
        
    intensity = exp(-(dist * dist) / (2 * sigma * sigma))
    
    return intensity
```

**Edge Cases and Pitfalls:**
*   **Near-Axial Case:** The capsule approximation performs well even in the near-axial case (when the line is nearly parallel to the view ray), matching or beating the exact volumetric primitive in visual quality [S1, unofficial].
*   **Joint Gaps/Double Brightness:** Without the deficit rule, corners in line networks will either appear brighter (due to overlapping Gaussian tails) or have gaps (due to insufficient coverage). The deficit rule must be implemented to handle arbitrary angles between connected segments [S1, unofficial].
*   **Zero-Length Segments:** The distance calculation must handle segments where $\mathbf{e}_1 = \mathbf{e}_2$ to avoid division by zero [G].
*   **Size Policy Threshold:** The threshold of 2 million effective segments is a hard cutoff. Below this, the capsule is used; above it, the quad is used. This threshold is fixed in the implementation [S1, unofficial].

## Subtopics

### Screen-Space Quad Primitive
A cheaper alternative used when the effective segment load exceeds 2 million. It is less accurate than the capsule but faster. [Link](1156-ray-driven-splatting.md) (Note: The quad is a sibling primitive in this context, not a separate method article, but its performance characteristics are discussed here).

### Exact Volumetric Primitive
The closed-form convolution of a segment with a 3D Gaussian. It is visually comparable to the capsule but 2.7–5x more expensive per frame. [Link](1069-ewa-splatting.md) (Note: This is the method the capsule replaces).

## Operating characteristics
*   **Time Complexity:** The capsule primitive is significantly faster than the exact volumetric primitive. It costs approximately 2.7–5x less than the screen-space quad per frame, and the quad itself is only 1.04–1.11x faster than the capsule on Apple GPUs, and ~1.5x faster on thin lines on discrete NVIDIA GPUs [S1, unofficial].
*   **Memory Complexity:** The capsule primitive requires minimal memory overhead, storing only the segment endpoints and width. The exact volumetric primitive requires more complex calculations but no additional memory storage for the primitive itself [S1, unofficial].
*   **Performance:** The capsule primitive is preferred for most cases due to its balance of visual quality and performance. The screen-space quad is used only for very high loads (>= 2 million effective segments) [S1, unofficial].
*   **Strengths:** High performance, good visual quality (matching exact primitive in blinded tests), simple implementation.
*   **Limitations:** Approximation error in the near-axial case (though visually acceptable), requires careful handling of joints to avoid artifacts.
*   **When to Use:** Use the capsule primitive for line rendering when the effective segment load is below 2 million. Use the quad primitive when the load exceeds this threshold. Avoid the exact volumetric primitive unless visual fidelity is paramount and performance is not a concern [S1, unofficial].

## Background
The Line Capsule Primitive was introduced in the Luxar viewer for Gaussian splatting of microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. It was developed to address the performance limitations of exact volumetric primitives for line-based data, providing a practical solution for real-time interaction with large datasets [S1, unofficial]. The method is part of the broader "Geometry Primitives and Mesh Shading" feature set in Luxar [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
