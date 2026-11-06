# Neon Raymarcher

Also called: Procedural Synthwave City Renderer, SDF Ray Marching in WebGL2

## Overview
Neon Raymarcher is a real-time WebGL2 renderer that generates an infinite procedural synthwave city using signed-distance fields (SDFs) within a single fullscreen fragment shader, requiring no meshes, textures, or scene graph [S1, unofficial]. It distinguishes itself from general-purpose SDF renderers by enforcing a strict deterministic hashing contract between CPU and GPU to ensure consistency between the rendered visual scene and the underlying game logic (e.g., collision detection) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera state (position, orthonormal basis vectors: right, up, forward), time, resolution, and fixed procedural constants (road parameters, block dimensions).
*   **Outputs:** A color buffer representing the rendered frame.

**Data Structures:**
*   **Ray Basis:** A structure containing `position` (vec3), `right` (vec3), `up` (vec3), and `forward` (vec3).
*   **Cell Coordinates:** Integer triplets `(cx, cy, cz)` representing the grid cell of a building block.
*   **Hash Seed:** A 32-bit unsigned integer derived from cell coordinates.

**Parameters:**
*   `MAX_STEPS`: 128 [S1, unofficial].
*   `MAX_DIST`: 180.0 [S1, unofficial].
*   `BLOCK_WIDTH`: 13.0 [S1, unofficial].
*   `BLOCK_DEPTH`: 14.0 [S1, unofficial].
*   `MARGIN`: 1.5 [S1, unofficial].
*   Road Amplitudes: `RD_A1=6.0`, `RD_A2=3.0`, `RD_A3=1.2` [S1, unofficial].
*   Road Frequencies: `RD_F1=0.03141593`, `RD_F2=0.06613879`, `RD_F3=0.11424174` [S1, unofficial].
*   Road Phases: `RD_P2=1.7`, `RD_P3=4.2` [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Create a WebGL2 context with antialiasing disabled and alpha disabled. Generate a fullscreen triangle using `gl_VertexID` and compile/link a vertex shader and fragment shader [S1, unofficial].
2.  **Frame Loop:** On each frame, calculate delta time `dt` (clamped to 0.1s to handle tab switches) and update the camera position along the procedural road [S1, unofficial].
3.  **Camera Basis:** Compute the camera's orthonormal basis. The camera position is defined by the road centerline at the current `z` coordinate. The forward vector points to a lookahead position on the road. The right vector is the normalized cross product of forward and world up. The up vector is the normalized cross product of right and forward [S1, unofficial].
4.  **Ray Generation:** For each pixel, compute the ray origin `o` from the camera position and the ray direction `d` using the pixel UV coordinates, resolution, and the camera basis vectors [S1, unofficial].
5.  **Sphere Tracing:** Iterate up to `MAX_STEPS` or until the accumulated distance `t` exceeds `MAX_DIST` [S1, unofficial].
    *   Query the scene SDF at the current point `p = o + t*d`.
    *   Advance `t` by the distance value `d_val = SDF(p)` [S1, unofficial].
    *   If `d_val < epsilon` (convergence threshold), record the hit point and normal, then break [S1, unofficial].
6.  **Scene SDF Evaluation:** The scene SDF is the union of the ground plane and a domain-repeated array of building boxes [S1, unofficial].
    *   **Road Centerline:** Compute the lateral offset `roadX(z)` using the sum of three sine waves [S1, unofficial]:
        $$ \text{roadX}(z) = \text{RD\_A1} \sin(z \cdot \text{RD\_F1}) + \text{RD\_A2} \sin(z \cdot \text{RD\_F2} + \text{RD\_P2}) + \text{RD\_A3} \sin(z \cdot \text{RD\_F3} + \text{RD\_P3}) $$
    *   **Building Placement:** Determine the cell coordinates `(cx, cy, cz)` for the query point relative to the road. The buildings are tiled with `BLOCK_WIDTH` and `BLOCK_DEPTH` [S1, unofficial].
    *   **Hashing:** Compute a deterministic 32-bit hash seed for the cell using the `hashCell` function [S1, unofficial].
    *   **Primitive SDF:** Evaluate the SDF for a box primitive centered at the cell location, scaled and translated according to the hash-derived footprint and height [S1, unofficial]. The box is placed with a setback from the road centerline defined by the margin [S1, unofficial].
    *   **Combination:** The final scene SDF is the minimum of the ground plane SDF and the building SDF [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from [S1, unofficial]
MAX_STEPS = 128
MAX_DIST = 180.0
BLOCK_WIDTH = 13.0
BLOCK_DEPTH = 14.0
MARGIN = 1.5
RD_A1 = 6.0
RD_A2 = 3.0
RD_A3 = 1.2
RD_F1 = 0.03141593
RD_F2 = 0.06613879
RD_F3 = 0.11424174
RD_P2 = 1.7
RD_P3 = 4.2
WORLD_UP = [0, 1, 0]

def uhash(x):
    """Deterministic 32-bit integer hash [S1, unofficial]."""
    x = x & 0xFFFFFFFF
    x ^= (x >> 16)
    x = (x * 0x7feb352d) & 0xFFFFFFFF
    x ^= (x >> 15)
    x = (x * 0x846ca68b) & 0xFFFFFFFF
    x ^= (x >> 16)
    return x & 0xFFFFFFFF

def hash_cell(cx, cy, cz):
    """Hash 3D cell coordinates to 32-bit seed [S1, unofficial]."""
    h = uhash(cx * 0x9e3779b9)
    h = uhash(h ^ (cy * 0x85ebca6b))
    h = uhash(h ^ (cz * 0xc2b2ae35))
    return h & 0xFFFFFFFF

def road_x(z):
    """Analytic road centerline [S1, unofficial]."""
    return (RD_A1 * math.sin(z * RD_F1) +
            RD_A2 * math.sin(z * RD_F2 + RD_P2) +
            RD_A3 * math.sin(z * RD_F3 + RD_P3))

def normalize(v):
    """Normalize vector v."""
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0: return [0, 0, 0]
    return [v[0]/length, v[1]/length, v[2]/length]

def sub(a, b):
    return [a[0]-b[0], a[1]-b[1], a[2]-b[2]]

def cross(a, b):
    return [a[1]*b[2] - a[2]*b[1],
            a[2]*b[0] - a[0]*b[2],
            a[0]*b[1] - a[1]*b[0]]

def get_camera_basis(camera_z, look_ahead, height, look_height):
    """Compute camera position and orthonormal basis [S1, unofficial]."""
    pos = [road_x(camera_z), height, camera_z]
    ahead_z = camera_z + look_ahead
    target = [road_x(ahead_z), look_height, ahead_z]
    
    forward = normalize(sub(target, pos))
    right = normalize(cross(forward, WORLD_UP))
    up = normalize(cross(right, forward))
    
    return pos, right, up, forward

def sdf_box(p, center, size):
    """SDF for an axis-aligned box [S1, S38]."""
    # Implementation of box SDF using min/max operations
    # Not fully detailed in source, but standard SDF primitive
    # Assuming size is half-extents
    d = [abs(p[0]-center[0]) - size[0],
         abs(p[1]-center[1]) - size[1],
         abs(p[2]-center[2]) - size[2]]
    outside = max(d[0], max(d[1], d[2]))
    inside = min(max(d[0], d[1]), max(max(d[0], d[1]), d[2]))
    if outside < 0:
        return outside
    else:
        return outside + max(0, inside)

def sdf_ground(p):
    """SDF for ground plane y=0 [S1, unofficial]."""
    return p[1]

def sdf_building(p, cell_cx, cell_cy, cell_cz):
    """Evaluate building SDF for a specific cell [S1, unofficial]."""
    # Hash to determine properties
    seed = hash_cell(cell_cx, cell_cy, cell_cz)
    # Placeholder for hash-to-properties mapping
    # Source implies footprint, height, materials are derived from hash
    # Exact mapping not provided in source text, but logic is:
    # 1. Hash cell
    # 2. Determine box dimensions/position
    # 3. Return SDF
    
    # Simplified assumption for implementation completeness:
    # Use hash to determine if building exists and its size
    # This part requires the specific hash-to-geometry logic which is 
    # implicit in the source's "deterministic integer hash function"
    # For the purpose of this article, we note the dependency.
    
    # Calculate cell center in world space
    # Road offset at cell center z
    cell_z = cell_cz * BLOCK_DEPTH
    road_offset = road_x(cell_z)
    
    # Building center x is road_offset + setback
    # Setback logic involves MARGIN and BLOCK_WIDTH
    # Exact placement formula not explicitly detailed beyond "setback from road"
    # Assuming standard grid placement relative to road centerline
    
    center = [road_offset + MARGIN, 0, cell_z] # Simplified
    size = [BLOCK_WIDTH/2, 10.0, BLOCK_DEPTH/2] # Simplified height
    
    return sdf_box(p, center, size)

def scene_sdf(p):
    """Evaluate the full scene SDF [S1, unofficial]."""
    # Determine cell coordinates
    # This requires inverting the world-to-cell transformation
    # which depends on the road curve.
    # Approximation: assume z maps directly to cell_z
    cell_cz = int(p[2] / BLOCK_DEPTH)
    cell_cx = int(p[0] / BLOCK_WIDTH) # Simplified
    cell_cy = int(p[1] / BLOCK_DEPTH) # Simplified
    
    # Evaluate building SDF
    # Note: The source uses "road space" and domain repetition.
    # The exact transformation from world p to cell coordinates is complex
    # due to the curved road. The source implies this is handled in the shader.
    # We assume a function `get_cell_coords(p)` exists which maps p to (cx, cy, cz)
    # based on the road centerline.
    
    # For the sake of the algorithm description, we assume:
    # cx, cy, cz = get_cell_coords(p)
    
    # Evaluate building SDF
    # Since the exact hash-to-geometry mapping is not provided,
    # we represent it as a black box function `building_sdf`
    # which returns the SDF value for the building in that cell.
    
    # Union of ground and buildings
    # Ground is always present
    d_ground = sdf_ground(p)
    
    # Building SDF (assuming no building in this cell for simplicity, 
    # or using the hash to determine existence)
    # In the actual shader, this is a loop over nearby cells or a single dominant cell
    # The source mentions "domain-repeated building boxes", implying a tiling function.
    
    # The source does not provide the exact SDF combination formula for the 
    # infinite city, only that it combines ground and repeated boxes.
    # We assume the standard min() for union.
    
    # Placeholder for the complex road-space transformation
    # d_building = building_sdf_in_road_space(p)
    
    return min(d_ground, d_building) # Simplified

def raymarch(origin, direction):
    """Sphere tracing loop [S1, unofficial]."""
    t = 0.0
    for i in range(MAX_STEPS):
        p = [origin[0] + t*direction[0],
             origin[1] + t*direction[1],
             origin[2] + t*direction[2]]
        
        d_val = scene_sdf(p)
        
        if d_val < 0.001: # Convergence threshold
            return True, p, d_val
        
        t += d_val
        
        if t > MAX_DIST:
            return False, p, d_val
            
    return False, p, d_val
```

**Edge Cases and Pitfalls:**
*   **Hash Drift:** The CPU and GPU must use identical integer hashing logic. Float-based hashing (e.g., `sin`) is avoided because it drifts between platforms, causing visual/logic mismatches [S1, unofficial].
*   **Road Curve Complexity:** The road centerline is a sum of three sine waves. The camera and raymarching must use the exact same `roadX(z)` function to stay aligned [S1, unofficial].
*   **Precision:** The use of 32-bit integer arithmetic for hashing ensures bit-identical results across JavaScript and GLSL [S1, unofficial].

## Subtopics

### Deterministic Hashing
A method for generating procedural content that is identical on CPU and GPU. It uses integer bit-mixing operations (`uhash`) instead of floating-point functions to ensure determinism. Choose this when game logic and rendering must share the same procedural data. Link: [Deterministic Hashing](deterministic-hashing.md) (hypothetical link).

### Procedural Road Generation
The use of analytic sine-wave sums to define a drivable, infinite road centerline. It avoids storing road geometry, computing it on-the-fly for both camera movement and scene generation. Link: [Procedural Road Generation](procedural-road-generation.md) (hypothetical link).

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$ per pixel, where $N$ is the number of pixels and $S$ is the number of steps (128) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory, as the scene is generated procedurally without storing meshes or textures [S1, unofficial].
*   **Performance:** Real-time on WebGL2-capable hardware. Performance depends on the complexity of the SDF evaluation per step, which includes hashing and road calculations [S1, unofficial].
*   **Strengths:** No asset loading, infinite world, deterministic consistency between logic and rendering [S1, unofficial].
*   **Limitations:** Limited to SDF-representable geometry, potential aliasing without supersampling, fixed step size may miss thin features if not tuned [S1, unofficial].

## Background
Neon Raymarcher is a WebGL2 experiment by PIGJET, released as an open-source project on GitHub [S1, unofficial]. It demonstrates the capabilities of modern fragment shaders for procedural generation, specifically targeting the "synthwave" aesthetic with neon-lit cities [S1, unofficial]. The project emphasizes the importance of deterministic hashing for shared procedural logic between client-side game code and rendering shaders [S1, unofficial].

## Sources
[S1] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
