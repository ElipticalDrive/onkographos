# Transparent Surface Picking

Also called: WYSIWYP Picking for Transparent Surfaces

## Overview
Transparent Surface Picking is a method for picking transparent polygonal surfaces analogous to WYSIWYP for DVR, by tracing a ray, accumulating opacities of hit polygons, and selecting the intersection with the largest opacity contribution [S1]. It differs from first-hit picking by evaluating the compositing contribution of each intersection rather than stopping at the nearest surface [S1].

## How it works
**Inputs:** A viewing ray and a set of transparent polygonal surfaces [S1].
**Outputs:** The intersection point of the ray with the polygon that has the largest opacity contribution [S1].
**Data Structures:** Not given in the sources.

**Steps:**
1. Trace the viewing ray through the scene to find all intersections with transparent polygons [S1].
2. Accumulate or composite the opacities of the different polygons hit by the ray using standard front-to-back compositing [G].
3. Select the intersection of the ray with the polygon that has the largest opacity contribution after compositing [S1].

**Formulas:**
Standard front-to-back compositing updates the accumulated opacity $\alpha_{acc}$ and computes the contribution $\Delta \alpha_j$ of each polygon $p_j$ [G]:
$$ \Delta \alpha_j = (1 - \alpha_{acc}) \cdot \alpha_j $$
$$ \alpha_{acc} \leftarrow \alpha_{acc} + \Delta \alpha_j $$
where $\alpha_j$ is the intrinsic opacity of polygon $p_j$ at the intersection point, and $\alpha_{acc}$ is initialized to $0$ [G].

**Parameters:** Not given in the sources.

**Edge cases and pitfalls:** Computing derivatives of accumulated opacity is not necessary because the intervals are intersections with surfaces, which are infinitesimally small [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transparent_surface_picking(ray_origin, ray_direction, polygons):
    hits = []
    for poly in polygons:
        t, point, normal = intersect_ray_polygon(ray_origin, ray_direction, poly)
        if t is not None and t > 0:
            alpha_poly = poly.get_opacity_at(point)
            hits.append({
                't': t,
                'point': point,
                'normal': normal,
                'alpha_poly': alpha_poly,
                'poly_id': poly.id
            })
    hits.sort(key=lambda h: h['t'])
    if not hits:
        return None
    acc_alpha = 0.0
    max_contrib = -1.0
    best_hit = None
    for hit in hits:
        contrib = (1.0 - acc_alpha) * hit['alpha_poly']
        if contrib > max_contrib:
            max_contrib = contrib
            best_hit = hit
        acc_alpha += contrib
        if acc_alpha >= 1.0:
            break
    return best_hit

def intersect_ray_polygon(origin, direction, poly):
    # Standard ray-polygon intersection (e.g., Möller–Trumbore for triangles) [G]
    # Returns (t, point, normal) or (None, None, None)
    pass
```

## Subtopics

### Opacity Threshold Picking
A volume picking technique that searches along the viewing ray using the compositing scheme and stops when accumulated opacity exceeds a user-defined threshold, assuming a surface exists at that location [S1]. It is used in VTK, ParaView, MeVisLab, and Voreen (first-hit) [S1]. Pitfalls include: using a zero threshold returns positions in regions surrounding features in "foggy" renderings because every position in the volume has non-zero opacity; using a non-zero threshold may miss relatively transparent but visible regions, causing the ray to reach the end of the dataset and pick the exit position instead of the visible feature [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Aligns with user perception of "what is seen" [S1].
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.

## Background
The method is analogous to the WYSIWYP framework for DVR [S1]. Gobbetti et al. introduced the opacity threshold volume picking technique, which searches along the ray using the compositing scheme and stops when accumulated opacity exceeds a threshold [S1]. The transparent surface picking method extends this concept to polygonal surfaces by selecting the intersection with the largest opacity contribution [S1].

## Sources
[S1] WYSIWYP: What You See Is What You Pick, https://doi.org/10.1109/tvcg.2012.292, fetched 2026-11-05
