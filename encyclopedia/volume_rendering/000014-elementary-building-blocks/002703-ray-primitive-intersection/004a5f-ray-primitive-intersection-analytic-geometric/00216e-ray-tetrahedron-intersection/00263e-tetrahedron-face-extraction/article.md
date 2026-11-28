# Tetrahedron Face Extraction


## Overview
Computes the four planar faces (unit normals and distances to the origin) from a four-vertex description of a tetrahedron [S1, unofficial]. It provides the exact plane representation required for analytic ray-tetrahedron intersection tests [G].

## How it works
**Inputs and Outputs**
- Input: `verts`, an array of four `r3d_rvec3` vectors representing the tetrahedron vertices $V_0, V_1, V_2, V_3$ [S1, unofficial].
- Output: `faces`, an array of four `r3d_plane` structures [S1, unofficial].

**Data Structures**
The internal layout of `r3d_rvec3` and `r3d_plane` is not given in the sources. The implementation uses standard 3D vector and plane representations [G].

**Algorithm**
For each face $i$ (where $i \in \{0, 1, 2, 3\}$), the face is defined by the three vertices excluding $V_i$ [S1, unofficial]. Let the vertices of face $i$ be $A, B, C$ [G].

1. **Compute Edge Vectors**: Calculate two edge vectors from the face vertices:
   $$ \mathbf{e}_1 = B - A $$
   $$ \mathbf{e}_2 = C - A $$
   [G]

2. **Compute Normal**: Calculate the unnormalized normal vector using the cross product:
   $$ \mathbf{n}_{raw} = \mathbf{e}_1 \times \mathbf{e}_2 $$
   [G]

3. **Normalize**: Compute the magnitude $|\mathbf{n}_{raw}|$ and normalize:
   $$ \mathbf{n} = \frac{\mathbf{n}_{raw}}{|\mathbf{n}_{raw}|} $$
   [G]

4. **Compute Distance**: The distance $d$ to the origin is derived from the plane equation $\mathbf{n} \cdot \mathbf{x} + d = 0$ [G]. Using any vertex $A$ on the plane:
   $$ d = -(\mathbf{n} \cdot A) $$
   [G]

5. **Store**: Assign $\mathbf{n}$ and $d$ to `faces[i]` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedron_face_extraction(verts):
    """
    verts: list of 4 Vec3 objects (V0, V1, V2, V3)
    returns: list of 4 Plane objects
    """
    faces = []
    # Face indices: each face excludes one vertex
    face_indices = [[1, 2, 3], [0, 2, 3], [0, 1, 3], [0, 1, 2]]
    
    for idx in face_indices:
        A = verts[idx[0]]
        B = verts[idx[1]]
        C = verts[idx[2]]
        
        # 1. Edge vectors
        e1 = Vec3_sub(B, A)
        e2 = Vec3_sub(C, A)
        
        # 2. Raw normal (cross product)
        n_raw = Vec3_cross(e1, e2)
        
        # 3. Normalize
        mag = Vec3_length(n_raw)
        if mag == 0.0:
            # [G] Degenerate face: return zero vector to avoid division by zero
            n = Vec3(0.0, 0.0, 0.0)
        else:
            n = Vec3_scale(n_raw, 1.0 / mag)
            
        # 4. Distance to origin
        # Plane equation: n . x + d = 0 => d = -(n . A)
        dot_product = Vec3_dot(n, A)
        d = -dot_product
        
        # 5. Store
        faces.append(Plane(n, d))
        
    return faces

def Vec3_sub(a, b):
    return Vec3(a.x - b.x, a.y - b.y, a.z - b.z)

def Vec3_cross(a, b):
    return Vec3(
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    )

def Vec3_length(v):
    return (v.x**2 + v.y**2 + v.z**2) ** 0.5

def Vec3_scale(v, s):
    return Vec3(v.x * s, v.y * s, v.z * s)

def Vec3_dot(a, b):
    return a.x * b.x + a.y * b.y + a.z * b.z

class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

class Plane:
    def __init__(self, normal, distance):
        self.normal = normal
        self.distance = distance
```

**Parameters and Edge Cases**
- **Degenerate Input**: If the four vertices are coplanar, the volume is zero, and at least one face will have a zero-length normal vector [G]. The pseudocode guards against division by zero during normalization [G].
- **Winding Order**: The direction of the normal depends on the winding order of the vertices $A, B, C$ [G]. Consistent winding is required for correct inside/outside tests in subsequent intersection steps [G].

## Subtopics

### Tetrahedron Signed Volume Calculation
Calculates the signed volume of a tetrahedron defined by four input vertices $P_a, P_b, P_c, P_d$ [S1, unofficial]. The return value is the signed volume; a positive value indicates a specific winding order, while a negative value indicates the opposite [S1, unofficial]. This is often used to verify vertex ordering or to correct winding by swapping vertices if the volume is negative [S1, unofficial]. The volume is computed as $\frac{1}{6} |(\mathbf{b}-\mathbf{a}) \cdot ((\mathbf{c}-\mathbf{a}) \times (\mathbf{d}-\mathbf{a}))|$ [G].

## Operating characteristics
**Time and Memory Complexity**
- Time: $O(1)$ per tetrahedron. It involves a constant number of vector operations (3 subtractions, 2 cross products, 1 dot product, 1 normalization) [G].
- Memory: $O(1)$ auxiliary space. The output size is fixed at 4 planes [G].

**Measured Performance and Scaling**
Not given in the sources.

**Strengths and Limitations**
- **Strengths**: Simple, deterministic, and numerically stable for non-degenerate tetrahedra [G]. It provides the exact plane representation needed for analytic ray intersection [G].
- **Limitations**: Does not handle degenerate tetrahedra (zero volume) gracefully without explicit checks [G]. The normal direction depends on vertex winding, which must be consistent across the mesh [G].

**When to Use**
Not given in the sources.

## Background
Originates from the `r3d` library ecosystem [S1, unofficial]. It is used for tetrahedral mesh preprocessing in ray tracing pipelines [G]. Specific historical development details and authorship are not given in the sources.

## Sources
[S1] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
