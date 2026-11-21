# SDFPerf Raymarching Shader

Also called: Node-based SDF Editor Raymarching, SDFPerf

## Overview
The SDFPerf Raymarching Shader is a fragment shader generated from a node-based graph editor that constructs Signed Distance Functions (SDFs) using basic primitives, CSG operations, and domain manipulations [S1, unofficial]. It differs from generic sphere tracing implementations by providing a specific, optimized set of domain transformation operators (bend, twist) and shading modes (ambient occlusion, depth, normals) directly integrated into the generated code, rather than relying on external scene graphs or complex hierarchical structures [S1, unofficial]. It builds on [Distance-field ray marching](21-distance-field-ray-marching.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `vs_texcoord` (vec2, location 0): Vertex shader texture coordinates, typically in range [0, 1].
    *   `u_camera_position` (vec3): Camera origin in world space.
    *   `u_camera_front` (vec3): Camera forward direction vector.
    *   `u_shading` (uint): Enum selecting the shading mode (0-4).
    *   `u_time` (float): Current time in seconds.
    *   `params_block` (SSBO, binding 0, std430): A variable-size array of `vec4` parameters used by domain manipulation nodes (e.g., bend angle, twist angle, primitive dimensions) [S1, unofficial].
*   **Outputs:**
    *   `o_color` (vec4, location 0): The final rendered color.

**Data Structures:**
*   **Params Block:** Defined as `layout (std430, binding = 0) buffer params_block { vec4 params[]; };`. The shader accesses specific `vec4` elements based on the node graph connections. For example, domain manipulation nodes read angles or scales from this buffer [S1, unofficial].
*   **Hit Info:** A custom structure or tuple containing `id` (primitive identifier) and `distance` (SDF value) returned by the `map(p)` function [S1, unofficial].

**Parameters:**
*   `MAX_STEPS`: 256 (maximum iterations per ray) [S1, unofficial].
*   `MAX_TRACE_DISTANCE`: 64.0 (maximum ray length) [S1, unofficial].
*   `MIN_HIT_DISTANCE`: 0.001 (convergence threshold for surface hit) [S1, unofficial].

**SDF Primitives and Operations:**
The generated shader evaluates a composite SDF function `map(p)` which returns the distance to the nearest surface. The following formulas are used for basic primitives and operations [S1, unofficial]:

1.  **Sphere:** Center $c$, Radius $r$.
    $$ d = \|p - c\| - r $$
2.  **Box:** Half-extents $b = (b_x, b_y, b_z)$.
    $$ d = \min(\max(d_x, \max(d_y, d_z)), 0.0) + \|\max(d, 0.0)\| $$
    where $d = |p| - b$.
3.  **Plane:** Height $h$ (normal along Y).
    $$ d = p_y - h $$
4.  **Torus:** Major radius $t_x$, Minor radius $t_y$.
    $$ d = \sqrt{(\|p_{xz}\| - t_x)^2 + p_y^2} - t_y $$
5.  **Union:**
    $$ d = \min(d_a, d_b) $$
6.  **Intersection:**
    $$ d = \max(d_a, d_b) $$
7.  **Subtraction:**
    $$ d = \max(-d_a, d_b) $$
8.  **Smooth Minimum:** Parameter $k$.
    $$ h = \text{clamp}(0.5 + 0.5 \frac{b - a}{k}, 0.0, 1.0) $$
    $$ d = \text{mix}(b, a, h) - k \cdot h \cdot (1.0 - h) $$

**Domain Manipulations:**
These functions transform the query point $p$ before evaluation, effectively deforming the space [S1, unofficial].

1.  **Twist:** Rotate XZ plane around Y axis by angle $t \cdot p_y$.
    $$ c = \cos(t \cdot p_y), \quad s = \sin(t \cdot p_y) $$
    $$ m = \begin{bmatrix} c & -s \\ s & c \end{bmatrix} $$
    $$ q = (m \cdot p_{xz}, p_y) $$
    Return $q$.
2.  **Bend:** Rotate XY plane around Z axis by angle $t \cdot p_y$.
    $$ c = \cos(t \cdot p_y), \quad s = \sin(t \cdot p_y) $$
    $$ m = \begin{bmatrix} c & -s \\ s & c \end{bmatrix} $$
    $$ q = (m \cdot p_{xy}, p_z) $$
    Return $q$.

**Algorithm Steps:**

1.  **Ray Generation:**
    *   Convert `vs_texcoord` to UV range [-1, 1].
    *   Compute camera right vector from `u_camera_front` and up vector.
    *   Calculate pixel position using camera position, front, right, up vectors, and a fixed FOV of 50 degrees.
    *   Construct ray $r$ with origin `u_camera_position` and normalized direction [S1, unofficial].

2.  **Raymarching Loop:**
    *   Initialize `res.id = -1`, `total_distance = 0`, `total_steps = 0` [S1, unofficial].
    *   Loop while `total_steps < MAX_STEPS`:
        *   Calculate current point $p = r.o + r.d \times \text{total\_distance}$.
        *   Evaluate $hit\_info = \text{map}(p)$ [S1, unofficial].
        *   Add $hit\_info.distance$ to `total_distance`.
        *   If $hit\_info.distance < \text{MIN\_HIT\_DISTANCE}$:
            *   Set `res.id = hit_info.id`.
            *   Break loop.
        *   If $\text{total\_distance} > \text{MAX\_TRACE\_DISTANCE}$:
            *   Reset `total_distance = 0.0`.
            *   Break loop (discarding the ray) [S1, unofficial].
        *   Increment `total_steps`.

3.  **Normal Estimation:**
    *   If a hit occurred, compute the normal $n$ using central finite differences with epsilon $\epsilon = 0.001$ [S1, unofficial]:
        $$ e = (0.001, 0.0, 0.0) $$
        $$ n_x = \text{map}(p + e_{xyy}).y - \text{map}(p - e_{xyy}).y $$
        $$ n_y = \text{map}(p + e_{yxy}).y - \text{map}(p - e_{yxy}).y $$
        $$ n_z = \text{map}(p + e_{yyx}).y - \text{map}(p - e_{yyx}).y $$
        $$ n = \text{normalize}(n_x, n_y, n_z) $$
    *   Note: The source code snippet uses `.y` to access the distance component of the hit info tuple, implying the SDF returns a vec2 or similar where the second component is the distance [S1, unofficial].

4.  **Ambient Occlusion (AO):**
    *   Compute AO using 5 steps along the normal [S1, unofficial]:
        $$ \text{attenuation} = 0.5 $$
        $$ \text{accum} = 0.0 $$
        $$ \text{scale} = 1.0 $$
        $$ \text{ao} = 1.0 $$
        For `step` from 0 to 4:
            $$ hr = 0.01 + 0.02 \times (\text{step}^2) $$
            $$ aopos = p + n \times hr $$
            $$ dist = \text{map}(aopos).y $$
            $$ ao = -(dist - hr) $$
            $$ \text{accum} += ao \times \text{scale} $$
            $$ \text{scale} *= \text{attenuation} $$
        $$ \text{ao} = 1.0 - \text{clamp}(\text{accum}, 0.0, 1.0) $$

5.  **Shading:**
    *   Based on `u_shading` mode [S1, unofficial]:
        *   **0 (Depth):** $\sqrt{\frac{p_z}{\text{MAX\_TRACE\_DISTANCE}}}$ (Note: Source says `hit.z`, likely referring to distance or depth component).
        *   **1 (Steps):** Map `total_steps` percentage to colors using `smoothstep`.
        *   **2 (AO):** $\text{pow}(\text{ao}, 3.0)$.
        *   **3 (Normals):** $n \times 0.5 + 0.5$.
        *   **4 (Diffuse):** Diffuse lighting with light at $(0, 2, 3)$ and AO $\text{pow}(\text{ao}, 3.0)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_STEPS = 256
MAX_TRACE_DISTANCE = 64.0
MIN_HIT_DISTANCE = 0.001
FOV = 50.0 # Degrees
LIGHT_POS = vec3(0.0, 2.0, 3.0)

# Helper: SDF Primitives
def sdf_sphere(p, center, radius):
    return length(center - p) - radius

def sdf_box(p, b):
    d = abs(p) - b
    return min(max(d.x, max(d.y, d.z)), 0.0) + length(max(d, 0.0))

def sdf_plane(p, h):
    return p.y - h

def sdf_torus(p, t):
    d = vec2(length(p.xz) - t.x, p.y)
    return length(d) - t.y

# Helper: CSG Operations
def sdf_union(a, b):
    return min(a, b)

def sdf_intersection(a, b):
    return max(a, b)

def sdf_subtraction(a, b):
    return max(-a, b)

def sdf_smooth_min(a, b, k):
    h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0)
    return mix(b, a, h) - k * h * (1.0 - h)

