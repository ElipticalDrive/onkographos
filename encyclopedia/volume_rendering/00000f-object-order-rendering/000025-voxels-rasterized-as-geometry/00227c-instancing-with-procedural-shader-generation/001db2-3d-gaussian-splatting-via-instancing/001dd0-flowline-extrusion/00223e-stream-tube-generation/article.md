# Stream Tube Generation

Also called: Pentagonal Stream Tube Generation, Streamlines with Tube Geometry, Streamtube Line Integration

## Overview
Stream Tube Generation solves the ambiguity of line position and orientation in still images by inflating thin stream lines into volumetric tubes [S1, unofficial]. Unlike [Flow Volumes](7690-flow-volumes.md), which render the entire field as a density, this method extrudes specific paths into mesh geometry that supports lighting and shading, making the flow structure visually distinct [S1, unofficial]. It builds on [Flowline Extrusion](1621-flowline-extrusion.md) by replacing point splatting with the generation of 3D tube meshes along the integrated path [S2, unofficial].

## How it works
**Inputs:**
- Seed points (positions) in the vector field.
- Vector field data (velocity vectors).
- Parameters: `radius` (tube radius), `geometrySampling` (integer, frequency of mesh generation), `useRk4` (boolean, integration method), `dt` (step size), `maxSteps` (maximum integration steps).
- Coordinate space multiplier `volumeCoordSpaceMult` (float3) to convert grid indices to world space.
- Magnitude Color Transfer Function texture `vectorMangitudeCtfTex` and normalization factor `mangitudeCtfNormalizeMult`.

**Outputs:**
- `outVetrices`: `float3*` array of tube vertices.
- `outVertexNormals`: `float3*` array of tube normals.
- `outVertexColors`: `float3*` array of tube colors.
- `outFaces`: `uint3*` array of tube indices (triangles).
- `outComputedSteps`: `uint*` array storing the number of valid steps per seed.

**Data Structures:**
- The tube cross-section is a regular pentagon with 5 vertices [S1, unofficial].
- Each tube segment (between two integration steps) consists of 5 vertices for the current slice and 5 for the previous slice, connected to form 10 triangles (2 triangles per pentagon side) [S1, unofficial].
- Indices are stored as `uint3` (three indices per triangle) [S2, unofficial].

**Numbered Steps:**

1.  **Initialize Bounds:** Calculate the maximum world coordinates based on volume size and coordinate space multiplier [S2, unofficial]:
    $$
    \begin{aligned}
    \text{maxWorld}.x &= \text{volumeSize}.width / \text{volumeCoordSpaceMult}.x \\
    \text{maxWorld}.y &= \text{volumeSize}.height / \text{volumeCoordSpaceMult}.y \\
    \text{maxWorld}.z &= \text{volumeSize}.depth / \text{volumeCoordSpaceMult}.z
    \end{aligned}
    $$
    Initialize `geometryStep = geometrySampling` and `step = 1` [S2, unofficial].

2.  **Integration Loop:** While `step < maxSteps`:
    a.  **Boundary Check:** If `position.x < 0` or `position.y < 0` or `position.z < 0` or `position.x > maxWorld.x` or `position.y > maxWorld.y` or `position.z > maxWorld.z`, break the loop [S2, unofficial].
    b.  **Integrate Position:**
        - If `useRk4` is true, compute `dv = rk4Integrate(position, dt, volumeCoordSpaceMult)` [S2, unofficial].
        - Else, compute `dv = eulerIntegrate(position, dt, volumeCoordSpaceMult)` [S2, unofficial].
        - Update position: `position.x += dv.x`, `position.y += dv.y`, `position.z += dv.z` [S2, unofficial].
    c.  **Geometry Generation:** Decrement `geometryStep`. If `geometryStep == 0`:
        - Reset `geometryStep = geometrySampling` [S2, unofficial].
        - **Color:** Sample magnitude color: `color = tex1D(vectorMangitudeCtfTex, dv.w * mangitudeCtfNormalizeMult)` [S2, unofficial].
        - **Vertices & Normals:** Call `createTubeBaseVertices(position, dv.xyz, radius, baseIndex, color, outVetrices, outNormals, outColors)` [S1, unofficial]. This function generates 5 vertices and normals for the current pentagonal cross-section [S1, unofficial].
        - **Indices:** Call `createTubeIndices(vertexBaseId, baseFaceId, outFaces)` [S1, unofficial]. This function generates indices connecting the current pentagon to the previous one [S1, unofficial].
        - Increment `baseIndex` by 5 and `baseFaceId` by 10 [S1, unofficial].
    d.  Increment `step` [S2, unofficial].

