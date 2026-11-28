# Stream Tube Generation

Also called: Pentagonal Stream Tube Generation, Streamlines with Tube Geometry, Streamtube Line Integration

## Overview
Stream Tube Generation is a technique for visualizing 3D vector fields by integrating seed points into streamlines and extruding them into volumetric tube geometry [S1][S3, unofficial]. It solves the ambiguity of line position and orientation in still images by inflating thin stream lines with volume, allowing for shading and better spatial perception [S2, unofficial]. It sets itself apart from [Flow Volumes](7690-flow-volumes.md) by generating explicit geometric primitives (triangles) rather than volume textures, and from [Fabric Tube Rendering](10641-fabric-tube-rendering.md) by constructing actual 3D mesh geometry rather than relying on image-space post-processing [S1].

## How it works
**Inputs:**
- Volumetric data set $\Omega \subset \mathbb{R}^3$ and associated vector field $V: \Omega \rightarrow \mathbb{R}^3$ [S1].
- Seed points $p$ [S1].
- Integration step size $\varepsilon$ [S1].
- Tube radius $r$ [S1].
- `useRk4`: Boolean flag selecting between Runge-Kutta 4th order (`rk4Integrate`) and Euler integration (`eulerIntegrate`) [S3, unofficial].
- `geometrySampling`: Integer controlling frequency of tube mesh generation (geometry generated every `geometrySampling` steps) [S3, unofficial].
- `mangitudeCtfNormalizeMult`: Normalization factor for vector magnitude texture lookup [S3, unofficial].

**Outputs:**
- `outVertices` (`float3*`): Tube vertices [S3, unofficial].
- `outVertexNormals` (`float3*`): Tube normals [S3, unofficial].
- `outVertexColors` (`float3*`): Tube colors [S3, unofficial].
- `outFaces` (`uint3*`): Tube indices [S3, unofficial].
- `outComputedSteps` (`uint*`): Step count per seed [S3, unofficial].

**Data Structures:**
- The tube cross-section is a regular pentagon with 5 vertices [S2, unofficial].
- Indices are generated per segment connecting two pentagonal slices [S2, unofficial].

**Steps:**

1. **Integration:** Solve the ODE $\frac{\partial x(t)}{\partial t} = V(x(t))$ with $x(0) = p$ to generate a streamline [S1]. Discretize using Euler's method:
   $$x_{i+1} = x_i + \varepsilon V(x_i)$$
   where $\varepsilon$ is a small value appropriate to the data set $\Omega$ to reduce integration error [S1]. If `useRk4` is true, use 4th-order Runge-Kutta instead [S3, unofficial].

2. **Boundary Check:** Initialize `maxWorld` bounds:
   ```python
   maxWorld.x = volumeSize.width / volumeCoordSpaceMult.x
   maxWorld.y = volumeSize.height / volumeCoordSpaceMult.y
   maxWorld.z = volumeSize.depth / volumeCoordSpaceMult.z
   ```
   If `position.x < 0` or `position.y < 0` or `position.z < 0` or `position.x > maxWorld.x` or `position.y > maxWorld.y` or `position.z > maxWorld.z`, break the loop [S3, unofficial].

3. **Geometry Generation Trigger:** Maintain a counter `geometryStep`. If `geometryStep == 0`, reset `geometryStep = geometrySampling` and generate geometry for the current position [S3, unofficial].

4. **Color Lookup:** Sample magnitude color:
   ```python
   color = tex1D(vectorMangitudeCtfTex, (float)(dv.w * mangitudeCtfNormalizeMult))
   ```
   where `dv.w` is the magnitude component from the integration result [S3, unofficial].

5. **Vertex Generation:** Call `createTubeBaseVertices` with `position`, velocity vector `v` (from integration), `radius`, `baseIndex`, and `color` [S3, unofficial]. This function computes the pentagonal cross-section in the plane defined by the velocity vector [S2, unofficial].

6. **Index Generation:** Call `createTubeIndices` with `vertexBaseId` and `baseFaceId` to generate triangles connecting the current pentagon to the previous one [S3, unofficial].

