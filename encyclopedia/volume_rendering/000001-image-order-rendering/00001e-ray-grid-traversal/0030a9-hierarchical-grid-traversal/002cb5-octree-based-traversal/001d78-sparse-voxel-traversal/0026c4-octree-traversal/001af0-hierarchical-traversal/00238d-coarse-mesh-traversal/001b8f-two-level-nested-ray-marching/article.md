# Two-level Nested Ray Marching

Also called: Front-to-back ray marching on AMR, Ray Marching with Coarse Mesh and Refinement Tree

## Overview
Two-level nested ray marching is a volume rendering technique for Adaptive Mesh Refinement (AMR) data that decouples traversal into a coarse mesh level and a fine refinement tree level [S1, unofficial]. It solves the problem of efficiently sampling hierarchical tetrahedral data by skipping empty coarse elements and performing dense front-to-back compositing only within active refinement structures [S1, unofficial]. Unlike sibling methods that may use hierarchical accelerators like K-D trees or octrees for general sparse voxel data, this method is specifically tailored for tree-based tetrahedral AMR, relying on a constant step size for sampling within the finest elements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{r}_{org}$, direction $\mathbf{r}_{dir}$, and current distance $t$ [S1, unofficial]. A coarse mesh structure providing `TraverseCoarseMesh` and macrocell density values `macrocellValues` [S1, unofficial]. A refinement tree structure providing `traverse` (to find the current tetrahedron) and `intersectFaces` (to find exit distance) [S1, unofficial]. A transfer function `tf` mapping scalar values to color and opacity [S1, unofficial]. A constant step size `stepSize` [S1, unofficial]. An opacity threshold of $0.99$ [S1, unofficial]. A background color `bgColor` [S1, unofficial].
*   **Output:** The accumulated color and final alpha value [S1, unofficial].

