# Transparent Surface Picking

Also called: WYSIWYP Picking for Transparent Surfaces

## Overview
Transparent Surface Picking is a method for selecting 3D positions on transparent polygonal surfaces within a scene, analogous to the "What You See Is What You Pick" (WYSIWYP) principle used in Direct Volume Rendering (DVR) [S1]. It solves the problem of accurately identifying which specific transparent geometry a user intends to interact with when multiple surfaces overlap along a viewing ray. Unlike standard opaque surface picking, which simply selects the first hit, this method traces a ray, accumulates the opacity contributions of all intersected polygons, and selects the intersection corresponding to the polygon that contributes the most to the final accumulated opacity [S1].

## How it works
**Inputs:**
- A viewing ray $r(t) = \mathbf{o} + t\mathbf{d}$ defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- A set of transparent polygonal surfaces $P = \{p_1, p_2, \dots, p_n\}$ in the scene.
- For each polygon $p_i$, its geometric intersection distance $t_i$ with the ray (if any) and its intrinsic opacity $\alpha_i$ (or opacity contribution derived from shading/transparency models).

**Outputs:**
- The 3D position $\mathbf{p}_{pick}$ on the surface of the selected polygon.
- The identifier of the selected polygon $p_{best}$.

**Data Structures:**
- A list of hit records, each containing:
  - `t`: float, distance from ray origin to intersection.
  - `alpha_contrib`: float, the opacity contribution of this polygon to the ray's color accumulation.
  - `point`: vec3, the 3D intersection point.
  - `polygon_id`: int, identifier of the polygon.

**Steps:**
1. **Ray Traversal:** Trace the viewing ray through the scene to find all intersections with transparent polygons. Let the sorted list of intersections be $I = \{i_1, i_2, \dots, i_k\}$ where $t(i_1) < t(i_2) < \dots < t(i_k)$.
2. **Opacity Accumulation:** Iterate through the intersections from front to back (increasing $t$). For each intersection $i_j$, compute the accumulated opacity $\alpha_{acc}$ up to that point. The standard front-to-back compositing equation is used:
   $$ \alpha_{acc}(t_j) = \alpha_{acc}(t_{j-1}) + (1 - \alpha_{acc}(t_{j-1})) \cdot \alpha_j $$
   where $\alpha_j$ is the opacity of polygon $p_j$ at the intersection point, and $\alpha_{acc}(t_0) = 0$.
3. **Contribution Calculation:** Calculate the specific opacity contribution $\Delta \alpha_j$ of each polygon $p_j$ to the final image. This is the amount of color/opacity added by $p_j$ that is not already covered by previous polygons:
   $$ \Delta \alpha_j = (1 - \alpha_{acc}(t_{j-1})) \cdot \alpha_j $$
   Note: $\alpha_{acc}(t_j) = \alpha_{acc}(t_{j-1}) + \Delta \alpha_j$.
4. **Selection:** Identify the polygon $p_{best}$ that has the maximum $\Delta \alpha_{best}$ among all intersections.
   $$ p_{best} = \arg \max_{j} (\Delta \alpha_j) $$
5. **Return:** Return the intersection point and normal of $p_{best}$.

**Pitfalls:**
- **Derivatives:** Computing derivatives of the accumulated opacity $\alpha_{acc}$ is not necessary for transparent surfaces because the intervals of integration are the intersections with surfaces, which are infinitesimally small [S1].
- **Zero Opacity:** Polygons with zero opacity do not contribute and can be ignored.
- **Numerical Precision:** When $\alpha_j$ is very small, floating-point precision may affect the accuracy of $\Delta \alpha_j$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transparent_surface_picking(ray_origin, ray_direction, polygons):
    # Step 1: Find all intersections
    hits = []
    for poly in polygons:
        t, point, normal = ray_intersect_polygon(ray_origin, ray_direction, poly)
        if t is not None and t > 0:
            # Compute opacity contribution of this polygon
            # alpha_poly is the intrinsic opacity of the polygon at this point
            alpha_poly = poly.get_opacity_at(point)
            hits.append({
                't': t,
                'point': point,
                'normal': normal,
                'alpha_poly': alpha_poly,
                'poly_id': poly.id
            })
    
    # Sort hits by distance (front to back)
    hits.sort(key=lambda h: h['t'])
    
    if not hits:
        return None

    # Step 2 & 3: Accumulate opacity and find max contribution
    acc_alpha = 0.0
    max_contrib = -1.0
    best_hit = None
    
    for hit in hits:
        # Contribution is the amount of new opacity added by this polygon
        # Formula: (1 - accumulated_opacity_before) * polygon_opacity
        contrib = (1.0 - acc_alpha) * hit['alpha_poly']
        
        if contrib > max_contrib:
            max_contrib = contrib
            best_hit = hit
            
        # Update accumulated opacity
        acc_alpha = acc_alpha + contrib
        
        # Optimization: if acc_alpha reaches 1.0, no further polygons contribute
        if acc_alpha >= 1.0:
            break
            
    return best_hit
```

## Subtopics

### Opacity Threshold Picking
A volume picking technique that searches along the viewing ray using the compositing scheme and stops when the accumulated opacity exceeds a user-defined threshold, assuming a surface exists at that location [S1]. It is simpler than Transparent Surface Picking as it returns the first position where opacity exceeds the threshold rather than the position of maximum contribution. It is used in VTK, ParaView, MeVisLab, and Voreen (first-hit) [S1]. Pitfalls include: using a zero threshold (first-hit) returns positions in regions surrounding features in "foggy" renderings because every position in the volume has non-zero opacity; using a non-zero threshold may miss relatively transparent but visible regions, causing the ray to reach the end of the dataset and pick the exit position instead of the visible feature [S1].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ or $O(N)$ depending on intersection finding, where $N$ is the number of polygons intersected by the ray. Sorting hits by distance is required.
- **Memory Complexity:** $O(K)$ where $K$ is the number of intersections along the ray.
- **Strengths:** Accurately reflects the visual importance of surfaces; aligns with user perception of "what is seen" [S1].
- **Limitations:** Requires access to polygon opacity and compositing logic; more computationally expensive than simple first-hit picking.
- **When to use:** When precise selection of transparent, overlapping geometry is required, and the visual contribution of the surface is the primary criterion for selection.

## Background
The method was introduced by Gobbetti et al. as part of the WYSIWYP (What You See Is What You Pick) framework for volume rendering and transparent surfaces [S1]. It extends the concept of opacity-based selection from DVR to polygonal surfaces, addressing the limitations of first-hit picking in transparent scenes. The technique is particularly relevant in medical visualization and scientific data exploration where transparency is used to reveal internal structures.

## Sources
[S1] WYSIWYP: What You See Is What You Pick, https://doi.org/10.1109/tvcg.2012.292, fetched 2026-11-05
