# Average Normal Calculation for Cut Triangles

Also called: Cut Plane Normal Averaging, Interior Tetrahedra Normal Calculation

## Overview
This method computes surface normals for newly generated triangles created when cutting tetrahedra that contain only cutting planes (interior to the isosurface) [S1, unofficial]. It differs from gradient-based approximations by averaging the normals of the intersecting cutting planes, providing a geometrically consistent normal for interior cut faces [S1, unofficial].

## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.
**Data Structures:** Not given in the sources.
**Steps:**
1. Identify new triangles belonging to tetrahedra that only contain cutting planes (interior to the isosurface) [S1, unofficial].
2. Compute the cutting plane normal for each vertex using the distance of the vertex's tetrahedron to the plane, as described in [ABE+03] [S1, unofficial].
3. Calculate the average normal by summing the cutting plane normals and normalizing the result [S1, unofficial].
4. Output the averaged normal for shading [S1, unofficial].
**Formulas:** $\mathbf{n}_{avg} = \frac{\sum_{i=0}^{3} \mathbf{n}_{cut, i}}{\|\sum_{i=0}^{3} \mathbf{n}_{cut, i}\|}$ [G: Standard vector normalization], where $\mathbf{n}_{cut, i}$ is the cutting plane normal for vertex $i$ and $\mathbf{n}_{avg}$ is the resulting averaged normal.
**Parameters:** Not given in the sources.
**Edge cases and pitfalls:** This specific normal calculation must be performed in the vertex shader [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_average_normal_cut_triangle(distances, vertex_positions):
    # Step 1: Identify interior tetrahedra triangles (handled by traversal)
    
    # Step 2: Compute cutting plane normals from vertex distances
    # Not given in the sources. Implementation depends on [ABE+03].
    # Compute normal from distance of vertex's tetrahedron to the plane.
    cut_normals = []
    for i in range(4):
        n_i = compute_cutting_plane_normal(distances[i], vertex_positions[i])
        cut_normals.append(n_i)
        
    # Step 3: Average the cutting plane normals
    sum_normal = [0.0, 0.0, 0.0]
    for n in cut_normals:
        sum_normal[0] += n[0]
        sum_normal[1] += n[1]
        sum_normal[2] += n[2]
        
    # Step 4: Normalize and return
    magnitude = (sum_normal[0]**2 + sum_normal[1]**2 + sum_normal[2]**2)**0.5
    if magnitude > 0.0:
        return [sum_normal[0]/magnitude, sum_normal[1]/magnitude, sum_normal[2]/magnitude]
    else:
        return [0.0, 0.0, 1.0]
```

## Subtopics

### Gradient-Based Normal Approximation for Cut Triangles
This variant approximates the normal from the gradient at the fragment's position using volume information stored in a 3D texture [S1, unofficial]. It is used for new triangles of the cut that belong to tetrahedra containing the isosurface [S1, unofficial]. It differs from the average normal method by using local gradient information rather than averaging cutting plane normals [S1, unofficial]. It must be performed in the fragment shader because 3D texture access is extremely slow at the vertex shader level [S1, unofficial].

## Operating characteristics
Time and memory complexity are not given in the sources [S1, unofficial]. Performance testing was conducted on a Pentium IV PC with 1 GB RAM and a GeForce 6800 Ultra graphics card with 256 MB memory, showing real-time framerates for interactive cutting on models like the heart, jaw, hip, and head [S1, unofficial]. Strengths include smooth normals for interior cut faces and efficient vertex shader execution [S1, unofficial]. Limitations include applicability only to tetrahedra containing only cutting planes [S1, unofficial]. Use for interior cut faces in Marching Tetrahedrons when vertex shader performance is critical; avoid for tetrahedra containing the isosurface [S1, unofficial].

## Background
This method is part of the MTCut algorithm, which extends the Marching Tetrahedrons Algorithm to support interactive cutting of isosurfaces [S1, unofficial]. The specific normal calculation for interior tetrahedra relies on the cutting plane normal computation described in [ABE+03] [S1, unofficial]. The method was tested on a Pentium IV PC with a GeForce 6800 Ultra graphics card, demonstrating its efficiency for real-time haptic and visual feedback [S1, unofficial].

## Sources
[S1] MTCut: GPU-based Marching Tetra Cuts, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/037-044, fetched 2026-10-26 (unofficial)