**Data Structures:**
*   `macrocellValues`: An array indexed by coarse element ID, storing the maximum density within that coarse element [S1, unofficial].
*   `tf`: A transfer function object with methods `inRange(scalar)` and `getColor(scalar)` [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Set accumulated color $C = \{0, 0, 0\}$ and accumulated alpha $\alpha = 0$ [S1, unofficial].
2.  **Coarse Loop:** While $\alpha < 0.99$:
    a.  Call `TraverseCoarseMesh(ray)` to obtain the current coarse element ID (`coarseID`), the back face distance of the coarse element (`tBack`), and the distance to the next face (`tNextFace`) [S1, unofficial].
    b.  If `coarseID` is $-1$, break the loop (ray has exited the volume) [S1, unofficial].
    c.  Check if `macrocellValues[coarseID] <= 0` [S1, unofficial].
    d.  If true, the coarse element is empty; skip processing [S1, unofficial].
    e.  If not empty, call `MarchElement(ray, C, \alpha, tBack, coarseID, stepSize, tf)` to process the interior [S1, unofficial].
    f.  Update `ray.t` to $\max(tNextFace, ray.t)$ to advance the ray to the start of the next coarse element [S1, unofficial].
3.  **Final Compositing:** Return the final color: $C_{final} = (1 - \alpha) \times bgColor + C$ [S1, unofficial].

**Sub-algorithm: MarchElement**
This function handles the nested march within a single coarse element [S1, unofficial].

1.  Initialize `tExit` = `ray.t`, `sampleTetID` = `coarseTetID`, and `tetVertices` = vertices of `coarseTetID` [G].
2.  **Fine Loop:** While `ray.t` < `tMax` (the back face distance of the coarse element):
    a.  Calculate the sample point: $\mathbf{p} = \mathbf{r}_{org} + ray.t \times \mathbf{r}_{dir}$ [S1, unofficial].
    b.  **Refinement Traversal:** If `ray.t` > `tExit`:
        i.  Call `traverse(samplePoint, coarseTetID)` to get the current tetrahedron ID (`sampleTetID`) and its vertices (`tetVertices`) [S1, unofficial].
        ii. Calculate `tExit` by intersecting the ray with the faces of the current tetrahedron using `intersectFaces(ray.t, samplePoint, ray.direction, tetVertices)` [S1, unofficial].
    c.  **Sampling:** Sample the scalar value $s$ at $\mathbf{p}$ using `sampleData(samplePoint, sampleTetID, tetVertices)` [S1, unofficial].
    d.  **Transfer Function Check:** If $s$ is not in the transfer function range (`!tf.inRange(s)`), continue to the next iteration (skip this sample) [S1, unofficial].
    e.  **Advance Ray:** Update `ray.t` = `ray.t` + `stepSize` [S1, unofficial].
    f.  **Compositing:**
        i.  Get the transfer function color and opacity: $tfColor = tf.getColor(s)$ [S1, unofficial].
        ii. Adjust opacity: $tfColor.opacity = tfColor.opacity \times tf.opacity$ [S1, unofficial].
        iii. Accumulate color and alpha using front-to-back compositing:
            $$C = C + (1 - \alpha) \times tfColor.opacity \times tfColor.color$$
            $$\alpha = \alpha + tfColor.opacity \times (1 - \alpha)$$
3.  Return the updated $C$ and $\alpha$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_coarse(ray, macrocell_values, step_size, tf, bg_color):
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    
    while alpha < 0.99:
        coarse_id, t_back, t_next_face = traverse_coarse_mesh(ray)
        
        if coarse_id == -1:
            break
            
        if macrocell_values[coarse_id] <= 0:
            ray.t = max(t_next_face, ray.t)
            continue
            
        color, alpha = march_element(ray, color, alpha, t_back, coarse_id, step_size, tf)
        ray.t = max(t_next_face, ray.t)
        
    final_color = (1.0 - alpha) * bg_color + color
    return final_color

def march_element(ray, color, alpha, t_max, coarse_tet_id, step_size, tf):
    t_exit = ray.t
    sample_tet_id = coarse_tet_id
    tet_vertices = get_tet_vertices(coarse_tet_id)
    
    while ray.t < t_max:
        sample_point = ray.org + ray.t * ray.dir
        
        if ray.t > t_exit:
            sample_tet_id, tet_vertices = traverse(sample_point, coarse_tet_id)
            t_exit = intersect_faces(ray.t, sample_point, ray.dir, tet_vertices)
            
        scalar = sample_data(sample_point, sample_tet_id, tet_vertices)
        
        if not tf.in_range(scalar):
            ray.t += step_size
            continue
            
        ray.t += step_size
        
        tf_color = tf.get_color(scalar)
        tf_color.opacity = tf_color.opacity * tf.opacity
        
        color = color + (1.0 - alpha) * tf_color.opacity * tf_color.color
        alpha = alpha + tf_color.opacity * (1.0 - alpha)
        
    return color, alpha

def traverse_coarse_mesh(ray):
    # Standard ray-AABB traversal for the coarse mesh grid
    t_min = (ray.box_min - ray.org) / ray.dir
    t_max_val = (ray.box_max - ray.org) / ray.dir
    t_enter = max(min(t_min))
    t_exit_val = min(max(t_max_val))
    if t_enter > t_exit_val or t_exit_val < 0:
        return -1, 0.0, 0.0
    return current_coarse_id, t_exit_val, t_enter + step_size

def traverse(sample_point, coarse_tet_id):
    # Standard tree descent: walk down from coarse_tet_id to find containing tet
    node = coarse_tet_id
    while not is_leaf(node):
        children = get_children(node)
        found = False
        for child in children:
            if point_in_tet(sample_point, get_tet_vertices(child)):
                node = child
                found = True
                break
        if not found:
            break
    return node, get_tet_vertices(node)

def intersect_faces(ray_t, sample_point, ray_dir, tet_vertices):
    # Ray-tetrahedron exit distance calculation
    max_t = float('inf')
    for face in tet_faces(tet_vertices):
        normal = cross(face[1] - face[0], face[2] - face[0])
        denom = dot(normal, ray_dir)
        if abs(denom) > 1e-8:
            t = dot(normal, face[0] - sample_point) / denom
            if t > ray_t and t < max_t:
                max_t = t
    return max_t if max_t != float('inf') else ray_t

def sample_data(sample_point, sample_tet_id, tet_vertices):
    # Barycentric interpolation of scalar values at tet vertices
    weights = barycentric_coordinates(sample_point, tet_vertices)
    scalar = sum(w * v.scalar for w, v in zip(weights, tet_vertices))
    return scalar

def get_tet_vertices(tet_id):
    return mesh.tetrahedra[tet_id].vertices

def is_leaf(node):
    return not mesh.has_children(node)

def get_children(node):
    return mesh.children[node]

def point_in_tet(point, vertices):
    weights = barycentric_coordinates(point, vertices)
    return all(w >= -1e-6 for w in weights)

def tet_faces(vertices):
    return [
        (vertices[0], vertices[1], vertices[2]),
        (vertices[0], vertices[1], vertices[3]),
        (vertices[0], vertices[2], vertices[3]),
        (vertices[1], vertices[2], vertices[3])
    ]

def barycentric_coordinates(point, vertices):
    v0, v1, v2, v3 = vertices
    vol = dot(v1 - v0, cross(v2 - v0, v3 - v0))
    w0 = dot(point - v1, cross(v2 - v1, v3 - v1)) / vol
    w1 = dot(point - v0, cross(v3 - v0, v2 - v0)) / vol
    w2 = dot(point - v0, cross(v1 - v0, v3 - v0)) / vol
    w3 = 1.0 - w0 - w1 - w2
    return [w0, w1, w2, w3]
```

**Parameters:**
*   **Step Size:** A constant adjustable step size. No adaptive step size adjustments are implemented [S1, unofficial]. Smaller step sizes increase sample count and impact performance [S1, unofficial].
*   **Opacity Threshold:** The ray marching loop continues while $\alpha < 0.99$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Coarse Elements:** The method explicitly checks `macrocellValues[coarseID] <= 0` to skip empty elements, avoiding unnecessary refinement tree traversals [S1, unofficial].
*   **Precision:** The method relies on a constant step size, which may lead to aliasing or missed features if the step size is too large relative to the finest tetrahedra [S1, unofficial].
*   **Boundary Conditions:** The `intersect_faces` function must correctly handle the ray exiting the current tetrahedron to update `tExit` [S1, unofficial]. Initializing `tExit` to `ray.t` prevents immediate re-traversal or numerical exit distance errors when the ray starts inside a cell [G].

## Operating characteristics
*   **Time Complexity:** The complexity depends on the number of coarse elements intersected and the number of fine tetrahedra sampled within each. The coarse traversal skips empty regions, but the fine march uses a constant step size, meaning performance degrades linearly with the number of samples required to cover the active volume [S1, unofficial].
*   **Memory Complexity:** Requires storage for the coarse mesh macrocell values and the refinement tree structure [S1, unofficial].
*   **Strengths:** Efficiently handles AMR data by leveraging coarse mesh density checks to skip empty regions. Simple implementation due to the use of a constant step size [S1, unofficial].
*   **Limitations:** The constant step size is not adaptive, potentially leading to inefficient sampling in regions with varying density or feature size. It is specific to tree-based tetrahedral AMR data [S1, unofficial].
*   **When to use:** When rendering tree-based tetrahedral AMR data where coarse mesh density information is available and adaptive step sizing is not required or desired [S1, unofficial].
*   **When not to use:** For data requiring adaptive sampling or non-tetrahedral hierarchical structures [S1, unofficial].

## Background
This method was described in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement data [S1, unofficial]. It builds on the concept of hierarchical traversal by explicitly separating the coarse mesh navigation from the fine element sampling [S1, unofficial]. The approach allows for optimizations such as macrocell density checks to skip empty coarse elements [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