7. **Loop:** Increment step counter. If `step < maxSteps`, continue integration. Otherwise, store `outComputedSteps[id] = step / geometrySampling` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_and_generate_tubes(seed_position, volume_field, volume_size, volume_coord_space_mult, 
                                 radius, use_rk4, geometry_sampling, magnitude_ctf_normalize_mult):
    """
    Generates stream tube geometry for a single seed.
    Returns vertices, normals, colors, indices, and step count.
    """
    # Output buffers
    vertices = []
    normals = []
    colors = []
    indices = []
    
    # State
    position = seed_position
    step = 0
    geometry_step = 0
    
    # Pre-compute max world bounds
    max_world_x = volume_size.width / volume_coord_space_mult.x
    max_world_y = volume_size.height / volume_coord_space_mult.y
    max_world_z = volume_size.depth / volume_coord_space_mult.z
    
    # Initial vertex index and face index
    vertex_base_id = 0
    face_base_id = 0
    
    # Generate first tube slice at seed position? 
    # Note: Sources imply geometry is generated after integration steps or at specific intervals.
    # We assume the first slice is generated at the start or after the first integration step.
    # Based on S3, geometry is generated when geometry_step == 0.
    
    while step < 1000: # maxSteps placeholder, typically determined by boundary or fixed limit
        # Boundary check
        if (position.x < 0 or position.y < 0 or position.z < 0 or 
            position.x > max_world_x or position.y > max_world_y or position.z > max_world_z):
            break
            
        # Integration
        if use_rk4:
            dv = rk4_integrate(position, 1.0, volume_field, volume_coord_space_mult) # dt=1.0 assumed or scaled
        else:
            dv = euler_integrate(position, 1.0, volume_field, volume_coord_space_mult)
            
        position.x += dv.x
        position.y += dv.y
        position.z += dv.z
        
        # Decrement geometry step
        geometry_step -= 1
        
        if geometry_step == 0:
            # Reset geometry step
            geometry_step = geometry_sampling
            
            # Sample color
            magnitude = dv.w # Assuming dv.w holds magnitude
            color = sample_magnitude_texture(magnitude * magnitude_ctf_normalize_mult)
            
            # Generate tube slice vertices
            v = float3(dv.x, dv.y, dv.z)
            base_index = len(vertices)
            
            # Create pentagonal cross-section
            create_tube_base_vertices(position, v, radius, base_index, color, vertices, normals, colors)
            
            # If not the first slice, generate indices connecting to previous slice
            if vertex_base_id > 0:
                # Previous slice started at (vertex_base_id - 5)
                create_tube_indices(vertex_base_id, face_base_id, indices)
                face_base_id += 10 # 5 faces per pentagon side * 2 triangles
                
            vertex_base_id = len(vertices)
            
        step += 1
        
    return vertices, normals, colors, indices, step

def create_tube_base_vertices(pos, v, radius, base_index, color, out_vertices, out_normals, out_colors):
    """
    Generates 5 vertices for a pentagonal cross-section at pos, oriented along v.
    """
    # Find perpendicular vector to v
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
        # Normal is the direction of the vertex from center
        normal = x_axis * cos_a + y_axis * sin_a
        out_normals.append(normal)
        
        # Vertex position
        vertex = pos + normal * radius
        out_vertices.append(vertex)
        
        # Color
        out_colors.append(color)

def create_tube_indices(vertex_base_id, base_face_id, out_indices):
    """
    Generates indices for the 5 quads (2 triangles each) connecting the current pentagon
    (starting at vertex_base_id) to the previous pentagon (starting at vertex_base_id - 5).
    """
    for i in range(5):
        i_next = (i + 1) % 5
        
        # Triangle 1: current[i], current[i_next], prev[i_next]
        out_indices.append((vertex_base_id + i, vertex_base_id + i_next, vertex_base_id - 5 + i_next))
        
        # Triangle 2: current[i], prev[i], prev[i_next]
        out_indices.append((vertex_base_id + i, vertex_base_id - 5 + i, vertex_base_id - 5 + i_next))

def find_perpendicular(v):
    """
    Finds a vector perpendicular to v.
    [G] Simple heuristic: if v is not parallel to x-axis, use cross(v, (1,0,0)), else cross(v, (0,1,0)).
    """
    if abs(v.x) < 0.9:
        return normalize(cross(v, float3(1, 0, 0)))
    else:
        return normalize(cross(v, float3(0, 1, 0)))

def euler_integrate(pos, dt, field, mult):
    """
    Euler integration step.
    """
    # Sample field at pos
    vel = sample_vector_field(pos, field)
    # Scale by dt and mult
    dv = vel * dt * mult
    # Add magnitude to w component (assuming field returns float4 or we append magnitude)
    # Note: S3 uses dv.w for magnitude lookup. We assume the integration function returns magnitude in w.
    mag = length(vel)
    return float3(dv.x, dv.y, dv.z), mag

