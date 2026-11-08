# Adaptive Volumetric Clipping by CSG Culling

Also called: CSG Culling, CSG Culling of Shell, CSG Culling of Solid

## Overview
Adaptive Volumetric Clipping by CSG Culling is a hierarchical culling and projection technique for medical volume Gaussian Splatting that integrates a virtual Constructive Solid Geometry (CSG) probe into the tree traversal process [S1, unofficial]. It solves the problem of rendering sharp volumetric intersections between a Shell (surface) and a Solid (volume) by dynamically adjusting splat resolution and projecting centers to the probe surface, thereby eliminating popping artifacts and silhouette errors that occur with binary keep/drop projections. It sets itself apart from [Exact Half-Space Clipping](3395-exact-half-space-clipping.md) by using hierarchical bounding sphere tests against an implicit probe rather than computing conditional Gaussian CDFs relative to affine-EWA footprints, and from [Camera-independent geometric cut error (CErr3D)](3838-camera-independent-geometric-cut-error-cerr3d.md) by actively modifying splat positions and sizes rather than just estimating error.

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical tree of Gaussian splats (Shell and Solid nodes), each with a bounding sphere defined by center $c$, radius $r$, and a boundary bit flag. A virtual CSG probe defined by an implicit signed distance function (SDF) $S(x)$, where $S(x) < 0$ is inside, $S(x) > 0$ is outside, and $S(x) = 0$ is the surface. The gradient $\nabla S(x)$ provides the normal vector $n$.
*   **Output:** A set of splats rendered with adaptive resolution and projected centers, forming the CSG intersection of the volume and the probe.

**Data Structures:**
*   **Node:** Contains bounding sphere parameters ($c, r$), a boolean `is_boundary` flag, and child pointers.
*   **Boundary Bit Inheritance:** The `is_boundary` flag of a parent Solid node is the logical OR of the `is_boundary` flags of its children. This ensures that nodes near the intersection curve are marked as boundary nodes, enabling adaptive resolution [S1, unofficial].

**Parameters:**
*   $\delta_r$: Minimum allowed pixel splat size for regular internal nodes.
*   $\delta_b$: Minimum allowed pixel splat size for boundary nodes.
*   Constraint: $\delta_r > \delta_b \approx 1$ pixel. This reduces silhouette appearance along the intersection curve caused by crossing/overlapping splats [S1, unofficial].

**Steps:**

1.  **Probe Evaluation:** For a given node with bounding sphere center $c$ and radius $r$, evaluate the SDF $S(c)$ and the gradient $\nabla S(c)$ to determine the signed shortest distance $d$ to the probe surface and the inward-pointing normal $n = -\frac{\nabla S(c)}{\|\nabla S(c)\|}$ [S1, unofficial]. Note: The source defines $n$ pointing inwards and $d$ positive inside. Thus, if $c$ is inside, $d > 0$. The offset direction is $-n$ (outwards) if we want to project *onto* the surface from inside? Let's re-read carefully. "Offset center c along local normal vector n... distance... d... sign convention of d is positive distances inside the probe, i.e. n points inwards... new center: $c - d * n$". If $c$ is inside ($d>0$) and $n$ points inwards, $c - d*n$ moves $c$ outwards towards the surface. Correct.

2.  **CSG Culling of Solid:**
    *   If the bounding sphere does not intersect the probe surface (i.e., the sphere is completely inside or completely outside), discard the node and its subtree.
    *   If the bounding sphere intersects the probe surface, traverse the subtree.
    *   For each node in the traversal:
        *   If the node is a **boundary node** (marked by inheritance or proximity to intersection), render if splat size $\le \delta_b$.
        *   If the node is a **regular internal node**, render if splat size $\le \delta_r$.
        *   If splat size $> \delta_{type}$, continue traversal to children.
    *   **Projection:** For all rendered Solid splats that intersect the probe, project the center $c$ to the probe surface:
        $$ c_{new} = c - d \cdot n $$
        This prevents popping effects when resolution changes [S1, unofficial].

