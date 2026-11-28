# SDF Transformations

Also called: Arbitrary Direction Mirroring, Reflection Transformation, SDF Composition, SDF Mirroring, SDF Modifiers

## Overview
SDF Transformations modify the spatial domain of a Signed Distance Field (SDF) primitive by applying geometric operations to the query point $\mathbf{p}$ before evaluation, enabling complex procedural geometry such as repetition, deformation, and dynamic animation without mesh processing [S1, unofficial][S2, unofficial]. Unlike static primitives, this method allows for the composition of infinite or dynamically varying scenes with extremely low memory footprint, as the scene is defined by functions rather than data structures [S1, unofficial]. It sets itself apart from [SDF Primitive Definitions](1578-sdf-primitive-definitions.md) by focusing on the manipulation of the input coordinate space rather than the analytical distance formula itself [G].

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $\mathbf{p}$ (typically in world space), primitive parameters (e.g., spacing, twist angle), and optionally time $t$ for dynamic operators [G].
*   **Output:** Transformed point $\mathbf{p}'$ used to evaluate the base SDF, or the final distance value $d = \text{SDF}_{\text{base}}(\mathbf{p}')$ [G].

**Data Structures:**
No persistent data structures are required [G]. The transformation is computed on-the-fly per ray march step [G]. For dynamic operators, a Holistic Scene Dynamics (HSD) function may store parameters per object instance [S2, unofficial].

**Steps:**
1.  **Identify Transformation Type:** Determine if the operation is static (repetition, deformation) or dynamic (time-varying affine transformation) [S1, unofficial][S2, unofficial].
2.  **Apply Coordinate Mapping:** Transform the query point $\mathbf{p}$ into the local or transformed space $\mathbf{p}'$ using the specific transformation function $T(\mathbf{p})$ [S1, unofficial][S2, unofficial].
    *   For **Repetition (Tiling)**: Apply modular arithmetic to create a periodic grid [S1, unofficial].
    *   For **Deformation**: Apply a non-linear function to the coordinates [S1, unofficial].
    *   For **Dynamic Operators**: Calculate a transformation matrix or vector based on the HSD function, time $t$, and object ID, then apply it to the ray position [S2, unofficial].
3.  **Evaluate Base SDF:** Compute the distance $d$ using the base primitive's SDF function with the transformed point: $d = \text{SDF}_{\text{base}}(\mathbf{p}')$ [S1, unofficial][S2, unofficial].
4.  **Return Distance:** Return $d$ to the ray marching loop [G].

**Formulas:**
*   **Repetition (Tiling):**
    $$ \mathbf{p}' = \text{mod}(\mathbf{p} + \mathbf{s}/2.0, \mathbf{s}) - \mathbf{s}/2.0 $$
    Where $\mathbf{s}$ is the spacing vector (e.g., `vec3(1.0)`) [S1, unofficial].
*   **Deformation (Twist):**
    $$ \mathbf{p}' = \text{twist}(\mathbf{p}) $$
    Where `twist` is a custom coordinate mapping function [S2, unofficial].
*   **Dynamic Transformation:**
    The position on the ray is transformed at each iteration [S3]. If $M(t, \mathbf{p})$ is the dynamic transformation matrix derived from the HSD function:
    $$ \mathbf{p}' = M(t, \mathbf{p}) \cdot \mathbf{p} $$
    The HSD function is defined as $f_{\text{HSD}}: D \times t \rightarrow V$, where $D$ is object input data, $t$ is simulation time, and $V$ is the dynamics variation (e.g., transformation matrix) [S2, unofficial].

