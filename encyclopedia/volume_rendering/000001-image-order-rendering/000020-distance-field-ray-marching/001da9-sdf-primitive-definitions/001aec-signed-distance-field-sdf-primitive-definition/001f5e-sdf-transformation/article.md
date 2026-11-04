# SDF Transformation

Also called: Domain Distortion, Domain Transformations, SDF Affine Transformation, SDF Inverse Transformation, SDF Spatial Transformations, Transforming SDF Geometry, Transforming implicit surfaces

## Overview
SDF Transformation is a technique for applying geometric transformations (rotation, translation, scaling, bending, twisting, tiling) to implicit surfaces defined by Signed Distance Fields [S1, S2, S5, unofficial]. It operates in the **domain** of the distance function: instead of transforming the object's vertices or the camera's view matrix, it transforms the query point $\mathbf{x}$ before evaluating the distance function $f$ [S5, unofficial]. This builds on [Signed Distance Field (SDF) Primitive Definition](890-signed-distance-field-sdf-primitive-definition.md) by allowing primitives to be manipulated spatially [S5, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A query position $\mathbf{x}$ in world space, and a transformation $T$ (defined by parameters such as rotation quaternion, translation vector, scale factor, or bend/twist coefficients) [S3, S5, unofficial].
- Output: A signed distance value $d$ representing the distance from $\mathbf{x}$ to the transformed surface [S6, unofficial].

**Data Structures:**
- Transformation parameters are stored as scalars (scale $s$, angle $\theta$), vectors (translation $\mathbf{d}$, bend axis), or quaternions (orientation $\mathbf{q}$) [S3, S5, unofficial].

**Method:**
The core principle is that to transform a surface by $T$, one must evaluate the original distance function $f$ at the **inverse-transformed** point $T^{-1}(\mathbf{x})$ [S6, unofficial].
1. **Compute Inverse Transform:** Calculate $\mathbf{x}' = T^{-1}(\mathbf{x})$ [S6, unofficial].
2. **Evaluate Distance:** Query the base SDF primitive: $d' = f(\mathbf{x}')$ [S6, unofficial].
3. **Adjust Distance (if necessary):** If $T$ is an isometry (rotation, translation, reflection), the distance is preserved: $d = d'$ [S6, unofficial]. If $T$ involves non-isometric scaling or deformation, the distance must be scaled by the appropriate factor (typically the scale factor $s$ or the Lipschitz constant) to maintain metric correctness [S6, unofficial][S7].

**Formulas:**

*   **General Principle:** The transformed surface is defined by the zero-level set of the composed function:
    $$f(T^{-1}(\mathbf{x})) = 0 \quad [S6, unofficial][S7]$$

*   **Isometries (Translation, Rotation):**
    For isometries, the distance value returned by the SDF does not need adjustment [S6, unofficial]:
    $$d(\mathbf{x}, \mathbf{I} \circ f^{-1}(0)) = d(\mathbf{I}^{-1}(\mathbf{x}), f^{-1}(0)) \quad [S6, unofficial]$$
    Where $\mathbf{I}$ is the isometry transformation [S6, unofficial].

*   **Uniform Scale:**
    A uniform scale $S(x) = s\mathbf{x}$ has inverse $S^{-1}(\mathbf{x}) = \frac{1}{s}\mathbf{x}$ [S6, unofficial]. The distance must be multiplied by $s$ to preserve accuracy [S6, unofficial]:
    $$d(\mathbf{x}, \mathbf{S}(f^{-1}(0))) = s \cdot d(\mathbf{S}^{-1}(\mathbf{x}), f^{-1}(0)) \quad [S6, unofficial]$$
    Step-by-step:
    1. $\mathbf{x}' = \frac{1}{s} \mathbf{x}$ [S6, unofficial].
    2. $d' = f(\mathbf{x}')$ [S6, unofficial].
    3. Return $s \cdot d'$ [S6, unofficial].

*   **Translation:**
    Shifts the shape by vector $\mathbf{d}$. The inverse translation subtracts $\mathbf{d}$ from the position [S5, unofficial]:
    $$\mathbf{x}' = \mathbf{x} - \mathbf{d}$$
    Example: To move a sphere by $\mathbf{d}$, evaluate `sphere(p - d)` [S4, unofficial].