3.  **CSG Culling of Shell:**
    *   If the bounding sphere is **completely inside** the probe ($S(c) + r < 0$), do not render the node or its children.
    *   If the bounding sphere is **completely outside** the probe ($S(c) - r > 0$), disable CSG culling for this branch. Render children only if they are leaf nodes or have splat size $\le \delta_r$.
    *   If the bounding sphere **intersects** the probe surface:
        *   Render the node if it is a leaf node or splat size $\le \delta_b$.
        *   Otherwise, traverse with CSG culling enabled.
    *   **Projection:** For boundary nodes (those intersecting the probe or marked as boundary), offset the sphere center before rendering:
        $$ c_{new} = c - (r + d) \cdot n $$
        This improves the outline and reduces silhouettes at the intersection [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_solid_node(node, probe_sdf, delta_r, delta_b):
    """
    Recursively traverse and render Solid splats with CSG culling.
    """
    c, r = node.center, node.radius
    d, n = probe_sdf.evaluate(c) # d is signed distance (positive inside), n is inward normal
    
    # Check intersection with probe surface
    # Sphere intersects if |d| < r. 
    # Note: Source says "If bounding sphere does not intersect... discard".
    # Intersection condition: -r < d < r ? 
    # Actually, SDF value d is distance to surface. If d > r, sphere is completely inside (assuming d positive inside).
    # If d < -r, sphere is completely outside.
    # Intersection if -r <= d <= r.
    
    if d > r or d < -r:
        return # Discard node and subtree

    # Determine minimum size based on boundary status
    is_boundary = node.is_boundary
    min_size = delta_b if is_boundary else delta_r

    # Render if small enough
    if node.splat_size <= min_size:
        # Project center for Solid
        c_proj = c - d * n
        render_splat(node, c_proj)
        return

    # Traverse children if too large
    for child in node.children:
        render_solid_node(child, probe_sdf, delta_r, delta_b)

def render_shell_node(node, probe_sdf, delta_r, delta_b):
    """
    Recursively traverse and render Shell splats with CSG culling.
    """
    c, r = node.center, node.radius
    d, n = probe_sdf.evaluate(c)

    # Case 1: Completely inside probe
    if d > r: # Assuming d positive inside. If d > r, center is deeper than radius from surface.
        return # Do not render

    # Case 2: Completely outside probe
    if d < -r:
        # Disable CSG culling for this branch
        # Render only if leaf or splat size <= delta_r
        if node.is_leaf or node.splat_size <= delta_r:
            render_splat(node, c) # No projection for outside? Source says "rendered only if...". 
                                  # It doesn't explicitly say project outside splats. 
                                  # It says "offset the sphere center of the boundary nodes". 
                                  # Boundary nodes are those intersecting.
        return

    # Case 3: Intersecting probe surface
    is_boundary = node.is_boundary or (abs(d) <= r) # Implicitly boundary if intersecting
    
    if node.is_leaf or node.splat_size <= delta_b:
        # Project center for boundary/intersecting nodes
        c_proj = c - (r + d) * n
        render_splat(node, c_proj)
        return

    # Traverse with CSG culling enabled
    for child in node.children:
        render_shell_node(child, probe_sdf, delta_r, delta_b)
```

## Operating characteristics
*   **Time Complexity:** Linear in the number of nodes visited. The hierarchical culling significantly reduces the number of splats processed near the intersection curve compared to rendering all splats and masking them later. The overhead is described as "insignificant" because few Solid spheres are located exactly along the intersection curve [S1, unofficial].
*   **Memory Complexity:** No additional memory beyond the existing tree structure and the SDF evaluation stack.
*   **Strengths:** Real-time performance is maintained. It provides smooth adaptive resolution at the CSG intersection, reducing visual artifacts like popping and silhouettes. It works with any implicit probe (plane, sphere, box, custom SDF) [S1, unofficial].
*   **Limitations:** Requires the ability to evaluate an SDF and its gradient efficiently. The quality depends on the choice of $\delta_r$ and $\delta_b$. If $\delta_r$ is not sufficiently larger than $\delta_b$, silhouette artifacts may persist.
*   **When to use:** When rendering CSG intersections of volumes and surfaces in real-time medical visualization where visual fidelity at the cut boundary is critical.
*   **When not to use:** If the probe is not defined by an efficient SDF, or if the scene does not require sharp volumetric clipping.

## Background
The method was introduced by Kenneth Museth and Olivier Lombeyda in the paper "TetSplat: Real-time Rendering and Volume Clipping of Large..." presented at the VIS 2004 conference [S1, unofficial]. It was developed to address the limitations of binary clipping in Gaussian Splatting, aiming to maintain interactive frame rates while achieving high-quality volumetric editing and clipping. The technique leverages the hierarchical nature of the splat tree and the properties of implicit surfaces to optimize the rendering of CSG operations.

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
