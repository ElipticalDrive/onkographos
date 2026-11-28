# Mesh-to-Voxel Geometry Shader

Also called: DirectX geometry shader voxelization, Mesh to voxel shaders

## Overview
The Mesh-to-Voxel Geometry Shader is a real-time technique that converts mesh geometry into discrete voxel-like cubes by iterating over integer grid positions within a triangle's bounding box and emitting cube geometry for positions close to the triangle plane [S1, unofficial]. Unlike rasterization-based voxelization which fills pixels, this method explicitly generates cube geometry for every grid cell intersecting the triangle, resulting in significantly higher vertex and triangle counts [S1, unofficial].

## How it works
**Inputs and Outputs:** The method takes a single triangle defined by three vertices in clip space as input to a geometry shader [S1, unofficial]. The output is a stream of triangles forming cubes centered at valid voxel positions [S1, unofficial].

**Data Structures:** The input vertex structure `appdata` contains `Position` (float3), `UV` (float4), `Normal` (float4), `Tangent` (float4), and `Binormal` (float4) [S1, unofficial]. The geometry shader output structure `geoOutput` contains `HPosition` (float4), `UV` (float2), and `Color` (float4) [S1, unofficial]. An internal array `cubes` stores up to 100 voxel centers and their distances to the triangle plane [S1, unofficial].

**Parameters:** The half-extent `size` of the generated voxel cubes is hardcoded to `0.5` [S1, unofficial]. The point light position `Lamp0Pos` defaults to `{-0.5, 2.0, 1.25}` [S1, unofficial]. The point light color `Lamp0Color` defaults to `{2.0, 2.0, 2.0}` [S1, unofficial]. The distance threshold for voxel inclusion is hardcoded to `1.0` [S1, unofficial].

**Steps:**
1. Transform the three input triangle vertices from clip space to world space using the `WorldXf` matrix [S1, unofficial].
2. Calculate the axis-aligned bounding box (AABB) of the triangle in world space by finding the minimum (`Ax`, `Ay`, `Az`) and maximum (`Bx`, `By`, `Bz`) integer coordinates via truncating the float world positions to `int` [S1, unofficial].
3. Compute the triangle's world-space normal `wNormal` by normalizing the cross product of two edge vectors [S1, unofficial].
4. Compute the plane constant `delta = dot(wpos1, wNormal)` [S1, unofficial].
5. Iterate through all integer grid positions `(x, y, z)` within the bounding box [S1, unofficial].
6. For each grid point `curPos`, project it onto the triangle plane to get `tempPos`, and compute the Euclidean distance `tempDist` [S1, unofficial].
7. If `tempDist < 1.0`, test if `curPos` lies inside the triangle [S1, unofficial]. If inside, store the voxel center and distance in the `cubes` array [S1, unofficial].
8. For each valid voxel center, define the 8 vertices of a cube with half-extent `size` [S1, unofficial].
9. Calculate diffuse lighting for each of the 6 faces by computing face normals, light vectors, and diffuse values, then scaling by `Lamp0Color` and a uniform color `(1,1,1)` [S1, unofficial].
10. Assign colors to the 12 triangles forming the cube by averaging the diffuse colors of the 3 faces each triangle belongs to, with alpha hardcoded to `1.0` [S1, unofficial].
11. Transform each vertex to view space using `ViewIXf` and emit the triangles to the stream [S1, unofficial].
12. Call `TriStream.RestartStrip()` after processing all cubes to end the primitive strip [S1, unofficial].

**Formulas:**
- Plane constant: `delta = dot(wpos1, wNormal)` [S1, unofficial].
- Point projection: `tempPos = curPos + (delta - dot(curPos, wNormal)) * wNormal` [S1, unofficial].
- Distance: `tempDist = distance(curPos, tempPos)` [S1, unofficial].

