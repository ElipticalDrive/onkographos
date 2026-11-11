# Plane-Cube Intersection

Also called: Plane-Cube Intersection Volume, Volume from a plane slicing a cube

## Overview
This method computes the volume of the irregular polyhedron formed by the intersection of a plane and a cube [S1, unofficial]. It is specifically designed for applications in computational astrophysics, such as determining the fraction of a cell that is partial ionized or covered by a burning front [S1, unofficial]. It extends the approach of Salama & Kolb (2005) by explicitly constructing an ordered list of face vertices to calculate the volume of the resulting inscribed irregular polyhedron [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**:
  - A unit cube defined by 8 vertices $\mathbf{V}_1$ through $\mathbf{V}_8$ [S1, unofficial].
  - A plane defined by a unit normal vector $\mathbf{\hat{n}}$ and a scalar distance $dl$ from the origin [S1, unofficial].
- **Outputs**:
  - The volume of the intersection polyhedron [S1, unofficial].
  - An ordered list of intersection points (vertices of the polygon) [S1, unofficial].

**Data Structures**
- **Vertices**: The cube vertices are enumerated and ordered as shown in the source figure [S1, unofficial]. The specific coordinate layout is not explicitly detailed in the text, but the unit cube implies vertices at $(0,0,0)$ to $(1,1,1)$ in some permutation. The source references a figure for the enumeration [S1, unofficial].
- **Edges**: Defined by pairs of vertices $(\mathbf{V}_i, \mathbf{V}_j)$ [S1, unofficial].
- **Intersection Points**: A list of points $\mathbf{P}$ where the plane intersects the cube edges [S1, unofficial].

**Steps**
1. **Define the Plane**:
   The plane is defined by the length of the normal ray $dl$ and spherical angles $\theta$ and $\phi$ [S1, unofficial].
   Calculate the Cartesian coordinates of the normal ray $\mathbf{n}$:
   $$ \mathbf{n} = [dl \cos(\theta), \ dl \sin(\theta)\sin(\phi), \ dl \cos(\phi) ] $$
   The unit normal vector is $\mathbf{\hat{n}} = \mathbf{n} / \|\mathbf{n}\|$ [S1, unofficial]. Note: If $dl=0$, the normal is undefined; this is an edge case [G].
   The plane equation is:
   $$ dl = \mathbf{\hat{n}} \cdot \mathbf{x} $$
   where $\mathbf{x}$ is a point on the plane [S1, unofficial].

2. **Iterate Over Edges**:
   For each edge $E_{i \rightarrow j}$ connecting vertex $\mathbf{V}_i$ to $\mathbf{V}_j$:
   - Define the line segment:
     $$ E_{i \rightarrow j} = \mathbf{V}_i + \lambda (\mathbf{V}_j - \mathbf{V}_i) $$
   - Calculate the intersection parameter $\lambda$:
     $$ \lambda = \frac{ dl - \mathbf{\hat{n}} \cdot \mathbf{V}_i }{\mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i ) } $$
   - Check validity: An intersection exists only if $0 \le \lambda \le 1$ [S1, unofficial]. If valid, compute the intersection point $\mathbf{P} = \mathbf{V}_i + \lambda (\mathbf{V}_j - \mathbf{V}_i)$ and add it to the list of intersection points [S1, unofficial].

3. **Construct Ordered List**:
   Construct an ordered list of the face vertices (intersection points) [S1, unofficial]. The source states this extends Salama & Kolb (2005) by constructing this ordered list [S1, unofficial]. The specific algorithm for ordering is not detailed in the provided text, but it is necessary for volume calculation [S1, unofficial].

4. **Compute Volume**:
   Calculate the volume of the irregular polyhedron defined by the ordered list of vertices [S1, unofficial]. The specific volume formula is not provided in the text [S1, unofficial].

**Parameters**
- $dl$: Length of the normal ray [S1, unofficial].
- $\theta$: Spherical angle, $0 \le \theta \le 2\pi$ [S1, unofficial].
- $\phi$: Spherical angle, $0 \le \phi \le \pi$ [S1, unofficial].

