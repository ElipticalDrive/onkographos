# Mesh-to-Voxel Geometry Shader

Also called: DirectX geometry shader voxelization, Mesh to voxel shaders

## Overview
The Mesh-to-Voxel Geometry Shader is a real-time DirectX technique that converts mesh objects into voxel-like cubes by iterating over integer grid positions within a triangle's bounding box and emitting explicit cube geometry for grid cells close to the triangle plane [S1, unofficial]. Unlike rasterization-based voxelization which fills pixels or conservative voxelization which determines occupancy, this method explicitly generates the 3D geometry of every intersecting voxel, resulting in significantly higher vertex and triangle counts [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method operates on a triangle primitive input in clip space. It outputs a stream of triangle strips representing the faces of the generated voxel cubes. The input vertex structure `appdata` contains `Position` (float3), `UV` (float4), `Normal` (float4), `Tangent` (float4), and `Binormal` (float4) [S1, unofficial]. The geometry shader output structure `geoOutput` contains `HPosition` (float4), `UV` (float2), and `Color` (float4) [S1, unofficial].

**Data Structures:**
The algorithm uses a fixed-size array `cubes` of 100 elements to store valid voxel centers and their distances. Each element is a `float4` where the RGB components represent the integer grid coordinates $(x, y, z)$ and the W component stores the distance metric (specifically $1 - \text{tempDist}$) [S1, unofficial].

**Parameters:**
*   `size`: Half-extent of the generated voxel cubes. Hardcoded to $0.5$ [S1, unofficial].
*   `Lamp0Pos`: World-space position of the point light source. Default: $\{-0.5, 2.0, 1.25\}$ [S1, unofficial].
*   `Lamp0Color`: Color/intensity of the point light source. Default: $\{2.0, 2.0, 2.0\}$ [S1, unofficial].

**Steps:**
1.  **Transform Vertices:** Transform the three input triangle vertices ($In[0], In[1], In[2]$) from clip space to world space using the `WorldXf` matrix to obtain $wpos1, wpos2, wpos3$ [S1, unofficial].
2.  **Calculate Bounding Box:** Compute the axis-aligned bounding box (AABB) of the triangle in world space. The minimum coordinates $(Ax, Ay, Az)$ and maximum coordinates $(Bx, By, Bz)$ are determined by comparing the world-space vertex coordinates. Note that the source code implicitly casts float world positions to `int` variables, which truncates towards zero [S1, unofficial].
3.  **Compute Triangle Normal:** Calculate the world-space normal $wNormal$ by normalizing the cross product of two edge vectors: $edgeA = wpos2 - wpos1$ and $edgeB = wpos3 - wpos1$ [S1, unofficial].
4.  **Compute Plane Delta:** Calculate the plane constant $\delta$ using the formula:
    $$ \delta = \text{dot}(wpos1, wNormal) $$
    where $wpos1$ is a vertex on the triangle plane and $wNormal$ is the normalized plane normal [S1, unofficial].
5.  **Iterate Grid Positions:** Iterate through all integer grid positions $(x, y, z)$ within the bounding box defined by $[Ax, Bx]$, $[Ay, By]$, and $[Az, Bz]$ [S1, unofficial].
6.  **Project and Distance Check:** For each grid point $curPos = (x, y, z)$:
    *   Project $curPos$ onto the triangle plane to get $tempPos$:
        $$ tempPos = curPos + (\delta - \text{dot}(curPos, wNormal)) \times wNormal $$
    *   Calculate the Euclidean distance $tempDist$ between $curPos$ and $tempPos$:
        $$ tempDist = \text{distance}(curPos, tempPos) $$
    *   If $tempDist < 1$, proceed to check if the point is inside the triangle [S1, unofficial].