3.  **Finalize:** Store `outComputedSteps[id] = step / geometrySampling` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_tube_base_vertices(pos, v, radius, base_index, color, out_vertices, out_normals, out_colors):
    """
    Generates 5 vertices and normals for a pentagonal cross-section at pos.
    v is the tangent vector (velocity).
    """
    # Find a perpendicular vector to v
    x_axis = normalize(find_perpendicular(v))
    y_axis = normalize(cross(v, x_axis))
    
    # Precomputed cos/sin values for pentagon angles (0, 72, 144, 216, 288 degrees)
    # cos(0)=1, sin(0)=0
    # cos(72)=0.3090, sin(72)=0.9511
    # cos(144)=-0.8090, sin(144)=0.5878
    # cos(216)=-0.8090, sin(216)=-0.5878
    # cos(288)=0.3090, sin(288)=-0.9511
    
    angles = [
        (1.0, 0.0),
        (0.3090, 0.9511),
        (-0.8090, 0.5878),
        (-0.8090, -0.5878),
        (0.3090, -0.9511)
    ]
    
    for i in range(5):
        cos_a, sin_a = angles[i]
        # Vector in the cross-section plane
        v_sec = x_axis * cos_a + y_axis * sin_a
        
        # Normal is the direction from center to vertex
        out_normals[base_index] = v_sec
        # Vertex position is center + radius * direction
        out_vertices[base_index] = pos + v_sec * radius
        out_colors[base_index] = color
        base_index += 1

def create_tube_indices(vertex_base_id, base_face_id, out_faces):
    """
    Generates indices for 10 triangles connecting the current pentagon (vertex_base_id)
    to the previous pentagon (vertex_base_id - 5).
    """
    for i in range(5):
        i_next = (i + 1) % 5
        
        # Two triangles per quad face between pentagon sides
        # Triangle 1: current[i], current[i+1], prev[i+1]
        out_faces[base_face_id] = (vertex_base_id + i, vertex_base_id + i_next, vertex_base_id - 5 + i_next)
        base_face_id += 1
        
        # Triangle 2: current[i], prev[i], prev[i+1]
        out_faces[base_face_id] = (vertex_base_id + i, vertex_base_id - 5 + i, vertex_base_id - 5 + i_next)
        base_face_id += 1

