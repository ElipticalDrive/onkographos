# Thin Lens Depth of Field

Also called: Concentric Disk Sampling for DOF, Depth of Field, Thin Lens Camera

## Overview
The Thin Lens Depth of Field method simulates optical blur (bokeh) for a camera with a finite aperture, keeping objects at the focal distance sharp while blurring others [S2, unofficial]. It differs from pinhole models by sampling a random point on the lens disk to displace the ray origin and recalculating the ray direction to pass through a specific focus point [S2, unofficial]. This builds on the parent [Depth of Field and Lens Effects](6232-depth-of-field-and-lens-effects.md) by providing the specific geometric construction for circular apertures using concentric disk sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $\mathbf{p}$, orthonormal basis vectors $\mathbf{right}$ and $\mathbf{up}$, base ray direction $\mathbf{v}$, aperture diameter $A$, and focus distance $d_f$ [S2, unofficial].
*   **Outputs:** A ray with origin $\mathbf{o}'$ and direction $\mathbf{d}'$ [S2, unofficial].

**Data Structures:**
*   **Lens Offset:** A 2D vector $\mathbf{disk} = (d_x, d_y)$ sampled from a unit disk and scaled by the aperture [S2, unofficial].
*   **Focus Point:** A 3D vector $\mathbf{f}$ computed in world space along the base ray direction [S2, unofficial].

**Steps:**
1.  **Sample Aperture:** Generate a random 2D point $\mathbf{c}$ uniformly distributed within the unit disk using concentric disk sampling [S1, unofficial].
2.  **Scale Offset:** Compute the lens offset vector $\mathbf{disk}$ by scaling $\mathbf{c}$ by half the aperture diameter:
    $$ \mathbf{disk} = \frac{A}{2} \mathbf{c} $$
3.  **Compute New Origin:** Displace the camera position $\mathbf{p}$ by the offset projected onto the lens plane:
    $$ \mathbf{o}' = \mathbf{p} + \mathbf{disk}_x \mathbf{right} + \mathbf{disk}_y \mathbf{up} $$
4.  **Compute Focus Point:** Determine the point in space that is in sharp focus along the normalized base ray direction:
    $$ \mathbf{f} = \mathbf{p} + d_f \frac{\mathbf{v}}{\|\mathbf{v}\|} $$
5.  **Compute New Direction:** Calculate the direction from the new origin to the focus point:
    $$ \mathbf{d}' = \frac{\mathbf{f} - \mathbf{o}'}{\|\mathbf{f} - \mathbf{o}'\|} $$
6.  **Return Ray:** Construct and return the ray $(\mathbf{o}', \mathbf{d}')$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

def random_in_unit_disk():
    # Standard concentric disk sampling [G]
    while True:
        u = random.uniform(-1, 1)
        v = random.uniform(-1, 1)
        if u*u + v*v < 1.0:
            return (u, v)

def thin_lens_dof_ray(position, right, up, ray_dir, aperture, focus_distance):
    # 1. Sample concentric disk
    c = random_in_unit_disk()
    
    # 2. Scale by aperture radius
    disk = (aperture * 0.5) * c
    
    # 3. New origin on lens plane
    lens_origin = [position[i] + disk[0]*right[i] + disk[1]*up[i] for i in range(3)]
    
    # 4. Focus point along original ray direction
    ray_dir_len = math.sqrt(sum(x*x for x in ray_dir))
    focus_point = [position[i] + focus_distance * ray_dir[i] / ray_dir_len for i in range(3)]
    
    # 5. New direction from lens origin to focus point
    direction = [focus_point[i] - lens_origin[i] for i in range(3)]
    dir_len = math.sqrt(sum(x*x for x in direction))
    direction = [x / dir_len for x in direction]
    
    return Ray(lens_origin, direction)
```

**Parameters:**
*   `aperture`: Lens diameter. Larger values produce more blur [S2, unofficial].
*   `focus_distance`: Distance from the camera position to the focal plane [S2, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [Polygonal Bokeh via Thin Lens Aperture](5576-polygonal-bokeh-via-thin-lens-aperture.md)
Polygonal Bokeh via Thin Lens Aperture is a depth-of-field technique where the camera aperture is modeled as a regular $N$-sided polygon rather than a circle, causing out-of-focus highlights to render as distinct polygonal shapes. It differs from the parent [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md) method, which uses concentric disk sampling, by introducing geometric blade constraints that simulate mechanical iris limitations.

### [Lens Shift](1851-lens-shift.md)
Modifies ray direction to simulate camera tilt/shift without changing focus plane geometry. Choose for architectural correction.

### [Magnifier](6754-magnifier.md)
Simulates a magnifying glass effect. Choose for macro or close-up simulation.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] Camera Model in Path Tracing - Pinhole, Lens, and DOF Guide | Dev ..., https://code-note-vr.vercel.app/pathtracer-learning-camera-model, fetched 2026-10-29 (unofficial)