**Pitfalls:** The bounding box calculation uses integer casting that truncates towards zero, which may cause voxels to be missed if the triangle spans negative coordinates or if the bounding box is not tightly aligned with the grid [S1, unofficial]. The distance threshold for voxel inclusion is hardcoded to `1.0`, which is independent of the voxel size (`0.5`) and may include or exclude voxels inconsistently relative to the actual geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_voxel_geometry_shader(triangle_clip, world_xf, view_ixf, lamp0_pos, lamp0_color):
    # Step 1: Transform to world space
    wpos1 = mul(triangle_clip[0], world_xf)
    wpos2 = mul(triangle_clip[1], world_xf)
    wpos3 = mul(triangle_clip[2], world_xf)
    
    # Step 2: Calculate AABB (integer truncation)
    Ax = int(wpos1.x); Bx = int(wpos1.x)
    if wpos2.x < Ax: Ax = int(wpos2.x)
    if wpos2.x > Bx: Bx = int(wpos2.x)
    if wpos3.x < Ax: Ax = int(wpos3.x)
    if wpos3.x > Bx: Bx = int(wpos3.x)
    
    Ay = int(wpos1.y); By = int(wpos1.y)
    if wpos2.y < Ay: Ay = int(wpos2.y)
    if wpos2.y > By: By = int(wpos2.y)
    if wpos3.y < Ay: Ay = int(wpos3.y)
    if wpos3.y > By: By = int(wpos3.y)
    
    Az = int(wpos1.z); Bz = int(wpos1.z)
    if wpos2.z < Az: Az = int(wpos2.z)
    if wpos2.z > Bz: Bz = int(wpos2.z)
    if wpos3.z < Az: Az = int(wpos3.z)
    if wpos3.z > Bz: Bz = int(wpos3.z)
    
    # Step 3: Compute Normal
    edgeA = wpos2 - wpos1
    edgeB = wpos3 - wpos1
    wNormal = normalize(cross(edgeA, edgeB))
    
    # Step 4: Compute Delta
    delta = dot(wpos1, wNormal)
    
    cubes = []
    
    # Step 5-7: Iterate and Check
    for x in range(Ax, Bx + 1):
        for y in range(Ay, By + 1):
            for z in range(Az, Bz + 1):
                curPos = float3(x, y, z)
                tempPos = curPos + (delta - dot(curPos, wNormal)) * wNormal
                tempDist = distance(curPos, tempPos)
                
                if tempDist < 1.0:
                    if point_in_triangle(curPos, wpos1, wpos2, wpos3):
                        cubes.append((curPos, 1.0 - tempDist))
    
    size = 0.5
    output_triangles = []
    
    # Step 8-11: Generate Cubes
    for voxel_center, _ in cubes:
        # Define 8 vertices
        pos1 = voxel_center + float4(-size, size, -size, 0)
        pos2 = voxel_center + float4(-size, -size, -size, 0)
        pos3 = voxel_center + float4(size, size, -size, 0)
        pos4 = voxel_center + float4(size, -size, -size, 0)
        pos5 = voxel_center + float4(-size, size, size, 0)
        pos6 = voxel_center + float4(-size, -size, size, 0)
        pos7 = voxel_center + float4(size, size, size, 0)
        pos8 = voxel_center + float4(size, -size, size, 0)
        
        # Calculate 6 face colors
        faces = [
            (pos1, pos2, pos3), (pos3, pos7, pos4), (pos4, pos7, pos8),
            (pos1, pos5, pos6), (pos5, pos1, pos2), (pos7, pos5, pos6)]
        
        face_colors = []
        for v1, v2, v3 in faces:
            n = normalize(cross(v2 - v1, v3 - v1))
            lv = normalize(lamp0_pos - voxel_center)
            d = max(dot(lv, n), 0.0)
            face_colors.append(lamp0_color * d)
        
        # Compute per-vertex colors as the average of the 3 adjacent faces
        vertex_colors = [
            (face_colors[0] + face_colors[3] + face_colors[4]) / 3.0, # pos1
            (face_colors[0] + face_colors[3] + face_colors[5]) / 3.0, # pos2
            (face_colors[0] + face_colors[1] + face_colors[2]) / 3.0, # pos3
            (face_colors[1] + face_colors[2] + face_colors[3]) / 3.0, # pos4
            (face_colors[3] + face_colors[4] + face_colors[5]) / 3.0, # pos5
            (face_colors[4] + face_colors[5] + face_colors[0]) / 3.0, # pos6
            (face_colors[1] + face_colors[2] + face_colors[5]) / 3.0, # pos7
            (face_colors[1] + face_colors[2] + face_colors[3]) / 3.0  # pos8
        ]
        
        # Emit 12 triangles using precomputed vertex colors
        triangles = [
            (pos4, pos2, pos3), (pos2, pos4, pos7), (pos4, pos2, pos1),
            (pos2, pos1, pos5), (pos1, pos5, pos6), (pos1, pos6, pos4),
            (pos3, pos7, pos8), (pos3, pos8, pos4), (pos7, pos3, pos1),
            (pos7, pos1, pos5), (pos8, pos7, pos6), (pos8, pos6, pos4)
        ]
        
        for v1, v2, v3 in triangles:
            # Transform to view space and emit
            out1 = geoOutput(mul(v1, view_ixf), v1.xy, vertex_colors[0])
            out2 = geoOutput(mul(v2, view_ixf), v2.xy, vertex_colors[1])
            out3 = geoOutput(mul(v3, view_ixf), v3.xy, vertex_colors[2])
            output_triangles.extend([out1, out2, out3])
    
    return output_triangles
```

## Operating characteristics
Not given in the sources.

## Background
The technique was developed by Jonathan Lessard as a real-time DirectX geometry shader voxelization method, originating from experimental graphics and demo scene work [S1, unofficial]. It is used for converting mesh objects into voxel-like cubes in real-time rendering pipelines [S1, unofficial].

## Sources
[S1] Mesh to voxel shaders and script - Jonathan Lessard, https://www.jonathanlessard.net/?p=18, fetched 2026-10-29 (unofficial)
