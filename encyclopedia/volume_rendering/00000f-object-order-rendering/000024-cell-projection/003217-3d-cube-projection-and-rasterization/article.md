# 3D Cube Projection and Rasterization

Also called: 3D Box Rasterization, 3D Cube Face Projection and Rasterization, Perspective Cube Drawing, Software 3D Cube Face Projection & Rasterization

## Overview
A software rendering technique that transforms the eight corners of a 3D voxel cube using sequential translation and rotation matrices, applies perspective projection to the vertices of each face, and rasterizes visible faces as 2D polygons [S1, unofficial]. It differs from hardware rasterization by performing all depth testing, culling, and pixel writing in software, and from standard raycasting by explicitly drawing polygonal faces rather than casting rays per pixel [S1, unofficial].

## How it works
**Inputs:**
*   Cube center position $(x, y, z)$ in world space.
*   Cube dimensions: width, height, depth.
*   Camera orientation: rotation angles $x\_rad, y\_rad, z\_rad$ (radians) around X, Y, Z axes.
*   Pre-translation rotation angles: $x\_rad\_translate, y\_rad\_translate, z\_rad\_translate$ (radians).
*   Screen dimensions: $window\_width, window\_height$.
*   Field of View scale factor: $FOV$.
*   Face color.

**Outputs:**
*   2D screen pixels filled with the face color for visible faces.

**Data Structures:**
*   Vertices: 4 points per face, each with $(x, y, z)$ coordinates.
*   Matrices: $3 \times 3$ rotation matrices for X, Y, Z axes.

**Steps:**
1.  **Define Local Vertices:** For the target face, define the 4 corner vertices in local object space relative to the cube center. For example, the front face corners are $(-width/2, -height/2, -depth/2)$, $(width/2, -height/2, -depth/2)$, $(width/2, -height/2, depth/2)$, and $(-width/2, -height/2, depth/2)$ [S1, unofficial].
2.  **Transform Vertices:** For each vertex $(vx, vy, vz)$, apply the following sequence of matrix multiplications to obtain world-space coordinates $(x_i, y_i, z_i)$ [S1, unofficial]:
    a.  Translate by half-dimensions: $P = [vx, vy, vz]$.
    b.  Rotate around Z by $z\_rad\_translate$: $P = P \times R_z(z\_rad\_translate)$.
    c.  Rotate around Y by $y\_rad\_translate$: $P = P \times R_y(y\_rad\_translate)$.
    d.  Rotate around X by $x\_rad\_translate$: $P = P \times R_x(x\_rad\_translate)$.
    e.  Translate to world position: $P = [x + P_x, y + P_y, z + P_z]$.
    f.  Rotate around Z by $z\_rad$: $P = P \times R_z(z\_rad)$.
    g.  Rotate around Y by $y\_rad$: $P = P \times R_y(y\_rad)$.
    h.  Rotate around X by $x\_rad$: $P = P \times R_x(x\_rad)$.
    i.  Store result as $(x_i, y_i, z_i)$ [S1, unofficial].
3.  **Visibility Check:** Verify if all 4 vertices are in front of the camera by checking $z_1 > 0, z_2 > 0, z_3 > 0, z_4 > 0$. If any vertex has $z_i \le 0$, the face is behind or on the camera plane and is not drawn [S1, unofficial].
4.  **Perspective Projection:** Project each vertex to 2D screen coordinates $(sx_i, sy_i)$ using:
    $$ sx_i = \frac{x_i}{z_i} \cdot \text{FOV} + \frac{window\_width}{2} $$
    $$ sy_i = \frac{y_i}{z_i} \cdot \text{FOV} + \frac{window\_height}{2} $$
    [S1, unofficial].
5.  **Rasterization:** Draw the projected quadrilateral as a filled polygon using the screen coordinates $(sx_1, sy_1)$ to $(sx_4, sy_4)$ [S1, unofficial].

**Formulas:**
Rotation matrices [S1, unofficial]:
$$ R_z(\theta) = \begin{bmatrix} \cos\theta & -\sin\theta & 0 \\ \sin\theta & \cos\theta & 0 \\ 0 & 0 & 1 \end{bmatrix}, \quad R_y(\theta) = \begin{bmatrix} \cos\theta & 0 & \sin\theta \\ 0 & 1 & 0 \\ -\sin\theta & 0 & \cos\theta \end{bmatrix}, \quad R_x(\theta) = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos\theta & -\sin\theta \\ 0 & \sin\theta & \cos\theta \end{bmatrix} $$

