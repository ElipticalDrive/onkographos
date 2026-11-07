# Thin Lens Depth of Field

Also called: Concentric Disk Sampling for DOF, Depth of Field, Thin Lens Camera

## Overview
The Thin Lens Depth of Field method simulates the optical blur (bokeh) produced by a camera with a finite aperture, where objects at the focal distance are sharp and others are blurry [S2, unofficial]. It differs from the pinhole model by displacing the ray origin onto a sampled point on the lens aperture and recalculating the ray direction to pass through a specific focus point [S1, unofficial]. This builds on the parent [Depth of Field and Lens Effects](6232-depth-of-field-and-lens-effects.md) by providing the specific geometric construction for circular apertures using concentric disk sampling.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $\mathbf{p}$, basis vectors $\mathbf{right}$ and $\mathbf{up}$ (orthonormal, perpendicular to view direction), aperture diameter $A$, focus distance $d_f$, and a base ray direction $\mathbf{v}$ (typically from pinhole projection).
*   **Outputs:** A new ray with origin $\mathbf{o}'$ and direction $\mathbf{d}'$.

**Data Structures:**
*   **Lens Offset:** A 2D vector $\mathbf{u} = (u_x, u_y)$ sampled from a unit disk, scaled by the aperture.
*   **Focus Point:** A 3D vector $\mathbf{f}$ computed in world space.

**Steps:**
1.  **Sample Aperture:** Generate a random point $\mathbf{c}$ uniformly distributed within the unit disk using concentric disk sampling [S1, unofficial]. This ensures uniform probability density over the aperture area, which is critical for correct Monte Carlo integration weights [S2, unofficial].
2.  **Scale Offset:** Compute the lens offset vector $\mathbf{disk}$ by scaling $\mathbf{c}$ by half the aperture diameter:
    $$ \mathbf{disk} = \frac{A}{2} \mathbf{c} $$
3.  **Compute New Origin:** Displace the camera position $\mathbf{p}$ by the offset projected onto the lens plane:
    $$ \mathbf{o}' = \mathbf{p} + \mathbf{disk}_x \mathbf{right} + \mathbf{disk}_y \mathbf{up} $$
4.  **Compute Focus Point:** Determine the point in space that is in sharp focus. This point lies along the original ray direction $\mathbf{v}$ at distance $d_f$:
    $$ \mathbf{f} = \mathbf{p} + d_f \frac{\mathbf{v}}{\|\mathbf{v}\|} $$
5.  **Compute New Direction:** Calculate the direction from the new origin to the focus point:
    $$ \mathbf{d}' = \frac{\mathbf{f} - \mathbf{o}'}{\|\mathbf{f} - \mathbf{o}'\|} $$
6.  **Return Ray:** Construct and return the ray $(\mathbf{o}', \mathbf{d}')$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def thin_lens_dof_ray(position, right, up, ray_dir, aperture, focus_distance):
    # 1. Sample concentric disk
    # random_in_unit_disk() returns a vec2 (u, v) uniformly distributed in unit circle
    c = random_in_unit_disk()
    
    # 2. Scale by aperture radius
    disk = (aperture * 0.5) * c
    
    # 3. New origin on lens plane
    lens_origin = position + (disk.x * right) + (disk.y * up)
    
    # 4. Focus point along original ray direction
    # Normalize ray_dir to get unit vector, then scale by focus distance
    focus_point = position + (focus_distance * normalize(ray_dir))
    
    # 5. New direction from lens origin to focus point
    direction = normalize(focus_point - lens_origin)
    
    return Ray(lens_origin, direction)
```

**Parameters:**
*   `aperture`: Lens diameter. Larger values increase the blur radius for out-of-focus objects [S2, unofficial].
*   `focus_distance`: Distance from the camera position to the focal plane [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Focus Distance Zero:** If $d_f = 0$, the focus point coincides with the camera position, causing division by zero or degenerate rays. Guard against $d_f \le 0$.
*   **Aperture Zero:** If $A = 0$, the method degenerates to a pinhole camera. The offset becomes zero, and the ray direction remains unchanged.
*   **Precision:** When the focus point is very close to the lens origin, numerical instability may occur in normalization. Ensure $d_f$ is sufficiently large relative to the aperture size.
*   **Basis Vectors:** $\mathbf{right}$ and $\mathbf{up}$ must be orthonormal and perpendicular to the view direction. If not, the lens plane will be skewed, producing elliptical bokeh instead of circular [S1, unofficial].

## Subtopics

### [Polygonal Bokeh via Thin Lens Aperture](5576-polygonal-bokeh-via-thin-lens-aperture.md)
Uses a polygonal aperture shape to simulate mechanical iris blades, creating polygonal bokeh. Choose this when simulating specific camera lens characteristics.

### [Lens Shift](1851-lens-shift.md)
Modifies ray direction to simulate camera tilt/shift without changing focus plane geometry. Choose for architectural correction.

### [Magnifier](6754-magnifier.md)
Simulates a magnifying glass effect. Choose for macro or close-up simulation.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray, assuming $O(1)$ sampling.
*   **Memory Complexity:** $O(1)$ additional memory per ray.
*   **Performance:** The method adds minimal computational overhead compared to pinhole rendering, requiring only a few vector operations and one normalization per ray.
*   **Strengths:** Produces realistic depth of field with circular bokeh; simple to implement; integrates easily with path tracing.
*   **Limitations:** Assumes a thin lens model, which ignores lens aberrations and complex optical effects. Does not simulate motion blur or focus breathing.
*   **When to Use:** When realistic depth of field is required for artistic or visual fidelity purposes.
*   **When Not to Use:** When performance is critical and depth of field is not visually necessary; when simulating specific lens distortions or aberrations.

## Background
The thin lens model is a standard approximation in optics and computer graphics for simulating depth of field. It was popularized in path tracing literature as a simple yet effective way to introduce optical realism. The use of concentric disk sampling for aperture sampling ensures uniform coverage and reduces variance in Monte Carlo integration compared to simple rejection sampling.

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] Camera Model in Path Tracing - Pinhole, Lens, and DOF Guide | Dev ..., https://code-note-vr.vercel.app/pathtracer-learning-camera-model, fetched 2026-10-29 (unofficial)
