# Average Normal Calculation for Cut Triangles

Also called: Cut Plane Normal Averaging, Interior Tetrahedra Normal Calculation

## Overview
This method computes smooth isosurface normals for newly generated triangles created by cutting tetrahedra during the Marching Tetrahedrons Algorithm. It specifically addresses triangles that lie strictly inside the isosurface (i.e., the parent tetrahedron contains only cutting planes, not the isosurface itself). It differs from gradient-based approximations by averaging the normals of the intersecting cutting planes, providing a geometrically consistent normal for interior cut faces.

## How it works
**Inputs:**
- The position of the vertex or fragment being shaded.
- The geometry of the tetrahedron containing the cut.
- The isovalue defining the cutting planes.
- The scalar field values at the tetrahedron's vertices.

**Outputs:**
- A normalized 3D vector representing the surface normal for the cut triangle.

**Data Structures:**
- Scalar field values $f_i$ for vertices $i=0..3$ of the tetrahedron.
- Vertex positions $\mathbf{v}_i$ for vertices $i=0..3$.
- Isosurface threshold $T$.

**Steps:**
1. Identify the tetrahedron configuration. Determine if the tetrahedron contains the isosurface or only cutting planes. This method applies only when the tetrahedron contains only cutting planes (interior to the isosurface) [S1, unofficial].
2. Compute the distance of each vertex to the cutting plane. For each vertex $i$, calculate the signed distance $d_i$ to the isosurface plane defined by the isovalue $T$:
   $$ d_i = f_i - T $$
   [S1, based on ABE+03 citation, unofficial].
3. Determine the cutting plane normal. The normal of the cutting plane is derived from the gradient of the scalar field or the geometric arrangement of the intersected edges. In the context of [ABE+03], the cutting plane normal $\mathbf{n}_{cut}$ is computed using the distances $d_i$ and vertex positions [S1, unofficial].
4. Calculate the average normal. For the new triangle formed by the cut, compute the average of the normals of the cutting planes associated with the tetrahedron [S1, unofficial].
   $$ \mathbf{n}_{avg} = \frac{\sum \mathbf{n}_{cut}}{\|\sum \mathbf{n}_{cut}\|} $$
   [G: Normalization is required to ensure a unit vector].
5. Output $\mathbf{n}_{avg}$ for shading.

**Pitfalls:**
- This calculation must be performed in the vertex shader. Performing it in the fragment shader is inefficient or incorrect for this specific variant [S1, unofficial].
- Precision issues may arise if the tetrahedron is nearly degenerate or if the isovalue is very close to a vertex value, causing unstable plane normals.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_average_normal_cut_triangle(tetrahedron, isovalue):
    """
    Calculates the average normal for a cut triangle in an interior tetrahedron.
    
    Args:
        tetrahedron: Object with vertices [v0, v1, v2, v3] and scalar values [f0, f1, f2, f3]
        isovalue: The threshold T for the isosurface
        
    Returns:
        Normalized 3D vector representing the average normal
    """
    # Step 1: Verify tetrahedron contains only cutting planes (interior)
    # This check is typically done during the marching tetrahedrons traversal
    # If the tetrahedron contains the isosurface, this method is not used.
    
    # Step 2: Compute distances to the cutting plane for each vertex
    distances = []
    for i in range(4):
        d_i = tetrahedron.scalar_values[i] - isovalue
        distances.append(d_i)
        
    # Step 3: Compute cutting plane normals
    # According to [ABE+03] cited in [S1, unofficial], the cutting plane normal is computed
    # using the distance of the vertex's tetrahedron to the plane.
    # The exact formula from ABE+03 is not provided in [S1, unofficial], so we assume
    # a standard gradient approximation or geometric normal calculation.
    # For implementation, one typically interpolates vertex normals or computes
    # the plane normal from the intersected edges.
    
    # Placeholder for ABE+03 specific normal calculation
    # In many implementations, the normal is the average of the gradients
    # at the vertices of the cut face.
    cut_normals = []
    for i in range(4):
        # Compute gradient/normal for vertex i based on distances and positions
        # This is a simplified representation; actual implementation depends on ABE+03
        # Assuming a simple average of vertex normals if available, or gradient
        # n_i = normalize(gradient_at_vertex_i)
        # Since [S1, unofficial] refers to ABE+03, we note that the normal is derived from
        # the distance to the plane.
        n_i = compute_plane_normal_from_distance(distances[i], tetrahedron.vertices[i])
        cut_normals.append(n_i)
        
    # Step 4: Average the cutting plane normals
    sum_normal = vec3(0.0, 0.0, 0.0)
    for n in cut_normals:
        sum_normal += n
        
    # Step 5: Normalize and return
    if length(sum_normal) > 0.0:
        return normalize(sum_normal)
    else:
        return vec3(0.0, 0.0, 1.0) # Fallback normal

def compute_plane_normal_from_distance(distance, vertex_position):
    """
    Placeholder for the specific normal calculation from ABE+03.
    In practice, this might involve the gradient of the scalar field.
    """
    # Not given in the sources. Implementation depends on ABE+03.
    # A common approach is to use the gradient of the scalar field at the vertex.
    return vertex_position # Placeholder
```

## Subtopics

### Gradient-Based Normal Approximation for Cut Triangles
This variant approximates the normal from the gradient at the fragment's position using volume information stored in a 3D texture. It is used for new triangles of the cut that belong to tetrahedra containing the isosurface. It differs from the average normal method by using local gradient information rather than averaging cutting plane normals. It must be performed in the fragment shader because 3D texture access is extremely slow at the vertex shader level [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per vertex/fragment, as it involves a fixed number of arithmetic operations and normalizations.
- **Memory Complexity:** $O(1)$ additional memory, as it does not require storing large data structures.
- **Performance:** Fast when implemented in the vertex shader, as it avoids expensive texture lookups [S1, unofficial].
- **Strengths:** Provides smooth normals for interior cut faces, enhancing visual quality. Efficient in the vertex shader.
- **Limitations:** Only applicable to tetrahedra containing only cutting planes. Requires careful handling of degenerate cases.
- **When to use:** When rendering interior cut faces in Marching Tetrahedrons and vertex shader performance is critical.
- **When not to use:** For tetrahedra containing the isosurface (use gradient-based approximation instead) or when fragment shader access is preferred for other reasons.

## Background
This method is part of the MTCut algorithm, which extends the Marching Tetrahedrons Algorithm to support interactive cutting of isosurfaces. It was introduced in the paper "MTCut: GPU-based Marching Tetra Cuts" [S1, unofficial]. The specific normal calculation for interior tetrahedra relies on the cutting plane normal computation described in [ABE+03]. The method was tested on a P-IV PC with a GeForce 6800 Ultra graphics card, demonstrating its efficiency for real-time haptic and visual feedback [S1, unofficial].

## Sources
[S1] MTCut: GPU-based Marching Tetra Cuts, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/037-044, fetched 2026-10-26 (unofficial)