def generate_stream_tubes(seeds, vector_field, params):
    """
    Main kernel logic for generating stream tubes.
    """
    volume_size = params.volume_size
    coord_mult = params.volume_coord_space_mult
    max_steps = params.max_steps
    geometry_sampling = params.geometry_sampling
    use_rk4 = params.use_rk4
    dt = params.dt
    radius = params.radius
    
    # Output buffers
    out_vertices = []
    out_normals = []
    out_colors = []
    out_faces = []
    out_computed_steps = []
    
    for seed in seeds:
        position = seed.position
        step = 1
        geometry_step = geometry_sampling
        base_vertex_index = len(out_vertices) // 3 # Assuming flat array
        base_face_index = len(out_faces) // 3
        
        # Store previous position for first tube segment? 
        # Note: The source code implies geometry is generated *after* integration step.
        # The first call to createTubeBaseVertices creates the first slice.
        # Subsequent calls create the next slice and indices connecting to the previous.
        # We need to track the previous vertex base index.
        prev_vertex_base_index = base_vertex_index
        
        while step < max_steps:
            # Boundary check
            max_world_x = volume_size.width / coord_mult.x
            max_world_y = volume_size.height / coord_mult.y
            max_world_z = volume_size.depth / coord_mult.z
            
            if (position.x < 0 or position.y < 0 or position.z < 0 or 
                position.x > max_world_x or position.y > max_world_y or position.z > max_world_z):
                break
            
            # Integration
            if use_rk4:
                dv = rk4_integrate(position, dt, coord_mult)
            else:
                dv = euler_integrate(position, dt, coord_mult)
            
            position.x += dv.x
            position.y += dv.y
            position.z += dv.z
            
            # Generate geometry every geometry_sampling steps
            geometry_step -= 1
            if geometry_step == 0:
                geometry_step = geometry_sampling
                
                # Color from magnitude
                magnitude = dv.w # Assuming dv.w is magnitude
                color = sample_magnitude_ctf(magnitude, params.ctf_params)
                
                # Create current slice vertices
                create_tube_base_vertices(position, dv.xyz, radius, base_vertex_index, color, out_vertices, out_normals, out_colors)
                
                # If not the first slice, create indices connecting to previous slice
                if base_vertex_index != prev_vertex_base_index:
                    create_tube_indices(base_vertex_index, base_face_index, out_faces)
                    base_face_index += 10 # 5 sides * 2 triangles
                
                prev_vertex_base_index = base_vertex_index
                base_vertex_index += 5 # 5 vertices per slice
            
            step += 1
        
        out_computed_steps.append(step // geometry_sampling)
    
    return out_vertices, out_normals, out_colors, out_faces, out_computed_steps
```

**Parameters:**
- `useRk4`: Boolean. If true, uses 4th-order Runge-Kutta integration; otherwise, uses Euler integration [S2, unofficial].
- `geometrySampling`: Integer. Controls how often tube mesh geometry is generated. Geometry is generated every `geometrySampling` integration steps [S2, unofficial].
- `mangitudeCtfNormalizeMult`: Normalization factor for vector magnitude texture lookup, calculated as `vectorMangitudeCtfLength / maxMangitude` during kernel launch [S2, unofficial].
- `radius`: Float. The radius of the stream tube [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The loop breaks if the position exceeds the volume bounds or becomes negative [S2, unofficial].
- **Adaptive Algorithms:** Adaptive integration algorithms are not supported for stream tubes due to array index management bugs in the reference implementation [S1, unofficial].
- **Memory Usage:** Stream tubes consume significantly more GPU memory than stream lines because they generate vertices, normals, and indices for faces, in addition to the integration computation [S1, unofficial].
- **Precision:** The pentagonal approximation uses hardcoded cosine/sine values (e.g., 0.3090, 0.9511) which may introduce minor visual artifacts compared to a true circle, but is deemed sufficient for smooth shading [S1, unofficial].

## Subtopics

### Flow Volumes
Renders the entire vector field as a volumetric density rather than discrete paths. Choose when you need to see the overall flow structure everywhere, not just along specific streamlines. [Flow Volumes](7690-flow-volumes.md)

## Operating characteristics
- **Time Complexity:** Integration time is the same as for stream lines. However, total time is higher due to the overhead of generating tube geometry (vertices, normals, indices) [S1, unofficial].
- **Memory Complexity:** Significantly higher than stream lines. Requires storage for vertices, normals, colors, and indices for every tube segment [S1, unofficial].
- **Performance:** On an NVIDIA Quadro K5000, 2^13 (8192) stream tubes with 2^13 (8192) samples took 240 ms [S1, unofficial].
- **Strengths:** Provides clear visual cues for flow direction and position in still images; supports lighting and shading for better depth perception [S1, unofficial].
- **Limitations:** High memory consumption; not suitable for adaptive integration in the reference implementation; more complex to render than lines [S1, unofficial].
- **When to Use:** When visual clarity of flow paths in static images is critical and GPU memory is sufficient.
- **When Not to Use:** When memory is constrained or when adaptive step sizes are required.

## Background
Stream Tube Generation was developed as part of a CUDA-based realtime visualization system for 3D vector fields. The method was introduced to address the difficulty of perceiving line orientation in still images by inflating lines into shaded tubes [S1, unofficial]. The implementation details, including the pentagonal cross-section and CUDA kernel structure, are documented in Marek Fiser's blog and the associated GitHub repository [S1, S2, unofficial].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] NightElfik/Vector-field-visualization-using-cuda, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