# Helper: Domain Manipulations
def domain_twist(p, t):
    c = cos(t * p.y)
    s = sin(t * p.y)
    m = mat2(c, -s, s, c)
    q = vec3(m * p.xz, p.y)
    return q

def domain_bend(p, t):
    c = cos(t * p.y)
    s = sin(t * p.y)
    m = mat2(c, -s, s, c)
    q = vec3(m * p.xy, p.z)
    return q

# Helper: Map Function (Composite SDF)
# Note: In the actual shader, this is generated code. 
# Here we assume a simple scene for demonstration.
def map(p):
    # Example: A twisted box
    p_twisted = domain_twist(p, 0.5) # t from params_block
    d = sdf_box(p_twisted, vec3(1.0, 1.0, 1.0))
    return vec2(0.0, d) # id=0, distance=d

# Helper: Normal Estimation
def get_normal(p):
    e = vec3(0.001, 0.0, 0.0)
    n_x = map(p + e.xyy).y - map(p - e.xyy).y
    n_y = map(p + e.yxy).y - map(p - e.yxy).y
    n_z = map(p + e.yyx).y - map(p - e.yyx).y
    return normalize(vec3(n_x, n_y, n_z))

# Helper: Ambient Occlusion
def get_ao(p, n):
    attenuation = 0.5
    accum = 0.0
    scale = 1.0
    ao = 1.0
    for step in range(5):
        hr = 0.01 + 0.02 * float(step * step)
        aopos = p + n * hr
        dist = map(aopos).y
        ao_val = -(dist - hr)
        accum += ao_val * scale
        scale *= attenuation
    return 1.0 - clamp(accum, 0.0, 1.0)

