# Vertex Normal Averaging for Shading

Also called: Face Normal Computation (shading-based), Shading Normal Computation, Surface Normal Averaging for Shading

## Overview
Vertex Normal Averaging for Shading computes a smooth shading normal for a polygonal face by averaging the pre-computed vertex normals of its three corners and normalizing the result [S2, unofficial]. It serves as the shading-based alternative to geometric face normal computation [G]. This approach reduces visible faceting on rendered isosurfaces compared to flat shading [S1].

## How it works
**Inputs:**
- Three vertex normals $\mathbf{n}_0, \mathbf{n}_1, \mathbf{n}_2$, each a 3D vector [S2, unofficial]. These are typically pre-computed as renormalized averages of the surface normal vectors of the triangles that share the vertex [S1].

**Outputs:**
- A single normalized 3D vector $\mathbf{n}_{face}$ representing the shading normal for the face [S2, unofficial].

**Data Structures:**
- Vertex normals are stored as 3D floating-point vectors [G]. Bit-packing details are Not given in the sources.

**Steps:**
1. Retrieve the three vertex normals $\mathbf{n}_0, \mathbf{n}_1, \mathbf{n}_2$ associated with the face's vertices [S2, unofficial].
2. Compute the component-wise arithmetic mean of the three vectors.
3. Normalize the resulting vector to ensure it is a unit vector.
4. Pass $\mathbf{n}_{face}$ to the underlying rendering function for use in shading the triangles [S1].

**Formulas:**
- Component-wise average: $\mathbf{v}_{avg} = \frac{\mathbf{n}_0 + \mathbf{n}_1 + \mathbf{n}_2}{3.0}$
- Normalization: $\mathbf{n}_{face} = \frac{\mathbf{v}_{avg}}{\|\mathbf{v}_{avg}\|}$
- Where $\mathbf{n}_i = (x_i, y_i, z_i)$ are the input vertex normals, $\mathbf{v}_{avg}$ is the intermediate average vector, and $\|\cdot\|$ denotes the Euclidean norm.

**Parameters:**
- Typical values are Not given in the sources.

**Edge Cases and Pitfalls:**
- **Degenerate Geometry:** If the three vertex normals sum to a zero vector (e.g., normals pointing in opposite directions that cancel out), normalization fails and requires a guard [G].
- **Precision:** The averaging process can slightly reduce the magnitude before normalization, but standard floating-point precision is sufficient for standard graphics applications [G].
- **Boundary Conditions:** This method relies entirely on the quality of the pre-computed vertex normals [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
    def __add__(self, other):
        return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __truediv__(self, scalar):
        return Vector3(self.x / scalar, self.y / scalar, self.z / scalar)
    def magnitude(self):
        return (self.x**2 + self.y**2 + self.z**2)**0.5

def compute_shading_normal(n0, n1, n2):
    # Step 2: Compute component-wise average
    avg = (n0 + n1 + n2) / 3.0
    # Step 3: Normalize the result
    mag = avg.magnitude()
    if mag < 1e-9:
        return Vector3(0.0, 0.0, 1.0) # [G] Default fallback for degenerate cancellation
    return avg / mag
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per face. It involves three vector additions, one division, and one normalization (square root and division) [G].
- **Memory Complexity:** $O(1)$ additional memory per face [G].
- **Performance:** Shading increases both computing time and memory usage compared to flat shading [S1]. The cost is low per triangle but adds up over large meshes [G].
- **Strengths:** Significantly reduces faceting artifacts on smooth surfaces or low-poly isosurfaces [S1]. It is computationally inexpensive compared to higher-order interpolation methods [G].
- **Limitations:** Not necessarily appropriate in all situations [S1]. It does not account for curvature within the face, only the average orientation of the vertices [G].

## Background
This method is described in the context of the `rgl` package in R, specifically for rendering isosurfaces of density functions [S1]. The technique is used to smooth the visual appearance of multiple isosurfaces, such as those derived from tri-variate normal distributions [S1]. It is implemented by computing surface normal vectors at vertices as renormalized averages of the surface normal vectors of the triangles that share the vertex, which are then passed to the underlying rendering function `triangles3d` [S1].

## Sources
[S1] Computing and Displaying Isosurfaces in R, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
