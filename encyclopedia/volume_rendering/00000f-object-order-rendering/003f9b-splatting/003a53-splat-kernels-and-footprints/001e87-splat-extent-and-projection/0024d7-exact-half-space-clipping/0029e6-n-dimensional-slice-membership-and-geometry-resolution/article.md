# N-Dimensional Slice Membership and Geometry Resolution
Also called: Hidden Dimension Attenuation, nD Slab Clipping
## Overview
N-Dimensional Slice Membership and Geometry Resolution renders $N$-dimensional data by intersecting primitives with a 3D display hyperplane, allowing users to navigate non-displayed dimensions via sliders or toggles [S1, unofficial]. It solves the problem of visualizing higher-dimensional spatial data by computing the geometric intersection and attribute attenuation of primitives relative to the slice, differing from standard 3D rendering by handling the hidden dimensions explicitly rather than ignoring them [S1, unofficial].
## How it works
The method processes different geometry types by determining their membership in the display hyperplane or their intersection with the $N$-dimensional slab defined by the slice thickness [S1, unofficial].

**Inputs:**
- Primitive geometry type: point, Gaussian splat, line segment, or triangle [S1, unofficial].
- Display hyperplane definition: normal vector $\mathbf{n}$ and distance $d_0$ from origin [G].
- Primitive parameters: center $\mathbf{c}$, radius $r$, covariance $\Sigma$, vertices $\mathbf{v}_i$ [S1, unofficial].

**Outputs:**
- Clipped or attenuated primitive geometry ready for rasterization [S1, unofficial].
- Zero or null if the primitive is completely outside the slice [S1, unofficial].

**Data Structures:**
- Points: Center $\mathbf{c} \in \mathbb{R}^N$, radius $r$ [S1, unofficial].
- Gaussian Splats: Center $\mathbf{c} \in \mathbb{R}^N$, covariance matrix $\Sigma \in \mathbb{R}^{N \times N}$, opacity $\alpha$ [S1, unofficial].
- Line Segments: Endpoints $\mathbf{p}_1, \mathbf{p}_2 \in \mathbb{R}^N$, attributes $\mathbf{a}_1, \mathbf{a}_2$ [S1, unofficial].
- Triangles: Vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3 \in \mathbb{R}^N$ [S1, unofficial].

**Steps:**

1. **Point Clouds:** Calculate the Euclidean distance $d$ from the point center $\mathbf{c}$ to the display hyperplane [S1, unofficial]. If $d > r$, discard the point [G]. Otherwise, compute the effective radius $r_{eff} = \sqrt{r^2 - d^2}$ and render the point with this radius [S1, unofficial].
2. **Gaussian Splats:** Compute the Mahalanobis distance from the splat center $\mathbf{c}$ to the display hyperplane. Attenuate the splat's opacity and scale based on the marginal density along the hidden dimensions, modeled as a shifted Gaussian in the Mahalanobis distance [S1, unofficial].
3. **Line Segments:** Clip the line segment against the $N$-dimensional slab defined by the slice. Determine the intersection parameters where the line enters and exits the slab. If no intersection exists, discard the segment. Otherwise, interpolate every attribute at the clip parameter to form the clipped segment [S1, unofficial].
4. **Triangles:** Perform a slab membership test for each of the three vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$. If all three vertices pass the test (i.e., are within the slab), draw the triangle. If any vertex fails, discard the triangle entirely, as this method acts as a whole-triangle cull instead of performing $N$-dimensional polygon clipping [S1, unofficial].

**Parameters:**
- Slice thickness: Not given in the sources.
- Reference nominal primitive size: 4 px for points and splats, 1.5 px for lines [S1, unofficial].