# Main Raymarching
def raymarch(ray_origin, ray_dir):
    res_id = -1
    total_distance = 0.0
    total_steps = 0
    
    while total_steps < MAX_STEPS:
        p = ray_origin + ray_dir * total_distance
        hit_info = map(p)
        dist = hit_info.y
        
        total_distance += dist
        
        if dist < MIN_HIT_DISTANCE:
            res_id = hit_info.x
            break
            
        if total_distance > MAX_TRACE_DISTANCE:
            total_distance = 0.0
            break
            
        total_steps += 1
        
    return res_id, total_distance, total_steps

# Fragment Shader Entry
def fragment_shader(vs_texcoord):
    # 1. Ray Generation
    uv = vs_texcoord * 2.0 - 1.0
    aspect = 1.0 # Assume square or handle aspect ratio
    fov_rad = radians(FOV)
    ray_dir = normalize(vec3(uv.x, uv.y / aspect, 1.0 / tan(fov_rad / 2.0)))
    
    # Transform ray direction to world space if necessary
    # Note: Source simplifies this to camera vectors
    # ray_dir = normalize(u_camera_front + uv.x * u_camera_right + uv.y * u_camera_up)
    
    ray_origin = u_camera_position
    
    # 2. March
    hit_id, hit_dist, steps = raymarch(ray_origin, ray_dir)
    
    # 3. Shading
    if hit_id != -1:
        p_hit = ray_origin + ray_dir * hit_dist
        n = get_normal(p_hit)
        ao = get_ao(p_hit, n)
        
        if u_shading == 0: # Depth
            color = sqrt(p_hit.z / MAX_TRACE_DISTANCE)
        elif u_shading == 1: # Steps
            color = smoothstep(0.0, float(MAX_STEPS), float(steps))
        elif u_shading == 2: # AO
            color = pow(ao, 3.0)
        elif u_shading == 3: # Normals
            color = n * 0.5 + 0.5
        elif u_shading == 4: # Diffuse
            l = normalize(LIGHT_POS - p_hit)
            diff = max(dot(n, l), 0.0)
            color = diff * ao * 3.0 # Simplified lighting
        else:
            color = vec3(0.0)
    else:
        color = vec3(0.0) # Background
        
    return color