def rk4_integrate(pos, dt, field, mult):
    """
    Runge-Kutta 4th order integration step.
    """
    # [G] Standard RK4 implementation for ODE dx/dt = V(x)
    k1 = field(pos)
    k2 = field(pos + 0.5 * dt * k1)
    k3 = field(pos + 0.5 * dt * k2)
    k4 = field(pos + dt * k3)
    
    new_pos = pos + (dt / 6.0) * (k1 + 2*k2 + 2*k3 + k4)
    
    # Approximate magnitude for color
    vel = (k1 + 2*k2 + 2*k3 + k4) / 6.0
    mag = length(vel)
    
    return new_pos, mag
```

**Edge Cases and Pitfalls:**
- **Frenet Frame Twist:** If using a swept circle along a B-spline curve $\gamma(u)$, the normal of the Frenet frame flips at inflection points, causing severe twists in the surface $\sigma(u, v)$ [S1]. The CUDA implementation avoids this by constructing the tube locally in the plane defined by the velocity vector at each step, rather than using a global Frenet frame [S2, unofficial].
- **Self-Intersections:** To avoid self-intersections in the stream tube, the radius must be smaller than $1/\kappa_{max}$, where $\kappa_{max}$ is the maximum curvature of the curve [S1].
- **Memory Usage:** Stream tubes consume significantly more GPU memory than stream lines due to the computation of indices, normals, and the increased vertex count [S2, unofficial].

## Subtopics

### [Fabric Tube Rendering](10641-fabric-tube-rendering.md)
A post-processing technique that interprets a composited image of a tensor field as streamtubes on the surface without creating actual 3D geometry. It uses binormals derived from eigenvectors and image-plane normals to determine tube position and width, creating a tube-like effect with proper spatial impression. It sets itself apart from [Stream Tube Generation](2707-stream-tube-generation.md) by avoiding explicit geometric construction and integration along the field, instead relying on local image-space analysis to compute shading normals.

### [Edge Enhanced Transparent Streamtubes](16713-edge-enhanced-transparent-streamtubes.md)
Edge Enhanced Transparent Streamtubes is a hardware-accelerated rendering technique that represents fibre tracts as semi-transparent tubes with emphasised silhouettes. It improves the identification of individual fibre tracts and their 3D arrangement by revealing inner structures and softening silhouette edges. Unlike standard polygon transparency which lacks real-world attenuation effects, this method approximates the apparent change in transparency at silhouette edges of real objects, reducing visual cluttering and aliasing effects in densely packed streamlines.

## Operating characteristics
- **Time Complexity:** Integration is $O(N)$ where $N$ is the number of steps per streamline. Geometry generation is proportional to the number of steps divided by `geometrySampling` [S3, unofficial].
- **Memory Complexity:** Significantly higher than stream lines. Requires storage for vertices, normals, colors, and indices for all tubes [S2, unofficial].
- **Performance:** On an NVIDIA Quadro K5000, $2^{13}$ stream tubes with $2^{13}$ samples took 240 ms [S2, unofficial]. This is comparable to stream lines in integration time, but the geometry generation and memory bandwidth for normals/indices add overhead [S2, unofficial].
- **Strengths:** Provides clear 3D orientation and position of flow paths; allows for shading and lighting effects; smooth shading with pentagonal cross-sections is visually sufficient [S2, unofficial].
- **Limitations:** High memory consumption; adaptive algorithms are difficult to implement due to index management bugs [S2, unofficial]; requires careful radius selection to avoid self-intersections [S1].

## Background
Stream Tube Generation was developed as a method to visualize 3D vector fields by solving the ODE $\frac{\partial x(t)}{\partial t} = V(x(t))$ and constructing tube geometry from the resulting streamlines [S1]. The approach was detailed in research on ambient occlusion effects for combined volumes and tubular geometry [S1]. Practical CUDA implementations for real-time visualization have been documented in academic blogs and open-source repositories [S2, S3, unofficial].

## Sources
[S1] Ambient Occlusion Effects for Combined Volumes and Tubular Geometry, https://doi.org/10.1109/tvcg.2012.306, fetched 2026-11-08
[S2] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S3] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