*   **Rotation:**
    Rotates the point by the inverse of the desired rotation [S4, S6, unofficial].
    - **Axis-Aligned:** Rotate around axis $k$ by angle $\theta$ [S5, unofficial]. Apply the inverse rotation to the query point [S5, unofficial].
    - **Quaternion:** Given orientation quaternion $\mathbf{q}$, apply the conjugate $\mathbf{q}^*$ to the vector part of the query position [S3, unofficial].
    - **Combined:** For combined Scale ($S$), Rotation ($R$), and Translation ($T$), apply the inverse of the product matrix to the query point in homogeneous coordinates:
      $$\mathbf{p}_{transformed} = ( \mathbf{p}_{query} \cdot (S \cdot R \cdot T)^{-1} ).xyz \quad [S4, unofficial]$$

*   **Twist:**
    Rotates the x-z plane around the y-axis by an angle proportional to $y$ [S2, unofficial]. Let $k$ be the twist factor. The angle is $\theta = k \cdot y$ [S2, unofficial].
    $$\mathbf{x}' = \text{rotateY}(\mathbf{x}, k \cdot y) \quad [S1, S2, unofficial]$$
    Note: The source [S1, unofficial] uses $0.5 \times p_y$ as the angle.

*   **Bend:**
    Bends the shape about the x-axis. Rotates the x-y plane around the z-axis by an angle proportional to $x$ [S2, unofficial]. Let $k$ be the bend factor.
    $$\mathbf{x}' = \text{rotateZ}(\mathbf{x}, k \cdot x) \quad [S2, unofficial]$$

*   **Infinite Tiling (Repetition):**
    Reproduces the shape infinitely. Takes period $c$. Requires position in the positive quadrant for standard modulo implementations, or uses a centered modulo [S5, unofficial].
    $$\mathbf{x}' = \text{mod}(\mathbf{x} + 0.5c, c) - 0.5c \quad [S2, unofficial]$$
    Example for x and z axes with period 1:
    ```glsl
    p.x = mod(p.x, 1.0) - 0.5;
    p.z = mod(p.z, 1.0) - 0.5;
    ```
    [S1, unofficial]