**Pitfalls:**
*   Division by zero if any vertex has $z \le 0$. Handled by the visibility check [S1, unofficial].
*   Distance culling: If the cube's position exceeds a maximum distance threshold, the cube is skipped entirely to save computation [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rotate_z(point, theta):
    c, s = math.cos(theta), math.sin(theta)
    return [point[0]*c - point[1]*s, point[0]*s + point[1]*c, point[2]]

def rotate_y(point, theta):
    c, s = math.cos(theta), math.sin(theta)
    return [point[0]*c + point[2]*s, point[1], -point[0]*s + point[2]*c]

def rotate_x(point, theta):
    c, s = math.cos(theta), math.sin(theta)
    return [point[0], point[1]*c - point[2]*s, point[1]*s + point[2]*c]

def transform_vertex(vx, vy, vz, x, y, z, width, height, depth, 
                     x_rad, y_rad, z_rad, 
                     x_rad_translate, y_rad_translate, z_rad_translate,
                     FOV, window_width, window_height):
    # 1. Translate by half-dimensions (local to center)
    p = [vx, vy, vz]
    
    # 2. Rotate around Z by z_rad_translate
    p = rotate_z(p, z_rad_translate)
    
    # 3. Rotate around Y by y_rad_translate
    p = rotate_y(p, y_rad_translate)
    
    # 4. Rotate around X by x_rad_translate
    p = rotate_x(p, x_rad_translate)
    
    # 5. Translate to world position
    p = [x + p[0], y + p[1], z + p[2]]
    
    # 6. Rotate around Z by z_rad
    p = rotate_z(p, z_rad)
    
    # 7. Rotate around Y by y_rad
    p = rotate_y(p, y_rad)
    
    # 8. Rotate around X by x_rad
    p = rotate_x(p, x_rad)
    
    x_i, y_i, z_i = p
    
    # Visibility check
    if z_i <= 0:
        return None
        
    # Perspective projection
    sx = (x_i / z_i) * FOV + (window_width / 2)
    sy = (y_i / z_i) * FOV + (window_height / 2)
    
    return (sx, sy)

def draw_face(vertices, x, y, z, width, height, depth, 
              x_rad, y_rad, z_rad, 
              x_rad_translate, y_rad_translate, z_rad_translate,
              FOV, window_width, window_height):
    projected_points = []
    for vx, vy, vz in vertices:
        pt = transform_vertex(vx, vy, vz, x, y, z, width, height, depth,
                              x_rad, y_rad, z_rad,
                              x_rad_translate, y_rad_translate, z_rad_translate,
                              FOV, window_width, window_height)
        if pt is None:
            return False # Face not visible
        projected_points.append(pt)
    return projected_points
```

## Subtopics

### [2D Raycasting with Z-Buffering](8233-2d-raycasting-with-z-buffering.md)
A software rendering method that projects 3D voxel block surfaces onto a 2D screen using perspective projection and resolves depth using a per-pixel Z-buffer. It differs from standard raycasting by rasterizing explicit polygonal faces of blocks rather than casting rays per pixel, and from hardware rasterization by performing all depth testing and pixel writing in software.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of visible faces. Each face requires 4 vertex transformations and a constant-time rasterization.
*   **Memory Complexity:** Constant per face for vertex storage.
*   **Strengths:** Simple to implement in software; explicit control over face rendering order and culling.
*   **Limitations:** No hardware acceleration; requires manual implementation of polygon filling and depth buffering if transparency or overlapping faces are involved (though this specific method assumes opaque faces drawn in back-to-front order as per the parent [Cell projection](26-cell-projection.md)).
*   **When to use:** When a lightweight, dependency-free software renderer is needed for simple voxel cubes without complex lighting or transparency.

## Background
Originated in early software voxel rendering engines, specifically documented in a Python-based voxel rendering engine repository [S1, unofficial]. It serves as a foundational technique for object-order rendering of voxel data before moving to more complex cell projection methods.

## Sources
[S1] anb2473/Python-Voxel-Rendering-Engine, https://github.com/anb2473/Python-Voxel-Rendering-Engine, fetched 2026-10-27 (unofficial)