**Edge Cases and Pitfalls**
- **Degenerate Plane**: If $dl=0$, the normal vector $\mathbf{n}$ is zero, and $\mathbf{\hat{n}}$ is undefined [G].
- **Parallel Edges**: If $\mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i) = 0$, the denominator is zero, meaning the edge is parallel to the plane. The intersection is either empty or the entire edge lies on the plane [G].
- **Boundary Conditions**: The condition $0 \le \lambda \le 1$ ensures the intersection is within the cube edge [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_cube_intersection(dl, theta, phi, cube_vertices):
    """
    Computes the volume of the intersection between a plane and a cube.
    
    Args:
        dl: Distance from origin to plane along normal.
        theta: Spherical angle for normal vector (0 to 2*pi).
        phi: Spherical angle for normal vector (0 to pi).
        cube_vertices: List of 8 vertices V1 to V8 of the unit cube.
        
    Returns:
        volume: Volume of the intersection polyhedron.
        intersection_points: List of intersection points on cube edges.
    """
    # Step 1: Define the plane normal
    # Calculate Cartesian coordinates of the normal ray
    nx = dl * cos(theta)
    ny = dl * sin(theta) * sin(phi)
    nz = dl * cos(phi)
    
    # Calculate unit normal vector
    norm_length = sqrt(nx**2 + ny**2 + nz**2)
    if norm_length == 0:
        raise ValueError("Normal vector is zero. dl=0 is degenerate.")
    
    n_hat_x = nx / norm_length
    n_hat_y = ny / norm_length
    n_hat_z = nz / norm_length
    
    # Step 2: Iterate over edges and find intersections
    intersection_points = []
    
    # Define edges as pairs of vertex indices (example: V1-V2, V2-V3, etc.)
    # The specific enumeration order is defined in the source figure [S1, unofficial]
    # Here we assume a generic list of edge pairs (i, j)
    edges = get_cube_edges(cube_vertices) 
    
    for V_i, V_j in edges:
        # Vector from V_i to V_j
        dx = V_j.x - V_i.x
        dy = V_j.y - V_i.y
        dz = V_j.z - V_i.z
        
        # Dot product of normal and edge vector
        denom = n_hat_x * dx + n_hat_y * dy + n_hat_z * dz
        
        # Check for parallel edge
        if abs(denom) < 1e-9:
            continue # Edge is parallel to plane
            
        # Calculate lambda
        # Numerator: dl - n_hat . V_i
        dot_Vi = n_hat_x * V_i.x + n_hat_y * V_i.y + n_hat_z * V_i.z
        lambda_val = (dl - dot_Vi) / denom
        
        # Check validity: 0 <= lambda <= 1
        if 0 <= lambda_val <= 1:
            # Calculate intersection point
            P_x = V_i.x + lambda_val * dx
            P_y = V_i.y + lambda_val * dy
            P_z = V_i.z + lambda_val * dz
            intersection_points.append((P_x, P_y, P_z))
            
    # Step 3: Construct ordered list of vertices
    # The source mentions constructing an ordered list of face vertices [S1, unofficial]
    # The specific ordering algorithm is not provided in the text [S1, unofficial]
    ordered_vertices = order_vertices(intersection_points)
    
    # Step 4: Compute volume
    # The specific volume formula is not provided in the text [S1, unofficial]
    volume = compute_polyhedron_volume(ordered_vertices)
    
    return volume, intersection_points

def get_cube_edges(vertices):
    """
    Returns a list of edge pairs (V_i, V_j) for a unit cube.
    The specific enumeration is defined in the source figure [S1, unofficial].
    This is a placeholder for the actual edge list.
    """
    # Example edge list for a standard unit cube enumeration
    # V1=(0,0,0), V2=(1,0,0), V3=(1,1,0), V4=(0,1,0), V5=(0,0,1), V6=(1,0,1), V7=(1,1,1), V8=(0,1,1)
    # Edges: 1-2, 2-3, 3-4, 4-1, 5-6, 6-7, 7-8, 8-5, 1-5, 2-6, 3-7, 4-8
    # Note: The source uses a specific enumeration shown in a figure [S1, unofficial]
    return [
        (vertices[0], vertices[1]), (vertices[1], vertices[2]), (vertices[2], vertices[3]), (vertices[3], vertices[0]),
        (vertices[4], vertices[5]), (vertices[5], vertices[6]), (vertices[6], vertices[7]), (vertices[7], vertices[4]),
        (vertices[0], vertices[4]), (vertices[1], vertices[5]), (vertices[2], vertices[6]), (vertices[3], vertices[7])
    ]

def order_vertices(points):
    """
    Orders the intersection points to form a polygon.
    The specific algorithm is not provided in the text [S1, unofficial].
    """
    # Placeholder for ordering logic
    return sorted(points)

def compute_polyhedron_volume(vertices):
    """
    Computes the volume of the irregular polyhedron.
    The specific formula is not provided in the text [S1, unofficial].
    """
    # Placeholder for volume calculation
    return 0.0
```

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of edges in the cube (constant 12 edges for a cube) [S1, unofficial].
- **Memory Complexity**: Constant, storing a fixed number of vertices and intersection points [S1, unofficial].
- **Strengths**: Provides an exact geometric solution for the volume of intersection [S1, unofficial].
- **Limitations**: The ordering of vertices for volume calculation is not explicitly detailed, which may require additional implementation effort [S1, unofficial]. The number of vertices of the intersecting polygon can vary between 3 and 6, leading to complex patterns in the $\theta-\phi$ plane [S1, unofficial].

## Background
- **Origin**: The method is based on the work of Salama & Kolb (2005) and is extended by the author of the source [S1, unofficial].
- **History**: The tool `cube_and_plane.tbz` was developed for applications in computational astrophysics [S1, unofficial].
- **Usage**: Used in computational astrophysics for determining the fraction of a cell that is partial ionized or covered by a burning front [S1, unofficial].

## Sources
[S1] volume of intersection between a plane and a cube - cococubed, https://cococubed.com/code_pages/raybox.shtml, fetched 2026-10-29 (unofficial)
