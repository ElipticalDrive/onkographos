# Hit Record Structure

Also called: FrontFace Determination, Hit Record & Face Normal Orientation, HitRecord

## Overview
The Hit Record Structure is a data container used in ray tracing to store the geometric and material properties of a ray-surface intersection. It solves the problem of passing intersection details (position, normal, distance, and material) from the intersection test phase to the shading phase. It is distinguished from the [Ray Structure](4026-ray-structure.md) by storing the *result* of the intersection rather than the *query* parameters, and from sibling containers like [Ray Bundle](2179-ray-bundle.md) by focusing on single-ray per-hit state rather than batched ray state.

## How it works
### Inputs and Outputs
- **Input**: A ray (origin, direction), a geometric primitive (e.g., sphere, plane, triangle), and optionally a material reference.
- **Output**: A populated `HitRecord` structure containing the hit point, oriented normal, distance along the ray, front-face flag, and material pointer.

### Data Structures
The structure typically contains the following fields [S1, S2, unofficial]:
- `P` (or `point`): The 3D coordinates of the intersection point.
- `Normal` (or `normal`): The surface normal at the hit point, oriented to point against the ray direction.
- `T` (or `t`): The scalar distance from the ray origin to the hit point.
- `FrontFace` (or `front_face`): A boolean flag indicating whether the ray hit the front or back face of the surface.
- `Mat` (or `material`): A reference or pointer to the material properties at the hit point.
- `Transmittance` (or `transmittance`): Optional field for volumetric or transmission effects, often initialized to 1.0 [S1, unofficial].
- `IsTransmission` (or `is_transmission`): Optional boolean flag indicating if the interaction is a transmission event [S1, unofficial].

### Front Face Determination
The core algorithmic step in populating the hit record is determining the orientation of the normal. Geometric primitives often compute a "geometric outward normal" ($N_{\text{out}}$) based on the surface definition. However, for shading purposes, the normal must point against the incoming ray direction ($D$).

The front-face flag is determined by the dot product of the ray direction and the outward normal [S1, S2, unofficial]:
$$ \text{FrontFace} = N_{\text{out}} \cdot D < 0 $$

If the dot product is negative, the ray is hitting the front face (the normal points against the ray). If positive, it is hitting the back face. The oriented normal ($N$) is then computed as [S2, unofficial]:
$$ N = \begin{cases} N_{\text{out}} & \text{if } N_{\text{out}} \cdot D < 0 \\ -N_{\text{out}} & \text{otherwise} \end{cases} $$

### Pseudocode
The following Python-like pseudocode implements the hit record structure and the front-face determination logic [S1, S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vector3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

    def dot(self, other):
        return self.x * other.x + self.y * other.y + self.z * other.z

    def negate(self):
        return Vector3(-self.x, -self.y, -self.z)

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

    def at(self, t):
        return Vector3(
            self.origin.x + t * self.direction.x,
            self.origin.y + t * self.direction.y,
            self.origin.z + t * self.direction.z
        )

class Material:
    """Placeholder for material properties."""
    pass

class HitRecord:
    def __init__(self):
        self.P = Vector3(0.0, 0.0, 0.0)
        self.normal = Vector3(0.0, 0.0, 0.0)
        self.t = 0.0
        self.front_face = False
        self.mat = None
        self.transmittance = 1.0  # Default initialization [S1, unofficial]
        self.is_transmission = False

    def set_face_normal(self, ray, outward_normal):
        """
        Determines front face and orients the normal.
        Based on [S1, S2, unofficial].
        """
        # Check if outward normal points against ray direction
        self.front_face = ray.direction.dot(outward_normal) < 0
        
        # Orient normal to point against ray
        if self.front_face:
            self.normal = outward_normal
        else:
            self.normal = outward_normal.negate()

def hit_sphere(sphere, ray, t_min, t_max):
    """
    Example intersection test for a sphere.
    Returns True if hit, populating rec.
    """
    # Vector from ray origin to sphere center
    oc = Vector3(
        ray.origin.x - sphere.center.x,
        ray.origin.y - sphere.center.y,
        ray.origin.z - sphere.center.z
    )
    
    # Quadratic equation coefficients
    a = ray.direction.dot(ray.direction)
    b = 2.0 * oc.dot(ray.direction)
    c = oc.dot(oc) - sphere.radius * sphere.radius
    
    discriminant = b * b - 4 * a * c
    
    if discriminant < 0:
        return False
    
    sqrt_disc = math.sqrt(discriminant)
    
    # Find the nearest valid t
    root = (-b - sqrt_disc) / (2.0 * a)
    if root < t_min or root > t_max:
        root = (-b + sqrt_disc) / (2.0 * a)
        if root < t_min or root > t_max:
            return False
            
    rec = HitRecord()
    rec.t = root
    rec.P = ray.at(root)
    
    # Compute geometric outward normal
    outward_normal = Vector3(
        (rec.P.x - sphere.center.x) / sphere.radius,
        (rec.P.y - sphere.center.y) / sphere.radius,
        (rec.P.z - sphere.center.z) / sphere.radius
    )
    
    # Set face normal and orientation
    rec.set_face_normal(ray, outward_normal)
    
    # Assign material
    rec.mat = sphere.material
    
    return True
```

### Parameters and Edge Cases
- **Precision**: Floating-point precision errors can cause the dot product to be exactly zero or slightly off, leading to incorrect front-face determination. Using a small epsilon for comparisons is common practice [G].
- **Boundaries**: The `t` value must be clamped to valid ranges (`t_min`, `t_max`) to avoid self-intersection or hitting the camera plane.
- **Degenerate Input**: Rays with zero-length direction vectors or primitives with zero radius should be handled as invalid inputs [G].

## Subtopics

### Ray Bundle
A container for multiple rays, often used in GPU ray tracing. It differs from HitRecord by managing batched state rather than single-hit details. [Link](2179-ray-bundle.md)

### RayBatch container
Similar to Ray Bundle, optimized for batch processing. [Link](4187-raybatch-container.md)

## Operating characteristics
- **Time Complexity**: The hit record structure itself has $O(1)$ creation and update time. The cost lies in the intersection test that populates it.
- **Memory Complexity**: $O(1)$ per hit record. The size is determined by the number of fields (typically 3-5 vectors/floats/bools).
- **Strengths**: Simple, efficient, and standardizes the interface between intersection and shading.
- **Limitations**: Does not handle multiple hits per ray (requires a list of HitRecords or a linked list).
- **When to use**: Standard in ray tracers for single-bounce or path tracing where one intersection per ray is processed at a time.

## Background
The Hit Record Structure is a fundamental component of ray tracing algorithms, popularized by the "Ray Tracing in One Weekend" series and similar educational resources. It abstracts the geometric intersection details into a uniform interface for shading. The front-face determination logic is a standard technique to ensure consistent normal orientation for lighting calculations, regardless of whether the ray hits the inside or outside of a closed surface.

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