**Edge Cases:**
- $d = r$: Effective radius becomes 0.
- Degenerate triangles: Handled by standard rasterization [G].
- Precision: Floating point errors in distance calculations may cause incorrect culling [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nD_slice_membership(primitive, hyperplane_normal, hyperplane_dist, slice_thickness):
    # Assume hyperplane_normal is unit length [G]
    def dist_to_hyperplane(p):
        return abs(sum(p[i] * hyperplane_normal[i] for i in range(len(p))) - hyperplane_dist)

    if primitive['type'] == 'point':
        c = primitive['center']
        r = primitive['radius']
        d = dist_to_hyperplane(c)
        if d > r:
            return None
        r_eff = (r**2 - d**2)**0.5
        return {'type': 'point', 'center': c, 'radius': r_eff}

    elif primitive['type'] == 'splat':
        c = primitive['center']
        alpha = primitive['opacity']
        # Attenuate by marginal density along hidden dimensions: shifted Gaussian in Mahalanobis distance from splat centre to slice [G]
        d = dist_to_hyperplane(c)
        attenuation = exp(-0.5 * d**2)
        return {'type': 'splat', 'center': c, 'opacity': alpha * attenuation}

    elif primitive['type'] == 'line':
        p1 = primitive['p1']
        p2 = primitive['p2']
        attrs = primitive['attrs']
        # Project endpoints onto the hyperplane normal to find slab intersection parameters
        val1 = sum(p1[i] * hyperplane_normal[i] for i in range(len(p1))) - hyperplane_dist
        val2 = sum(p2[i] * hyperplane_normal[i] for i in range(len(p2))) - hyperplane_dist
        half_thick = slice_thickness / 2.0
        t_min, t_max = 0.0, 1.0
        if abs(val2 - val1) > 1e-9:
            t1 = (-half_thick - val1) / (val2 - val1)
            t2 = (half_thick - val1) / (val2 - val1)
            if t1 > t2: t1, t2 = t2, t1
            t_min = max(t_min, t1)
            t_max = min(t_max, t2)
        if t_min > t_max:
            return None
        # Interpolate geometry and attributes at clip parameters
        new_p1 = [p1[i] + t_min * (p2[i] - p1[i]) for i in range(len(p1))]
        new_p2 = [p1[i] + t_max * (p2[i] - p1[i]) for i in range(len(p1))]
        # Interpolate each attribute channel linearly between endpoints [G]
        new_attrs = [attrs[i] + t_min * (attrs[i] - attrs[i]) for i in range(len(attrs))]
        return {'type': 'line', 'p1': new_p1, 'p2': new_p2, 'attrs': new_attrs}

    elif primitive['type'] == 'triangle':
        v1, v2, v3 = primitive['v1'], primitive['v2'], primitive['v3']
        def in_slab(v):
            return dist_to_hyperplane(v) <= slice_thickness / 2.0
        if in_slab(v1) and in_slab(v2) and in_slab(v3):
            return primitive
        else:
            return None
```
## Subtopics

### [4D Block Geometry Slicing](5091-4d-block-geometry-slicing.md)
Renders a 3D cross-section of a 4D voxel world by intersecting 4D primitives with a 3D hyperplane. It differs by treating blocks as 4D primitives (4-balls or 4-boxes) and generating accurate polygonal meshes for the resulting 3D volumes, allowing shape changes (e.g., rectangle to hexagon) as the hyperplane moves.

## Operating characteristics
- **Time Complexity:** Depends on primitive type. Point and splat tests are $O(N)$. Line clipping is $O(N)$. Triangle culling is $O(1)$ per triangle [G].
- **Memory Complexity:** $O(1)$ per primitive [G].
- **Performance:** Frame rate is held at the cost of sharpness. Measurement window is 3 s at reference nominal primitive size (4 px for points, 1.5 px for lines) [S1, unofficial].
- **Strengths:** Allows interactive navigation of $N$-dimensional data. Simple culling for triangles avoids complex polygon clipping [S1, unofficial].
- **Limitations:** Triangle culling may lose detail if triangles are partially inside the slab. Sharpness is reduced for points near the edge of the slice [S1, unofficial].
- **When to use:** Interactive visualization of multidimensional scientific data where navigation speed is critical [S1, unofficial].
- **When not to use:** When high-fidelity rendering of partial primitives is required [S1, unofficial].
## Background
The method enables viewers to display three spatial dimensions and navigate non-displayed dimensions via sliders (continuous) or toggles (categorical/boolean) [S1, unofficial].
## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