```

## Subtopics

### Ray-marched Shadows
Adds shadow rays to the SDF scene for accurate occlusion from light sources [1087-ray-marched-shadows.md].

### Raymarching with Depth Buffer Occlusion
Uses the depth buffer to occlude raymarched geometry, improving performance and visual coherence [1880-raymarching-with-depth-buffer-occlusion.md].

### SDF Depth Buffer Integration
Integrates SDF raymarching with existing depth buffers for hybrid rendering [1955-sdf-depth-buffer-integration.md].

### Lighting and Shading Models
Discusses various shading models (Phong, Blinn-Phong, PBR) applicable to SDF surfaces [2055-lighting-and-shading-models.md].

### Surface Point Sampling for SDFs
Techniques for sampling points on the SDF surface for texturing or displacement [2708-surface-point-sampling-for-sdfs.md].

### Chamfer-L1 Distance Computation
Uses L1 distance for faster computation, trading smoothness for speed [2752-chamfer-l1-distance-computation.md].

### Raymarched SDF Volume Rendering
Extends SDF raymarching to volumetric data, treating SDFs as density fields [3663-raymarched-sdf-volume-rendering.md].

### Pyramid Recursive Ray Marching
Uses a hierarchical approach to accelerate raymarching [3802-pyramid-recursive-ray-marching.md].

### Deformation and Transformation Handling
General techniques for handling dynamic transformations in SDFs [3850-deformation-and-transformation-handling.md].

### Sphere Tracing with Composite SDF and Adaptive Sampling
Combines composite SDFs with adaptive step sizes for better quality [3867-sphere-tracing-with-composite-sdf-and-adaptive-sampling.md].

### Lidar Sphere Tracing
Adapts sphere tracing for LiDAR data simulation [3889-lidar-sphere-tracing.md].

### Primary Ray Integrator
The core integrator for primary rays in path tracers, often using SDFs [4401-primary-ray-integrator.md].

### SDF Normal Estimation via Finite Differences
Detailed analysis of normal estimation techniques [4619-sdf-normal-estimation-via-finite-differences.md].

### Antialiasing (AA)
Techniques for antialiasing SDF raymarching, such as supersampling or analytical AA [4802-antialiasing-aa.md].

### Volumetric and Atmospheric Rendering
Rendering atmospheric effects using SDFs or raymarching [4968-volumetric-and-atmospheric-rendering.md].

### Acceleration and Optimization
General acceleration structures and optimizations for raymarching [6222-acceleration-and-optimization.md].

### Sphere Tracing with Represent Hit
Using a "represent hit" technique to handle complex CSG operations efficiently [6684-sphere-tracing-with-represent-hit.md].

### Ray Marching with Blinn-Phong Lighting
Specific implementation of Blinn-Phong shading in raymarching [6898-ray-marching-with-blinn-phong-lighting.md].

### Dynamic SDF Ray Marching with Structured Buffers
Using GPU structured buffers for dynamic SDF data [7434-dynamic-sdf-ray-marching-with-structured-buffers.md].

### Sphere Tracing and Step Rules
Different step rules and convergence criteria for sphere tracing [9643-sphere-tracing-and-step-rules.md].

### Gridhopping
A technique for accelerating raymarching in grid-based SDFs [10027-gridhopping.md].

### Trace-based SDF Rendering
Rendering SDFs using trace-based methods for specific visual effects [10785-trace-based-sdf-rendering.md].

### Procedural and Mathematical Surfaces
Creating complex surfaces using mathematical functions and SDFs [13400-procedural-and-mathematical-surfaces.md].

### Neural and Implicit Representations
Using neural networks to represent SDFs [15012-neural-and-implicit-representations.md].

### WebGL2 Volume Raycasting
Implementing volume raycasting in WebGL2, often using SDFs [15994-webgl2-volume-raycasting.md].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the average number of steps per ray (up to `MAX_STEPS`). The complexity of the `map(p)` function depends on the number of primitives and operations in the node graph.
*   **Memory Complexity:** $O(1)$ per pixel, plus $O(P)$ for the `params_block` where $P$ is the number of parameters. No large textures or acceleration structures are required.
*   **Measured Performance:** Tested on NVIDIA GeForce GTX 970M and GTX 980 [S1, unofficial]. Performance is highly dependent on the complexity of the SDF graph and the number of steps. The fixed `MAX_STEPS` of 256 and `MAX_TRACE_DISTANCE` of 64.0 provide a predictable upper bound on computation.
*   **Scaling:** Scales linearly with resolution. Does not scale well with increasing SDF complexity (more primitives/operations) due to the lack of spatial acceleration structures.
*   **Strengths:** Simple implementation, no need for external scene graphs or BVH construction, easy to edit via node-based interface, good for real-time procedural geometry.
*   **Limitations:** No spatial acceleration (slow for complex scenes), fixed step count can lead to aliasing or slow convergence, limited to convex primitives and simple CSG, no native support for shadows or reflections without additional raymarching passes.
*   **When to use:** Real-time visualization of procedural geometry, educational purposes, simple VFX, scenes with low primitive count.
*   **When not to use:** Complex scenes with many primitives, scenes requiring shadows/reflections, high-resolution rendering where aliasing is unacceptable.

## Background
The SDFPerf project was created by mwalczyk as a work-in-progress node-based editor for creating and manipulating signed-distance fields [S1, unofficial]. It was tested on Windows 8.1, Windows 10, and Ubuntu 18.04, using NVIDIA GeForce GTX 970M and GTX 980 GPUs, and built with Rust compiler version 1.37.0-nightly [S1, unofficial]. The project generates a raymarching shader string from a graph of operators, supporting basic SDF primitives, domain manipulation, and CSG operations [S1, unofficial].

## Sources
[S1] mwalczyk/sdfperf, https://github.com/mwalczyk/sdfperf, fetched 2026-10-27 (unofficial)