7.  **Point-in-Triangle Test:** If the distance check passes, determine if $curPos$ lies inside the triangle using a `PointInTriangle` function. This function typically checks if the point is on the same side of all three edges (e.g., using cross products and dot products) [S1, unofficial].
8.  **Store Valid Voxels:** If the point is inside the triangle, store the voxel center and distance in the `cubes` array. The source code stores `float4(curPos, 1 - tempDist)` [S1, unofficial].
9.  **Generate Cube Geometry:** For each valid voxel center stored in `cubes`:
    *   Define the 8 vertices of a cube with half-extent `size` ($0.5$) centered at the voxel position. The vertices are calculated as offsets from the center position [S1, unofficial].
    *   Calculate diffuse lighting for each of the 6 faces of the cube. For each face, compute the face normal via the cross product of its edges, compute the light vector from the voxel position to `Lamp0Pos`, and calculate:
        $$ diffuseValue = \max(\text{dot}(lightVector, wNormal), 0.0) $$
    *   Compute the color for each face:
        $$ couleur = Lamp0Color \times couleur \times diffuseValue $$
        where `couleur` is a uniform color $\{1, 1, 1\}$ [S1, unofficial].
    *   Assign colors to the 12 triangles forming the cube. Each triangle's color is the average of the diffuse colors of the 3 faces it belongs to, multiplied by the uniform color and light color. Alpha is hardcoded to 1 [S1, unofficial].
    *   Transform the cube vertices to view space using `ViewIXf` and append them to the `TriStream` [S1, unofficial].