*   **Finite Tiling:**
    Variant of Infinite Tile with a bounding box defined by corner vector $\mathbf{l}$ [S5, unofficial]. The additional vector $\mathbf{l}$ is used to limit the number of repetitions of the shape [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rotate_y(p, angle):
    """Rotate vector p around Y axis by angle (radians)."""
    c = math.cos(angle)
    s = math.sin(angle)
    x, y, z = p
    return (x * c + z * s, y, -x * s + z * c)

def rotate_z(p, angle):
    """Rotate vector p around Z axis by angle (radians)."""
    c = math.cos(angle)
    s = math.sin(angle)
    x, y, z = p
    return (x * c - y * s, x * s + y * c, z)

def transform_sdf(p, transform_type, params, base_sdf):
    """
    Apply transformation to SDF query point.
    
    Args:
        p: Input query position (tuple of 3 floats)
        transform_type: String identifier ('translate', 'scale', 'twist', 'bend', 'tile')
        params: Dictionary of transformation parameters
        base_sdf: Function f(x) returning distance to untransformed primitive
        
    Returns:
        Signed distance value
    """
    x, y, z = p
    
    if transform_type == 'translate':
        # Inverse translation: subtract translation vector
        d = params['translation'] # tuple of 3 floats
        p_prime = (x - d[0], y - d[1], z - d[2])
        
    elif transform_type == 'scale':
        # Inverse scale: divide by scale factor
        s = params['scale'] # float
        p_prime = (x / s, y / s, z / s)
        # Scale distance output
        d = base_sdf(p_prime)
        return s * d
        
    elif transform_type == 'twist':
        # Twist around Y axis. Angle proportional to y.
        k = params['twist_factor'] # float
        angle = k * y
        p_prime = rotate_y(p, angle)
        
    elif transform_type == 'bend':
        # Bend around X axis (rotates X-Y plane around Z).
        # Angle proportional to x.
        k = params['bend_factor'] # float
        angle = k * x
        p_prime = rotate_z(p, angle)
        
    elif transform_type == 'tile':
        # Infinite tiling
        c = params['period'] # float
        # Centered modulo applied to all components
        p_prime = (
            (x + 0.5 * c) % c - 0.5 * c,
            (y + 0.5 * c) % c - 0.5 * c,
            (z + 0.5 * c) % c - 0.5 * c
        )
        
    else:
        # Identity or unsupported
        p_prime = p
        
    return base_sdf(p_prime)
```

**Edge Cases and Pitfalls:**
- **Precision:** When combining multiple transformations (e.g., Scale * Rotation * Translation), matrix inversion order matters [S4, unofficial]. The combined transformation matrix is inverted as a whole before application [S4, unofficial].
- **Tiling Boundaries:** Infinite tiling using modulo can cause discontinuities at the boundaries if the primitive is not symmetric or if the modulo implementation does not center the result correctly [S1, S5, unofficial].
- **Unbounded Twist:** Twisting is not Lipschitz continuous over unbounded domains [S7]. It must be constrained to a specific domain (e.g., a cylinder) to ensure the Lipschitz constant remains bounded for ray marching stability [S7].

## Subtopics

### [Implicit Surface Field Function Evaluation](1599-implicit-surface-field-function-evaluation.md)
A volume described by a field function $d = f(x,y,z)$, where $d$ represents density/distance. Points inside have $d < 0$, outside have $d > 0$, and on the contour $d = 0$. This is the fundamental evaluation step within the transformed domain.

### Lipschitz Transformation Bounds
Computes distances to implicitly defined surfaces after applying spatial transformations by adjusting the Lipschitz constant of the inverse transformation [S7].
- **Definition:** Ensures the distance function remains valid for root-finding algorithms by bounding the rate of change [S7].
- **Formulas:**
    - Isometry: No adjustment needed [S7].
    - Uniform Scale: Distance scaled by $1/s$ in the inverse domain, but returned distance scaled by $s$ [S7].
    - Taper: Lipschitz constant $Lip_{taper} = \min \frac{1}{s(z)}$ [S7].
    - Twist (Bounded): $Lip_{twist} = \sqrt{4 + \frac{\theta^2}{4}}$ [S7].
- **Pitfall:** Twisting is not Lipschitz on unbounded domains; it must be constrained to a domain like a unit cylinder along the twisting axis [S7].

### Deformation SDF
A technique to create a 'blob effect' by perturbing the position $p$ with trigonometric functions before querying the SDF [S8, unofficial].
- **Definition:** Perturbs coordinates using sine/cosine waves [S8, unofficial].
- **Step:**
    1. $p_x += 0.02 \cdot \cos(8 p_y + 3 p_z)$ [S8, unofficial].
    2. $p_y += 0.02 \cdot \sin(6 p_z + 7 p_x)$ [S8, unofficial].
    3. $p_z += 0.02 \cdot \cos(8 p_x + 9 p_x)$ (Note: Source [S8, unofficial] uses $p_x$ twice in the last term) [S8, unofficial].
- **Pitfall:** This is considered "cheating" because it breaks the strict mathematical properties of the SDF (Lipschitz continuity), potentially causing ray marching artifacts if the perturbation amplitude is too large relative to the frequency [S8, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, same as the base SDF evaluation [S4, S5, unofficial]. The cost is dominated by the arithmetic operations of the inverse transform (matrix-vector multiplication or trigonometric functions) [S4, S5, unofficial].
- **Memory Complexity:** $O(1)$. No additional memory is required for geometry or transformation matrices beyond the parameters passed to the shader/function [S5, unofficial].
- **Performance:** Highly efficient for instancing and animation. Transformations are computed per-ray-sample, allowing for dynamic animation (e.g., twisting, bending) without CPU-side updates [S1, S4, unofficial].
- **Strengths:**
    - No need to store transformed vertices [S5, unofficial].
    - Infinite tiling is free [S1, unofficial].
    - Easy to compose complex shapes from simple primitives [S5, unofficial].
- **Limitations:**
    - Non-isometric transformations (scaling, bending) require careful distance compensation to maintain correct ray marching step sizes [S6, unofficial][S7].
    - Complex deformations (twist, bend) may violate Lipschitz conditions if not bounded, leading to missed intersections or jitter in ray marching [S7].
    - Normal calculation requires differentiating the inverse transform, which can be complex for non-linear transformations like twist/bend [S4, unofficial].

## Background
SDF transformations originated in the context of ray marching and implicit surface rendering [S1, S4, S5, S8, unofficial]. Early work by Hart [S6, unofficial] formalized the use of inverse transformations for isometries and uniform scaling. Barr [S7] extended this to general deformations (taper, twist) and introduced the concept of Lipschitz constants to ensure the validity of the sphere tracing algorithm. The technique became popular in shader programming and procedural modeling (e.g., "shadertoy" culture) for its ability to create complex, animated geometry with minimal code [S1, S4, S5, S8, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S4] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S5] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S6] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S7] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S8] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