**Parameters:**
*   **Spacing ($\mathbf{s}$):** Vector defining the tile size. Typical value: `vec3(1.0)` for unit cubes [S1, unofficial].
*   **Twist Angle:** Scalar defining rotation per unit height [G].
*   **Time ($t$):** Float for dynamic animations. Updates per frame [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Lipschitz Condition:** Transformations must preserve the Lipschitz constant of 1 to ensure the distance value remains a safe lower bound for sphere tracing [G]. Non-linear deformations like twisting can violate this if not carefully implemented, potentially causing tunneling [G].
*   **Boundary Precision:** In repetition, the `mod` operation can introduce precision errors at tile boundaries [S1, unofficial]. Mirroring perpendicular to the shape's edge works best; otherwise, concave edges may break the interior distance field [S1, unofficial].
*   **Dynamic Grid Calculation:** In Dynamic Operators, the grid cell vector must be calculated independently from the Instancing operator to ensure the transformation is applied correctly before instancing logic [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, v): return Vec3(self.x + v.x, self.y + v.y, self.z + v.z)
    def __sub__(self, v): return Vec3(self.x - v.x, self.y - v.y, self.z - v.z)
    def __mul__(self, s): return Vec3(self.x * s, self.y * s, self.z * s)
    def __truediv__(self, s): return Vec3(self.x / s, self.y / s, self.z / s)
    def dot(self, v): return self.x * v.x + self.y * v.y + self.z * v.z
    def mod(self, s): return Vec3(self.x % s.x, self.y % s.y, self.z % s.z)

class Mat4:
    def __init__(self, vals): self.m = vals # 16 floats representing row-major 4x4 matrix
    def transform(self, p):
        x, y, z = p.x, p.y, p.z
        return Vec3(
            self.m[0]*x + self.m[4]*y + self.m[8]*z + self.m[12],
            self.m[1]*x + self.m[5]*y + self.m[9]*z + self.m[13],
            self.m[2]*x + self.m[6]*y + self.m[10]*z + self.m[14]
        )

def sdf_transformations(p, primitive_func, transform_type, params):
    """
    Applies SDF transformations to a query point.
    
    Args:
        p: Vec3, query point in world space.
        primitive_func: function, base SDF function (e.g., sphere, box).
        transform_type: str, 'repetition', 'deformation', or 'dynamic'.
        params: dict, transformation parameters.
        
    Returns:
        float, signed distance to the transformed surface.
    """
    p_prime = p
    
    if transform_type == 'repetition':
        spacing = params.get('spacing', Vec3(1.0, 1.0, 1.0))
        # Apply tiling transformation: mod(p + s/2, s) - s/2
        p_prime = (p + spacing / 2.0).mod(spacing) - spacing / 2.0
        
    elif transform_type == 'deformation':
        twist_angle = params.get('twist_angle', 0.0)
        # Apply twist deformation: rotate x,y based on z
        z = p.z
        angle = z * twist_angle
        c = math.cos(angle)
        s = math.sin(angle)
        p_prime = Vec3(p.x * c - p.y * s, p.x * s + p.y * c, z)
        
    elif transform_type == 'dynamic':
        # Dynamic Operators require additional processing
        # 1. Calculate grid cell vector independently
        grid_size = params.get('grid_size', 1.0)
        grid_cell = p / grid_size
        
        # 2. Apply HSD function to get transformation
        # HSD: f_HSD(D, t) -> V
        time = params.get('time', 0.0)
        object_id = params.get('object_id', 0)
        transformation_matrix = hsd_function(grid_cell, time, object_id)
        
        # 3. Apply transformation to ray position
        p_prime = transformation_matrix.transform(p)
        
    # Evaluate base primitive with transformed point
    distance = primitive_func(p_prime)
    
    return distance

def hsd_function(grid_cell, time, object_id):
    """
    Holistic Scene Dynamics Function.
    Returns a transformation matrix based on deterministic noise/hash [G].
    """
    # Deterministic pseudo-noise generator (sources do not specify exact implementation) [G]
    val = math.sin(grid_cell.x * 12.9898 + grid_cell.y * 78.233 + time * 0.5)
    tx, ty, tz = val, 0.0, 0.0
    # Identity matrix with translation
    m = [1,0,0,0, 0,1,0,0, 0,0,1,0, tx,ty,tz,1]
    return Mat4(m)
```

## Subtopics

### Dynamic Operators
An extension to the Global operators collection that applies unique affine transformations and material animations to 3D objects using the Holistic Scene Dynamics (HSD) function [S3]. Unlike original Global operators which used the Instancing operator's output (grid cell vector) as input, Dynamic Operators calculate the grid cell vector independently and transform the ray before applying the Instancing operator [S2, unofficial]. This allows for dynamic variations per 3D object, such as wind effects on grass or flying objects, by applying an additional transformation to the position on the ray from the camera to the SDF component at each iteration of the ray marching algorithm [S3].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray march step, as transformations are simple arithmetic or matrix operations [G].
*   **Memory Complexity:** $O(1)$, as no scene data structures are stored; only shader code and parameters [G].
*   **Performance:** Extremely efficient for complex procedural landscapes with hundreds of blended primitives, consuming effectively zero scene memory beyond the framebuffer [S1, unofficial].
*   **Strengths:** Infinite detail, low memory usage, natural integration with volumetrics, and ability to create complex geometry from simple functions [S1, unofficial].
*   **Limitations:** Transformations must preserve the Lipschitz condition to avoid tunneling; complex deformations may require smaller step sizes or adaptive marching [G]. Dynamic operators add overhead for HSD function evaluation and grid cell calculation [S2, unofficial].

## Background
SDF Transformations are a core technique in procedural ray marching, using `mod` for repetition and coordinate manipulation for deformation, standard in GLSL shader programming [S1, unofficial][S2, unofficial]. Dynamic Operators were introduced in the context of the Holistic UOI rendering pipeline to address the need for dynamic, unlimited object instancing with unique variations [S3].

## Sources
[S1] Signed Distance Fields - by Xor - GM Shaders, https://mini.gmshaders.com/p/sdf, fetched 2026-10-27 (unofficial)
[S2] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S3] Real-time visualization of Dynamic Unlimited Objects Instancing, https://doi.org/10.24132/csrn.2018.2802.4, fetched 2026-10-26