10. **Restart Strip:** Call `TriStream.RestartStrip()` after processing all cubes for the input triangle to end the primitive strip [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_voxel_geometry_shader(triangle_vertices_clip, world_matrix, view_inverse_matrix, lamp_pos, lamp_color):
    """
    Converts a single triangle into voxel cube geometry.
    
    Args:
        triangle_vertices_clip: List of 3 float4 vertices in clip space.
        world_matrix: 4x4 world transformation matrix.
        view_inverse_matrix: 4x4 view inverse transformation matrix.
        lamp_pos: float3 position of the light source.
        lamp_color: float3 color/intensity of the light source.
        
    Returns:
        List of triangle strips (list of vertices) representing the voxels.
    """
    
    # Step 1: Transform vertices to world space
    wpos1 = transform_clip_to_world(triangle_vertices_clip[0], world_matrix)
    wpos2 = transform_clip_to_world(triangle_vertices_clip[1], world_matrix)
    wpos3 = transform_clip_to_world(triangle_vertices_clip[2], world_matrix)
    
    # Step 2: Calculate Bounding Box (Integer Truncation)
    Ax = int(wpos1.x)
    Bx = int(wpos1.x)
    if wpos2.x < Ax: Ax = int(wpos2.x)
    if wpos2.x > Bx: Bx = int(wpos2.x)
    if wpos3.x < Ax: Ax = int(wpos3.x)
    if wpos3.x > Bx: Bx = int(wpos3.x)
    
    Ay = int(wpos1.y)
    By = int(wpos1.y)
    if wpos2.y < Ay: Ay = int(wpos2.y)
    if wpos2.y > By: By = int(wpos2.y)
    if wpos3.y < Ay: Ay = int(wpos3.y)
    if wpos3.y > By: By = int(wpos3.y)
    
    Az = int(wpos1.z)
    Bz = int(wpos1.z)
    if wpos2.z < Az: Az = int(wpos2.z)
    if wpos2.z > Bz: Bz = int(wpos2.z)
    if wpos3.z < Az: Az = int(wpos3.z)
    if wpos3.z > Bz: Bz = int(wpos3.z)
    
    # Step 3: Compute Triangle Normal
    edgeA = wpos2 - wpos1
    edgeB = wpos3 - wpos1
    wNormal = normalize(cross(edgeA, edgeB))
    
    # Step 4: Compute Plane Delta
    delta = dot(wpos1, wNormal)
    
    # Step 5-8: Iterate and Find Intersecting Voxels
    cubes = []
    for x in range(Ax, Bx + 1):
        for y in range(Ay, By + 1):
            for z in range(Az, Bz + 1):
                curPos = float3(x, y, z)
                
                # Project point onto plane
                tempPos = curPos + (delta - dot(curPos, wNormal)) * wNormal
                
                # Distance check
                tempDist = distance(curPos, tempPos)
                
                if tempDist < 1.0:
                    # Point-in-Triangle Test
                    if point_in_triangle(curPos, wpos1, wpos2, wpos3):
                        cubes.append((curPos, 1.0 - tempDist))
    
    # Step 9: Generate Cube Geometry
    output_triangles = []
    size = 0.5
    
    for voxel_center, _ in cubes:
        # Define 8 vertices of the cube
        # pos1: (-size, size, -size), pos2: (-size, -size, -size), etc.
        # Note: The source code defines specific vertex indices for the 12 triangles.
        # We will generate the 12 triangles explicitly as per the source.
        
        # Vertices relative to center
        v0 = voxel_center + float3(-size, size, -size)   # pos1
        v1 = voxel_center + float3(-size, -size, -size)  # pos2
        v2 = voxel_center + float3(size, size, -size)    # pos3
        v3 = voxel_center + float3(size, -size, -size)   # pos4
        v4 = voxel_center + float3(-size, size, size)    # pos5
        v5 = voxel_center + float3(-size, -size, size)   # pos6
        v6 = voxel_center + float3(size, size, size)     # pos7
        v7 = voxel_center + float3(size, -size, size)    # pos8
        
        # Calculate face normals and diffuse colors
        # Face 1 (Front, z=-size): v0, v1, v3, v2
        n1 = normalize(cross(v1 - v0, v2 - v0))
        l1 = normalize(lamp_pos - voxel_center)
        d1 = max(dot(l1, n1), 0.0)
        c1 = lamp_color * float3(1, 1, 1) * d1
        
        # Face 2 (Back, z=size): v6, v7, v5, v4
        n2 = normalize(cross(v7 - v6, v4 - v6)) # Note: Source uses pos7-pos3 and pos4-pos3 for face 2? 
        # Let's follow the source's specific face definitions for colors.
        # Source Face 2: edgeA = pos7-pos3, edgeB = pos4-pos3. 
        # pos7=v6, pos3=v2, pos4=v3. So n2 = cross(v6-v2, v3-v2).
        n2 = normalize(cross(v6 - v2, v3 - v2))
        l2 = normalize(lamp_pos - voxel_center)
        d2 = max(dot(l2, n2), 0.0)
        c2 = lamp_color * float3(1, 1, 1) * d2
        
        # Face 3 (Left, x=-size): v0, v4, v5, v1
        # Source Face 3: edgeA = pos5-pos1, edgeB = pos6-pos1.
        # pos5=v4, pos1=v0, pos6=v5. n3 = cross(v4-v0, v5-v0).
        n3 = normalize(cross(v4 - v0, v5 - v0))
        l3 = normalize(lamp_pos - voxel_center)
        d3 = max(dot(l3, n3), 0.0)
        c3 = lamp_color * float3(1, 1, 1) * d3
        
        # Face 4 (Right, x=size): v2, v3, v7, v6
        # Source Face 4: edgeA = pos6-pos4, edgeB = pos7-pos4.
        # pos6=v5, pos4=v3, pos7=v6. n4 = cross(v5-v3, v6-v3).
        n4 = normalize(cross(v5 - v3, v6 - v3))
        l4 = normalize(lamp_pos - voxel_center)
        d4 = max(dot(l4, n4), 0.0)
        c4 = lamp_color * float3(1, 1, 1) * d4
        
        # Face 5 (Top, y=size): v0, v2, v6, v4
        # Source Face 5: edgeA = pos3-pos1, edgeB = pos7-pos1.
        # pos3=v2, pos1=v0, pos7=v6. n5 = cross(v2-v0, v6-v0).
        n5 = normalize(cross(v2 - v0, v6 - v0))
        l5 = normalize(lamp_pos - voxel_center)
        d5 = max(dot(l5, n5), 0.0)
        c5 = lamp_color * float3(1, 1, 1) * d5
        
        # Face 6 (Bottom, y=-size): v1, v3, v7, v5
        # Source Face 6: edgeA = pos7-pos5, edgeB = pos6-pos5.
        # pos7=v6, pos5=v4, pos6=v5. n6 = cross(v6-v4, v5-v4).
        n6 = normalize(cross(v6 - v4, v5 - v4))
        l6 = normalize(lamp_pos - voxel_center)
        d6 = max(dot(l6, n6), 0.0)
        c6 = lamp_color * float3(1, 1, 1) * d6
        
        # Assign colors to 12 triangles
        # Triangle 1: pos4, pos2, pos3 -> v3, v1, v2. Color: (c1+c2+c3)/3
        # Note: The source code order for triangles is specific.
        # Let's map the source's triangle definitions to vertices.
        # Source Triangle 1: pos4, pos2, pos3 -> v3, v1, v2
        t1 = [v3, v1, v2]
        col1 = (c1 + c2 + c3) / 3.0
        
        # Source Triangle 2: pos2, pos8, pos3 -> v1, v7, v2
        t2 = [v1, v7, v2]
        col2 = (c1 + c3 + c5) / 3.0 # Source says (couleur1+couleur3+couleur5)/3
        
        # Source Triangle 3: pos3, pos2, pos4 -> v2, v1, v3
        t3 = [v2, v1, v3]
        col3 = (c1 + c2 + c4) / 3.0 # Source says (couleur1+couleur2+couleur4)/3
        
        # Source Triangle 4: pos3, pos8, pos4 -> v2, v7, v3
        t4 = [v2, v7, v3]
        col4 = (c2 + c4 + c6) / 3.0 # Source says (couleur2+couleur4+couleur6)/3
        
        # Source Triangle 5: pos4, pos2, pos8 -> v3, v1, v7
        t5 = [v3, v1, v7]
        col5 = (c1 + c3 + c4) / 3.0 # Source says (couleur1+couleur3+couleur4)/3
        
        # Source Triangle 6: pos4, pos8, pos6 -> v3, v7, v5
        t6 = [v3, v7, v5]
        col6 = (c4 + c5 + c6) / 3.0 # Source says (couleur4+couleur5+couleur6)/3
        
        # Source Triangle 7: pos5, pos1, pos7 -> v4, v0, v6
        t7 = [v4, v0, v6]
        col7 = (c3 + c5 + c6) / 3.0 # Source says (couleur3+couleur5+couleur6)/3
        
        # Source Triangle 8: pos5, pos7, pos1 -> v4, v6, v0
        t8 = [v4, v6, v0]
        col8 = (c3 + c5 + c6) / 3.0 # Source says (couleur3+couleur5+couleur6)/3
        
        # Source Triangle 9: pos1, pos5, pos3 -> v0, v4, v2
        t9 = [v0, v4, v2]
        col9 = (c1 + c3 + c5) / 3.0 # Source says (couleur1+couleur3+couleur5)/3
        
        # Source Triangle 10: pos1, pos3, pos5 -> v0, v2, v4
        t10 = [v0, v2, v4]
        col10 = (c1 + c3 + c5) / 3.0 # Source says (couleur1+couleur3+couleur5)/3
        
        # Source Triangle 11: pos7, pos5, pos1 -> v6, v4, v0
        t11 = [v6, v4, v0]
        col11 = (c3 + c5 + c6) / 3.0 # Source says (couleur3+couleur5+couleur6)/3
        
        # Source Triangle 12: pos5, pos7, pos3 -> v4, v6, v2
        t12 = [v4, v6, v2]
        col12 = (c3 + c5 + c6) / 3.0 # Source says (couleur3+couleur5+couleur6)/3
        
        # Append triangles to output
        # Each triangle is a strip of 3 vertices
        output_triangles.append([t1[0], t1[1], t1[2]])
        output_triangles.append([t2[0], t2[1], t2[2]])
        output_triangles.append([t3[0], t3[1], t3[2]])
        output_triangles.append([t4[0], t4[1], t4[2]])
        output_triangles.append([t5[0], t5[1], t5[2]])
        output_triangles.append([t6[0], t6[1], t6[2]])
        output_triangles.append([t7[0], t7[1], t7[2]])
        output_triangles.append([t8[0], t8[1], t8[2]])
        output_triangles.append([t9[0], t9[1], t9[2]])
        output_triangles.append([t10[0], t10[1], t10[2]])
        output_triangles.append([t11[0], t11[1], t11[2]])
        output_triangles.append([t12[0], t12[1], t12[2]])
        
    return output_triangles

def transform_clip_to_world(clip_vertex, world_matrix):
    # Inverse of WorldXf is not provided, but source uses WorldXf to transform from clip to world?
    # Actually, source says: "Transform the three input triangle vertices (In[0], In[1], In[2]) from clip space to world space using the WorldXf matrix"
    # This implies In[0] is in clip space? No, usually VS outputs clip space.
    # The source code does: float4 wpos1 = mul(In[0].HPosition, WorldXf);
    # In[0].HPosition is the output of the vertex shader, which is clip space.
    # Multiplying clip space by World matrix is unusual. Usually you multiply World matrix by local space.
    # However, we must follow the source.
    return mul(clip_vertex, world_matrix)

def point_in_triangle(p, a, b, c):
    # Check if p is on the same side of all edges
    def same_side(p1, p2, a, b):
        cp1 = cross(b - a, p1 - a)
        cp2 = cross(b - a, p2 - a)
        if dot(cp1, cp2) >= 0:
            return True
        return False
    
    if same_side(p, a, b, c) and same_side(p, b, a, c) and same_side(p, c, a, b):
        return True
    return False
```

**Edge Cases and Pitfalls:**
*   **Bounding Box Truncation:** The bounding box calculation uses integer casting which truncates towards zero. This may cause voxels to be missed if the triangle spans negative coordinates or if the bounding box is not tightly aligned with the grid [S1, unofficial].
*   **Distance Threshold:** The distance threshold for voxel inclusion is hardcoded to 1. This means voxels are included if their center is within distance 1 of the triangle plane, regardless of the voxel size (0.5) [S1, unofficial].
*   **Performance:** Explicitly generating cube geometry for every grid cell intersecting the triangle results in significantly higher vertex and triangle counts compared to rasterization-based methods [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity is proportional to the number of integer grid positions within the triangle's bounding box. For each position, a constant number of operations (projection, distance check, point-in-triangle test) are performed. If $N$ is the number of voxels generated, the output size is $O(N)$ triangles.
*   **Memory Complexity:** The algorithm uses a fixed-size array of 100 cubes per triangle. This limits the number of voxels that can be generated per triangle to 100. If more voxels are needed, the array size must be increased, or the algorithm must be modified to use dynamic memory or multiple passes [S1, unofficial].
*   **Strengths:** Real-time execution on DirectX hardware; generates explicit 3D geometry suitable for further processing or rendering with standard rasterization pipelines.
*   **Limitations:** High vertex and triangle counts; fixed limit on voxels per triangle; bounding box truncation issues; hardcoded parameters.

## Background
The Mesh-to-Voxel Geometry Shader was developed by Jonathan Lessard as a real-time DirectX geometry shader technique [S1, unofficial]. It is part of a collection of experimental graphics shaders and scripts aimed at converting mesh objects into voxel-like cubes for real-time applications.

## Sources
[S1] Mesh to voxel shaders and script, https://www.jonathanlessard.net/?p=18, fetched 2026-10-29 (unofficial)
