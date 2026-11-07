# Two-level Nested Ray Marching

Also called: Front-to-back ray marching on AMR, Ray Marching with Coarse Mesh and Refinement Tree

## Overview
Two-level Nested Ray Marching is a volume rendering technique that performs front-to-back compositing by traversing a coarse mesh of tetrahedral clusters and then marching through the fine-level tetrahedra contained within each active cluster [S1, unofficial]. It solves the problem of efficiently visualizing sparse, multi-resolution Adaptive Mesh Refinement (AMR) data by avoiding the memory blow-up of uniform grid resampling while maintaining native resolution [S1, unofficial]. Unlike its sibling [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md) which may use different sampling strategies, this method explicitly uses a constant adjustable step size for the inner loop and relies on macrocell density statistics for early-out culling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a coarse mesh BVH; macrocell values (scalar statistics per coarse element); a refinement tree mapping coarse elements to descendant tetrahedra; a transfer function $tf$; a constant step size $s$; a background color $\mathbf{c}_{bg}$; and an opacity threshold $\alpha_{thresh} = 0.99$ [S1, unofficial].
*   **Output:** The final accumulated color $\mathbf{c}_{out}$ and opacity $\alpha_{out}$ for the pixel [S1, unofficial].

**Data Structures:**
*   **Coarse Mesh:** Indexed by a BVH. Each node/leaf represents a cluster of tetrahedra.
*   **Macrocell Values:** An array `macrocellValues` where `macrocellValues[id]` stores the maximum density or scalar value for coarse element `id` [S1, unofficial].
*   **Refinement Tree:** A structure allowing lookup of tetrahedra descendants for a given coarse element ID.
*   **Transfer Function:** Provides color $\mathbf{c}_{tf}$ and opacity $\alpha_{tf}$ for a given scalar value [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Set accumulated color $\mathbf{c} = \{0, 0, 0\}$ and accumulated opacity $\alpha = 0$ [S1, unofficial].
2.  **Coarse Traversal Loop:** While $\alpha < \alpha_{thresh}$:
    a.  Call `TraverseCoarseMesh(ray)` to find the current coarse element ID `coarseID`, the entry distance $t_{back}$, and the exit distance to the next face $t_{nextFace}$ [S1, unofficial].
    b.  If `coarseID` is $-1$ (no more intersections), break the loop [S1, unofficial].
    c.  **Culling:** Check if `macrocellValues[coarseID] <= 0`. If true, the element is empty or inactive. Skip processing and update `ray.t = max(t_{nextFace}, ray.t)` to advance to the next coarse element [S1, unofficial].
    d.  **Fine Traversal:** If not skipped, call `MarchElement(ray, c, \alpha, t_{back}, coarseID, s, tf)` [S1, unofficial].
    e.  **Advance:** Update `ray.t = max(t_{nextFace}, ray.t)` [S1, unofficial].
3.  **Final Compositing:** Return $\mathbf{c}_{out} = (1 - \alpha) \mathbf{c}_{bg} + \mathbf{c}$ [S1, unofficial].

**Inside `MarchElement`:**
1.  Initialize local exit distance $t_{exit} = 0$ and tetrahedron ID `sampleTetID` [S1, unofficial].
2.  **Fine Loop:** While `ray.t < t_{max}` (where $t_{max}$ is the exit of the coarse element):
    a.  Calculate sample point: $\mathbf{p} = \mathbf{o} + \text{ray}.t \cdot \mathbf{d}$ [S1, unofficial].
    b.  **Tetrahedron Lookup:** If `ray.t > t_{exit}`:
        i.  Call `traverse(\mathbf{p}, coarseTetID)` to get the current tetrahedron ID `sampleTetID` and its vertices `tetVertices` [S1, unofficial].
        ii. Calculate $t_{exit}$ by intersecting the ray with the faces of `tetVertices` [S1, unofficial].
    c.  **Sampling:** Sample the scalar value $v$ at $\mathbf{p}$ using `sampleData(\mathbf{p}, sampleTetID, tetVertices)` [S1, unofficial].
    d.  **Step Advance:** Update `ray.t = ray.t + s` [S1, unofficial].
    e.  **Transfer Function Check:** If $v$ is not in the transfer function range (`!tf.inRange(v)`), continue to the next iteration [S1, unofficial].
    f.  **Compositing:**
        i.  Get transfer function color and opacity: $\mathbf{c}_{tf}, \alpha_{tf} = tf.getColor(v)$ [S1, unofficial].
        ii. Apply global opacity: $\alpha_{tf} = \alpha_{tf} \cdot tf.opacity$ [S1, unofficial].
        iii. Accumulate color: $\mathbf{c} = \mathbf{c} + (1 - \alpha) \cdot \alpha_{tf} \cdot \mathbf{c}_{tf}$ [S1, unofficial].
        iv. Accumulate opacity: $\alpha = \alpha + \alpha_{tf} \cdot (1 - \alpha)$ [S1, unofficial].
3.  Return updated $\mathbf{c}$ and $\alpha$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_coarse(ray, macrocell_values, step_size, tf, bg_color, alpha_thresh=0.99):
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    
    while alpha < alpha_thresh:
        coarse_id, t_back, t_next_face = traverse_coarse_mesh(ray)
        
        if coarse_id == -1:
            break
            
        if macrocell_values[coarse_id] <= 0:
            ray.t = max(t_next_face, ray.t)
            continue
            
        color, alpha = march_element(ray, color, alpha, t_back, coarse_id, step_size, tf)
        ray.t = max(t_next_face, ray.t)
        
    return (1.0 - alpha) * bg_color + color

def march_element(ray, color, alpha, t_max, coarse_tet_id, step_size, tf):
    t_exit = 0.0
    sample_tet_id = 0
    tet_vertices = []
    
    while ray.t < t_max:
        sample_point = ray.org + ray.t * ray.dir
        
        if ray.t > t_exit:
            sample_tet_id, tet_vertices = traverse(sample_point, coarse_tet_id)
            t_exit = intersect_faces(ray.t, sample_point, ray.direction, tet_vertices)
            
        scalar = sample_data(sample_point, sample_tet_id, tet_vertices)
        ray.t = ray.t + step_size
        
        if not tf.in_range(scalar):
            continue
            
        tf_color = tf.get_color(scalar)
        tf_opacity = tf_color.opacity * tf.opacity
        
        color = color + (1.0 - alpha) * tf_opacity * tf_color.color
        alpha = alpha + tf_opacity * (1.0 - alpha)
        
    return color, alpha
```

**Parameters:**
*   **Step Size ($s$):** A constant adjustable step size. Smaller values increase accuracy but reduce performance [S1, unofficial].
*   **Opacity Threshold:** The loop terminates when accumulated opacity exceeds 0.99 [S1, unofficial].

## Subtopics

### [AMR Volume Ray-Casting](4032-amr-volume-ray-casting.md)
A variant that ray-marches AMR octrees directly without resampling, storing leaves in per-level hashes to scale cost with actual data rather than bounding box volume.

## Operating characteristics
*   **Complexity:** Time complexity depends on the number of coarse elements intersected and the number of fine tetrahedra sampled within them, scaled by the inverse of the step size [S1, unofficial]. Memory complexity is proportional to the size of the AMR data structure and the coarse mesh BVH [S1, unofficial].
*   **Performance:** Performance is impacted by the choice of step size; smaller steps yield more samples and higher quality but slower rendering [S1, unofficial].
*   **Strengths:** Avoids memory blow-up associated with uniform grid resampling ($2^{l_{max}})^3$) and maintains native resolution of the simulation data [S1, unofficial].
*   **Limitations:** Uses a constant step size, lacking the adaptive adjustments that could optimize sampling density in complex regions [S1, unofficial].

## Background
The method originates from research on direct volume rendering of tree-based tetrahedral adaptive mesh refinement (AMR) data [S1, unofficial]. It was developed to address the inefficiencies of resampling sparse AMR data onto uniform grids, which incurs significant memory and time costs [S1, unofficial]. The approach builds on the principles of hierarchical traversal and macrocell-based culling established in the parent [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md) [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
