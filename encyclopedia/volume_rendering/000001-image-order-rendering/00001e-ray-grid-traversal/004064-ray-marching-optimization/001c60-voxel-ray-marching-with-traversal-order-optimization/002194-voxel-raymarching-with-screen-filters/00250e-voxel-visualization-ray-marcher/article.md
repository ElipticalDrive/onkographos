# Voxel Visualization Ray Marcher

Also called: 3D Texture Visualizer, Simple Fragment Shader Path Tracer

## Overview
A fragment shader technique that visualizes a 3D voxel grid by ray marching through a 3D texture, accumulating color along the ray until opacity saturates or the ray exits the volume [S1, unofficial]. It differs from its parent, [Voxel Raymarching with Screen Filters](2555-voxel-raymarching-with-screen-filters.md), by using a fixed-step march with pre-computed ray origins rather than screen-space filtering or adaptive traversal [S1, unofficial]. Unlike [Voxel Visualization Ray Casting](5081-voxel-visualization-ray-casting.md), which relies on pre-computed intersection textures for ray initialization, this method computes the ray direction dynamically from front and back intersection points [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `textureCoordinateFrag`: `in vec2`, fragment screen-space coordinates.
    *   `textureFront`: `sampler2D`, stores ray origin coordinates for each fragment.
    *   `textureBack`: `sampler2D`, stores ray direction endpoints for each fragment.
    *   `texture3D`: `sampler3D`, contains the voxelized volume data.
    *   `cameraPosition`: `vec3`, world-space camera position.
    *   `state`: `int`, mipmap level for sampling (default 0).
*   **Outputs**:
    *   `color`: `out vec4`, final accumulated color.

**Data Structures**
*   **Uniforms**:
    *   `STEP_LENGTH`: `float`, fixed distance advanced per step (default 0.005f).
    *   `INV_STEP_LENGTH`: `float`, computed as `1.0f / STEP_LENGTH`.
*   **Textures**:
    *   `textureFront` and `textureBack` are 2D textures mapping screen fragments to 3D space points.
    *   `texture3D` is a 3D texture containing voxel colors and alpha.

**Parameters**
*   `STEP_LENGTH`: 0.005f.
*   `INV_STEP_LENGTH`: 200.0f (derived).
*   `state`: 0.
*   `cameraPosition`: Provided by application.

**Algorithm**
1.  **Initialize Ray Origin**:
    *   Check if `cameraPosition` is inside the unit cube with a margin of 0.2f.
    *   If inside, set `origin = cameraPosition`.
    *   Otherwise, sample `origin` from `textureFront` using `textureCoordinateFrag`.
    *   *Pitfall*: The origin must avoid invalid positions. Using the front buffer ensures valid ray starting points when the camera is outside the volume [S1, unofficial].

2.  **Calculate Total Steps**:
    *   Sample `backPoint` from `textureBack` using `textureCoordinateFrag`.
    *   Compute `direction = backPoint - origin`.
    *   Compute `rayLength = length(direction)`.
    *   Compute `numberOfSteps = uint(INV_STEP_LENGTH * rayLength)`.

3.  **Initialize Accumulator**:
    *   Set `color = vec4(0.0f)`.

4.  **Ray Marching Loop**:
    *   Iterate `step` from 0 to `numberOfSteps`.
    *   **Early Exit**: If `color.a >= 0.99f`, break the loop.
    *   **Compute Position**:
        *   `currentPoint = origin + STEP_LENGTH * float(step) * direction`.
        *   *Note*: The source specifies `direction` here, implying `direction` is normalized or `STEP_LENGTH` accounts for it. However, step 2 calculates `numberOfSteps` based on `length(direction)`. If `direction` is not normalized, `currentPoint` calculation in step 4.2 must use the normalized direction or adjust `STEP_LENGTH`. The source formula `currentPoint = origin + STEP_LENGTH * step * direction` suggests `direction` might be normalized in the source implementation, or `STEP_LENGTH` is an angular step. Given `INV_STEP_LENGTH` is derived from `STEP_LENGTH` and used with `length(direction)`, `direction` is likely normalized before this step. Let $d_{norm} = \frac{direction}{length(direction)}$. Then `currentPoint = origin + STEP_LENGTH * float(step) * d_{norm}`.
        *   *Correction based on source text*: The source says `currentPoint = origin + STEP_LENGTH * step * direction`. If `direction` is the vector from front to back, its length is the ray length. If `STEP_LENGTH` is a fixed world distance, `direction` must be normalized. We assume normalization occurs or `direction` is pre-normalized.
    *   **Sample Volume**:
        *   Scale and bias `currentPoint` to [0, 1] range (assuming `currentPoint` is in [0,1] world space or needs transformation). The source says "scale and bias currentPoint to [0,1]".
        *   `sampledColor = texture3D(texture3D, currentPoint, float(state))`.
    *   **Accumulate Color**:
        *   Apply alpha compositing:
            $$c_{out} = c_{in} + (1.0 - c_{in}.a) \times s$$
            Where $c_{in}$ is the accumulated `color`, $c_{in}.a$ is its alpha, and $s$ is `sampledColor`.
        *   `color = color + (1.0 - color.a) * sampledColor`.

5.  **Gamma Correction**:
    *   Apply sRGB gamma correction to RGB channels: `color.rgb = pow(color.rgb, vec3(1.0/2.2))`.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs
textureCoordinateFrag: vec2
textureFront: sampler2D
textureBack: sampler2D
texture3D: sampler3D
cameraPosition: vec3
state: int = 0

# Constants
STEP_LENGTH: float = 0.005f
INV_STEP_LENGTH: float = 1.0f / STEP_LENGTH

# Step 1: Initialize Ray Origin
origin: vec3
if is_inside_unit_cube(cameraPosition, margin=0.2f):
    origin = cameraPosition
else:
    origin = texture2D(textureFront, textureCoordinateFrag).xyz

# Step 2: Calculate Ray Direction and Steps
backPoint: vec3 = texture2D(textureBack, textureCoordinateFrag).xyz
direction: vec3 = backPoint - origin
rayLength: float = length(direction)
# Normalize direction for consistent step size
direction = direction / rayLength
numberOfSteps: uint = uint(INV_STEP_LENGTH * rayLength)

# Step 3: Initialize Output
color: vec4 = vec4(0.0f)

# Step 4: Ray Marching Loop
for step in range(0, numberOfSteps):
    # Early exit if opaque
    if color.a >= 0.99f:
        break
    
    # Compute current point
    # Note: direction is normalized, STEP_LENGTH is world distance
    currentPoint: vec3 = origin + STEP_LENGTH * float(step) * direction
    
    # Scale and bias to [0, 1] if necessary
    # Assuming currentPoint is already in [0,1] or needs specific transform
    # The source implies a scale/bias operation. If the volume is in [0,1], 
    # and currentPoint is in world space, this step is critical.
    # For this implementation, we assume currentPoint is in [0,1] space 
    # or the texture lookup handles it. If not, apply:
    # currentPoint = scale * currentPoint + bias
    
    # Sample 3D texture
    sampledColor: vec4 = texture(texture3D, currentPoint, float(state))
    
    # Accumulate using alpha compositing
    # c_out = c_in + (1.0 - c_in.a) * s
    color = color + (1.0f - color.a) * sampledColor

# Step 5: Gamma Correction
color.rgb = pow(color.rgb, vec3(1.0/2.2))

# Output
out_color = color
```

## Subtopics

### [Voxel Visualization Ray Casting](5081-voxel-visualization-ray-casting.md)
A fragment shader technique that visualizes a 3D voxel grid by casting rays from the camera through a unit cube, using pre-computed front and back intersection textures to determine the ray origin and direction. Unlike traditional mesh rendering which requires draw calls per object, this method holds geometric complexity in 3D voxel textures and renders the entire structure in a single draw call using only a few triangles. It differs from its parent, Voxel Visualization Ray Marcher, by explicitly utilizing pre-computed intersection textures for ray initialization rather than computing intersections or marching from a generic camera position.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of steps per ray. $N$ is proportional to the ray length divided by `STEP_LENGTH`.
*   **Memory Complexity**: $O(1)$ per fragment for the shader state, plus storage for the 3D texture and front/back 2D textures.
*   **Performance**: Performance depends on `STEP_LENGTH` and ray length. Smaller `STEP_LENGTH` increases accuracy but reduces performance.
*   **Strengths**: Simple implementation, single draw call, works with any 3D texture.
*   **Limitations**: Fixed step size may lead to aliasing or missed details if `STEP_LENGTH` is too large. No early exit for empty space (unless alpha threshold is met).
*   **When to use**: Real-time visualization of voxel data where simplicity is preferred over optimization.
*   **When not to use**: Large volumes with sparse data (use [Voxel Raymarching with Screen Filters](2555-voxel-raymarching-with-screen-filters.md) or [Voxel Ray Marching with Sector-Based Indexing](6915-voxel-ray-marching-with-sector-based-indexing.md) for better culling).

## Background
Originating from unofficial implementations such as the `voxel-cone-tracing` repository, this method provides a straightforward approach to rendering voxel data using fragment shaders [S1, unofficial]. It builds on the concept of ray marching through 3D textures, a common technique in volume rendering.

## Sources
[S1] Friduric/voxel-cone-tracing, https://github.com/Friduric/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
