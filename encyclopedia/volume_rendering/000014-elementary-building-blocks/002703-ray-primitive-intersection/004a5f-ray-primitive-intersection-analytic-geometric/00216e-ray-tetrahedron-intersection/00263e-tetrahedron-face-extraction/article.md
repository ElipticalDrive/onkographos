# Tetrahedron Face Extraction


## Overview
Computes the four planar faces (unit normals and distances to the origin) from a four-vertex description of a tetrahedron [S1, unofficial]. It serves as the geometric preprocessing step for [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md), converting vertex-based topology into plane-based primitives suitable for analytic ray intersection tests. Unlike sibling methods that optimize traversal or use Plücker coordinates, this method focuses strictly on the algebraic derivation of face planes from vertex coordinates [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: `verts`, an array of four `r3d_rvec3` vectors representing the tetrahedron vertices $V_0, V_1, V_2, V_3$.
- Output: `faces`, an array of four `r3d_plane` structures. Each `r3d_plane` contains a normal vector $\mathbf{n}$ and a scalar distance $d$ defining the plane equation $\mathbf{n} \cdot \mathbf{x} + d = 0$.

**Data Structures**
The `r3d_rvec3` structure holds three floating-point components $(x, y, z)$ [S1, unofficial]. The `r3d_plane` structure holds a normal vector and a scalar distance [S1, unofficial].

**Algorithm**
For each face $i$ (where $i \in \{0, 1, 2, 3\}$), the face is defined by the three vertices excluding $V_i$. Let the vertices of face $i$ be $A, B, C$.

1. **Compute Edge Vectors**:
   Calculate two edge vectors from the face vertices:
   $$ \mathbf{e}_1 = B - A $$
   $$ \mathbf{e}_2 = C - A $$
   [S1, unofficial]

2. **Compute Normal**:
   Calculate the unnormalized normal vector using the cross product:
   $$ \mathbf{n}_{raw} = \mathbf{e}_1 \times \mathbf{e}_2 $$
   [S1, unofficial]

3. **Normalize**:
   Compute the magnitude $|\mathbf{n}_{raw}|$ and normalize:
   $$ \mathbf{n} = \frac{\mathbf{n}_{raw}}{|\mathbf{n}_{raw}|} $$
   [S1, unofficial]

4. **Compute Distance**:
   The distance $d$ to the origin is derived from the plane equation $\mathbf{n} \cdot \mathbf{x} + d = 0$. Using any vertex $A$ on the plane:
   $$ d = -(\mathbf{n} \cdot A) $$
   [S1, unofficial]

5. **Store**:
   Assign $\mathbf{n}$ and $d$ to `faces[i]`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedron_face_extraction(verts):
    """
    verts: list of 4 r3d_rvec3 objects (V0, V1, V2, V3)
    returns: list of 4 r3d_plane objects
    """
    faces = []
    
    # Define the indices for the 4 faces. 
    # Face 0 excludes V0, Face 1 excludes V1, etc.
    face_indices = [
        [1, 2, 3], # Face opposite V0
        [0, 2, 3], # Face opposite V1
        [0, 1, 3], # Face opposite V2
        [0, 1, 2]  # Face opposite V3
    ]
    
    for indices in face_indices:
        A = verts[indices[0]]
        B = verts[indices[1]]
        C = verts[indices[2]]
        
        # 1. Edge vectors
        e1 = vector_sub(B, A)
        e2 = vector_sub(C, A)
        
        # 2. Raw normal (cross product)
        n_raw = vector_cross(e1, e2)
        
        # 3. Normalize
        mag = vector_length(n_raw)
        if mag == 0:
            # Degenerate tetrahedron (coplanar vertices)
            # [G] Return a zero normal or handle error appropriately.
            n = vector_zero()
        else:
            n = vector_scale(n_raw, 1.0 / mag)
            
        # 4. Distance to origin
        # Plane equation: n . x + d = 0 => d = -(n . A)
        dot_product = vector_dot(n, A)
        d = -dot_product
        
        # 5. Store
        faces.append(r3d_plane(normal=n, distance=d))
        
    return faces

def vector_sub(a, b):
    return r3d_rvec3(a.x - b.x, a.y - b.y, a.z - b.z)

def vector_cross(a, b):
    return r3d_rvec3(
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    )

def vector_length(v):
    return math.sqrt(v.x**2 + v.y**2 + v.z**2)

def vector_scale(v, s):
    return r3d_rvec3(v.x * s, v.y * s, v.z * s)

def vector_dot(a, b):
    return a.x * b.x + a.y * b.y + a.z * b.z

def vector_zero():
    return r3d_rvec3(0.0, 0.0, 0.0)
```

**Parameters and Edge Cases**
- **Degenerate Input**: If the four vertices are coplanar, the volume is zero, and at least one face will have a zero-length normal vector. The implementation must guard against division by zero during normalization [S1, unofficial].
- **Winding Order**: The direction of the normal depends on the winding order of the vertices $A, B, C$. Consistent winding is required for correct inside/outside tests in subsequent intersection steps [S1, unofficial].

## Subtopics

### Tetrahedron Signed Volume Calculation
Calculates the signed volume of a tetrahedron defined by four input vertices $P_a, P_b, P_c, P_d$ [S1, unofficial]. The return value is the signed volume; a positive value indicates a specific winding order, while a negative value indicates the opposite [S1, unofficial]. This is often used to verify vertex ordering or to correct winding by swapping vertices if the volume is negative [S1, unofficial]. The volume is typically computed as $\frac{1}{6} |(\mathbf{b}-\mathbf{a}) \cdot ((\mathbf{c}-\mathbf{a}) \times (\mathbf{d}-\mathbf{a}))|$ [G].

## Operating characteristics
**Time and Memory Complexity**
- Time: $O(1)$ per tetrahedron. It involves a constant number of vector operations (3 subtractions, 2 cross products, 1 dot product, 1 normalization).
- Memory: $O(1)$ auxiliary space. The output size is fixed at 4 planes.

**Strengths and Limitations**
- **Strengths**: Simple, deterministic, and numerically stable for non-degenerate tetrahedra. It provides the exact plane representation needed for analytic ray intersection.
- **Limitations**: Does not handle degenerate tetrahedra (zero volume) gracefully without explicit checks. The normal direction depends on vertex winding, which must be consistent across the mesh.

**When to Use**
- Use when you have a static tetrahedral mesh and need to perform ray-tetrahedron intersection tests using analytic plane equations.
- Do not use if the mesh contains degenerate cells or if winding order is inconsistent.

## Background
This method is part of the `r3d` library, a C-based ray tracing library developed by Devon Powell [S1, unofficial]. The library provides low-level geometric primitives for ray tracing, including tetrahedral intersection support. The specific function `r3du_tet_faces_from_verts` is documented in the library's source code and headers [S1, unofficial].

## Sources
[S1] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
