# Voxel Ray Traversal (AABB-optimized)

Also called: Fast Voxel Traversal, VoxelRay3Iterator

## Overview
Voxel Ray Traversal (AABB-optimized) is an iterator that traverses a 3D voxel volume along a ray, returning hits for voxels contained within the volume [S1, unofficial]. It solves the problem of efficiently enumerating voxels by combining a standard 3D DDA traversal with an initial Axis-Aligned Bounding Box (AABB) rejection test [S1, unofficial]. What sets it apart from its siblings, such as [Octree Traversal](1587-octree-traversal.md) or [Sparse Voxel Ray Marching with AABB Exit Planes](3252-sparse-voxel-ray-marching-with-aabb-exit-planes.md), is its reliance on a flat or chunk-based volume structure where the AABB test serves primarily as a "jump ahead" mechanism to skip empty space before the voxel-by-voxel iteration begins, rather than using hierarchical node structures for traversal [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `ray`: A ray defined by origin `p` (Vec3), direction `d` (Vec3, normalized), and `length` (f32) [S1, unofficial].
- `volume`: A `BoundingVolume3` object containing `size` (Vec3) and a `contains_point` method to check voxel occupancy [S1, unofficial].

**Outputs:**
- An iterator yielding `Ray3hit` structs, each containing:
  - `distance`: f32, the distance along the ray to the voxel entry point.
  - `voxel`: IVec3, the integer coordinates of the voxel.
  - `normal`: Option<IVec3>, the normal of the face crossed to enter the voxel (negative step direction) [S1, unofficial].

### Data Structures
The iterator state maintains the following fields:
- `volume`: Reference to the volume being traversed.
- `max_d`: f32, the maximum distance the ray can travel.
- `i`: IVec3, the current voxel index.
- `step`: IVec3, the direction of stepping for each axis (+1 or -1).
- `delta`: Vec3, the distance along the ray to cross one voxel in each axis.
- `dist`: Vec3, the distance from the current position `p` to the next voxel boundary in each axis.
- `t_max`: Vec3, the distance along the ray to the nearest voxel boundary in each axis.
- `t`: f32, the current traveled distance.
- `norm`: Option<IVec3>, the normal of the last crossed face.
- `done`: bool, flag indicating iteration completion [S1, unofficial].

### Steps

1. **Initialization and AABB Rejection:**
   - Normalize the ray direction: $d = \text{normalize}(ray.direction)$ [S1, unofficial].
   - Initialize traveled distance $t = 0.0$ [S1, unofficial].
   - Check if the floor of the ray origin $p$ is inside the volume: if $\text{volume.contains_point}(\lfloor p \rfloor)$ is false, perform an AABB test against the chunk bounds using `test_aabb_of_chunk(volume, p, d, ray.length)` [S1, unofficial].
   - If the AABB test fails (returns `None`), return a done iterator [S1, unofficial].
   - If the AABB test succeeds, unwrap the intersection point $aabb$ [S1, unofficial].
   - Back the origin $p$ off by at least 1 voxel: $p = aabb - d \times 2.0$ [S1, unofficial].
   - Update the traveled distance: $t += \text{length}(p - aabb) - 2.0$ [S1, unofficial].

2. **Calculate Maximum Distance:**
   - Calculate `max_d` as the minimum of the ray's specified length and the current distance $t$ plus the corner-to-corner length of the volume plus 2.0:
     $$ \text{max\_d} = \min(\text{ray.length}, t + \text{length}(\text{volume.size}) + 2.0) $$
     [S1, unofficial]

3. **Calculate Starting Voxel and Step Direction:**
   - Starting voxel $i = \lfloor p \rfloor$ converted to `IVec3` [S1, unofficial].
   - Step direction $\text{step} = \text{signum}(d)$ converted to `IVec3` [S1, unofficial].

4. **Calculate Delta:**
   - For each axis $k \in \{x, y, z\}$, calculate $\delta_k$:
     $$ \delta_k = \begin{cases} \infty & \text{if } |d_k| < \epsilon \\ 1.0 / |d_k| & \text{otherwise} \end{cases} $$
     where $\epsilon = \text{f32::EPSILON}$ [S1, unofficial].

5. **Calculate Distance to Boundary:**
   - For each axis $k$, calculate $\text{dist}_k$:
     $$ \text{dist}_k = \begin{cases} i_k + 1.0 - p_k & \text{if } \text{step}_k > 0 \\ p_k - i_k & \text{if } \text{step}_k \le 0 \end{cases} $$
     [S1, unofficial]

6. **Calculate t_max:**
   - For each axis $k$, calculate $t_{\text{max}, k}$:
     $$ t_{\text{max}, k} = \begin{cases} \delta_k \times \text{dist}_k & \text{if } \delta_k < \infty \\ \infty & \text{otherwise} \end{cases} $$
     [S1, unofficial]

7. **Iteration Loop:**
   - While $t \le \text{max\_d}$:
     - Check if the current voxel $i$ is within the volume using $\text{volume.contains_point}(i)$ [S1, unofficial].
     - If true, create a `Ray3hit` with `distance = t`, `voxel = i`, and `normal` mapped from `norm` [S1, unofficial].
     - Select the axis with the smallest $t_{\text{max}}$:
       - If $t_{\text{max}, x} < t_{\text{max}, y}$ and $t_{\text{max}, x} < t_{\text{max}, z}$:
         - $i_x += \text{step}_x$
         - $t = t_{\text{max}, x}$
         - $t_{\text{max}, x} += \delta_x$
         - $\text{norm} = (-\text{step}_x, 0, 0)$
       - Else if $t_{\text{max}, y} < t_{\text{max}, z}$:
         - $i_y += \text{step}_y$
         - $t = t_{\text{max}, y}$
         - $t_{\text{max}, y} += \delta_y$
         - $\text{norm} = (0, -\text{step}_y, 0)$
       - Else:
         - $i_z += \text{step}_z$
         - $t = t_{\text{max}, z}$
         - $t_{\text{max}, z} += \delta_z$
         - $\text{norm} = (0, 0, -\text{step}_z)$
     - If a hit was found in the current voxel, return it immediately [S1, unofficial].
   - If the loop finishes without finding a hit, set `done = true` and return `None` [S1, unofficial].

### AABB Intersection Calculation (Helper)
The `test_aabb_of_chunk` function calculates the intersection of a ray with an AABB defined by `min` (Vec3::ZERO) and `max` (volume.size as Vec3):
1. For each axis $i \in \{0, 1, 2\}$, calculate $t_i$:
   $$ t_i = \begin{cases} \frac{\text{min}_i - \text{from}_i}{\text{direction}_i} & \text{if } \text{direction}_i > 0.0 \\ \frac{\text{max}_i - \text{from}_i}{\text{direction}_i} & \text{otherwise} \end{cases} $$
   [S1, unofficial]
2. Find the axis $m_i$ with the smallest $t$ value using nested comparisons [S1, unofficial].
3. Validate intersection: If $t_{m_i} \ge 0.0$ and $t_{m_i} \le \text{distance}$, calculate the intersection point $pt = \text{from} + \text{direction} \times t_{m_i}$ [S1, unofficial].
4. Check if the other two dimensions of $pt$ are within $[\text{min}, \text{max}]$ [S1, unofficial]. If so, return $pt$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import Optional, Tuple

class Vec3:
    def __init__(self, x: float, y: float, z: float):
        self.x = x
        self.y = y
        self.z = z

    def normalize(self) -> 'Vec3':
        length = math.sqrt(self.x**2 + self.y**2 + self.z**2)
        if length == 0:
            return Vec3(0.0, 0.0, 0.0)
        return Vec3(self.x / length, self.y / length, self.z / length)

    def length(self) -> float:
        return math.sqrt(self.x**2 + self.y**2 + self.z**2)

    def signum(self) -> 'Vec3':
        return Vec3(
            1.0 if self.x > 0 else (-1.0 if self.x < 0 else 0.0),
            1.0 if self.y > 0 else (-1.0 if self.y < 0 else 0.0),
            1.0 if self.z > 0 else (-1.0 if self.z < 0 else 0.0)
        )

class IVec3:
    def __init__(self, x: int, y: int, z: int):
        self.x = x
        self.y = y
        self.z = z

    @staticmethod
    def floor(p: Vec3) -> 'IVec3':
        return IVec3(math.floor(p.x), math.floor(p.y), math.floor(p.z))

class Ray3hit:
    def __init__(self, distance: float, voxel: IVec3, normal: Optional[IVec3]):
        self.distance = distance
        self.voxel = voxel
        self.normal = normal

class BoundingVolume3:
    def __init__(self, size: Vec3):
        self.size = size

    def contains_point(self, voxel: IVec3) -> bool:
        # Placeholder for actual occupancy check logic
        # In a real implementation, this would check a sparse data structure
        return True 

def test_aabb_of_chunk(volume: BoundingVolume3, from_pt: Vec3, direction: Vec3, distance: float) -> Optional[Vec3]:
    min_pt = Vec3(0.0, 0.0, 0.0)
    max_pt = volume.size
    
    t = [0.0, 0.0, 0.0]
    for i in range(3):
        dir_i = direction.x if i == 0 else (direction.y if i == 1 else direction.z)
        from_i = from_pt.x if i == 0 else (from_pt.y if i == 1 else from_pt.z)
        min_i = min_pt.x if i == 0 else (min_pt.y if i == 1 else min_pt.z)
        max_i = max_pt.x if i == 0 else (max_pt.y if i == 1 else max_pt.z)
        if dir_i > 0.0:
            t[i] = (min_i - from_i) / dir_i
        else:
            t[i] = (max_i - from_i) / dir_i
            
    mi = 0
    if t[0] > t[1]:
        if t[0] > t[2]:
            mi = 0
        else:
            mi = 2
    else:
        if t[1] > t[2]:
            mi = 1
        else:
            mi = 2
            
    if t[mi] >= 0.0 and t[mi] <= distance:
        pt_x = from_pt.x + direction.x * t[mi]
        pt_y = from_pt.y + direction.y * t[mi]
        pt_z = from_pt.z + direction.z * t[mi]
        
        o1 = (mi + 1) % 3
        o2 = (mi + 2) % 3
        
        # Check other dimensions
        if o1 == 1:
            if not (min_pt.y <= pt_y <= max_pt.y): return None
        elif o1 == 2:
            if not (min_pt.z <= pt_z <= max_pt.z): return None
            
        if o2 == 1:
            if not (min_pt.y <= pt_y <= max_pt.y): return None
        elif o2 == 2:
            if not (min_pt.z <= pt_z <= max_pt.z): return None
            
        return Vec3(pt_x, pt_y, pt_z)
    return None

class VoxelRay3Iterator:
    def __init__(self, volume: BoundingVolume3, p: Vec3, d: Vec3, ray_length: float):
        self.volume = volume
        self.max_d = 0.0
        self.i = IVec3(0, 0, 0)
        self.step = IVec3(0, 0, 0)
        self.delta = Vec3(0.0, 0.0, 0.0)
        self.dist = Vec3(0.0, 0.0, 0.0)
        self.t_max = Vec3(0.0, 0.0, 0.0)
        self.t = 0.0
        self.norm = None
        self.done = False
        
        d_normalized = d.normalize()
        t = 0.0
        
        if not volume.contains_point(IVec3.floor(p)):
            aabb = test_aabb_of_chunk(volume, p, d_normalized, ray_length)
            if aabb is None:
                self.done = True
                return
            
            aabb_pt = aabb
            p = Vec3(aabb_pt.x - d_normalized.x * 2.0, aabb_pt.y - d_normalized.y * 2.0, aabb_pt.z - d_normalized.z * 2.0)
            t += (p - aabb_pt).length() - 2.0
            
        max_d_val = min(ray_length, t + Vec3(volume.size.x, volume.size.y, volume.size.z).length() + 2.0)
        self.max_d = max_d_val
        
        self.i = IVec3.floor(p)
        self.step = IVec3(
            1 if d_normalized.x > 0 else (-1 if d_normalized.x < 0 else 0),
            1 if d_normalized.y > 0 else (-1 if d_normalized.y < 0 else 0),
            1 if d_normalized.z > 0 else (-1 if d_normalized.z < 0 else 0)
        )
        
        eps = 1e-7 # f32::EPSILON approximation
        self.delta = Vec3(
            float('inf') if abs(d_normalized.x) < eps else 1.0 / abs(d_normalized.x),
            float('inf') if abs(d_normalized.y) < eps else 1.0 / abs(d_normalized.y),
            float('inf') if abs(d_normalized.z) < eps else 1.0 / abs(d_normalized.z)
        )
        
        self.dist = Vec3(
            self.i.x + 1.0 - p.x if self.step.x > 0 else p.x - self.i.x,
            self.i.y + 1.0 - p.y if self.step.y > 0 else p.y - self.i.y,
            self.i.z + 1.0 - p.z if self.step.z > 0 else p.z - self.i.z
        )
        
        self.t_max = Vec3(
            self.delta.x * self.dist.x if self.delta.x < float('inf') else float('inf'),
            self.delta.y * self.dist.y if self.delta.y < float('inf') else float('inf'),
            self.delta.z * self.dist.z if self.delta.z < float('inf') else float('inf')
        )
        
        self.t = t

    def __iter__(self):
        return self

    def __next__(self) -> Optional[Ray3hit]:
        if self.done:
            return None
            
        while self.t <= self.max_d:
            hit = None
            if self.volume.contains_point(self.i):
                hit = Ray3hit(self.t, self.i, self.norm)
                
            if self.t_max.x < self.t_max.y:
                if self.t_max.x < self.t_max.z:
                    self.i.x += self.step.x
                    self.t = self.t_max.x
                    self.t_max.x += self.delta.x
                    self.norm = IVec3(-self.step.x, 0, 0)
                else:
                    self.i.z += self.step.z
                    self.t = self.t_max.z
                    self.t_max.z += self.delta.z
                    self.norm = IVec3(0, 0, -self.step.z)
            else:
                if self.t_max.y < self.t_max.z:
                    self.i.y += self.step.y
                    self.t = self.t_max.y
                    self.t_max.y += self.delta.y
                    self.norm = IVec3(0, -self.step.y, 0)
                else:
                    self.i.z += self.step.z
                    self.t = self.t_max.z
                    self.t_max.z += self.delta.z
                    self.norm = IVec3(0, 0, -self.step.z)
                    
            if hit is not None:
                return hit
                
        self.done = True
        return None
```

### Parameters
- `volume`: `BoundingVolume3` - The volume being traversed, containing `size` and containment logic [S1, unofficial].

### Edge Cases and Pitfalls
- **Zero Direction Component:** If a direction component is near zero ($|d_k| < \epsilon$), $\delta_k$ is set to $\infty$ to prevent division by zero and correctly handle rays parallel to an axis [S1, unofficial].
- **Boundary Precision:** The initial AABB test backs off the origin by 2.0 units to ensure the ray starts inside the first voxel, preventing missed hits due to floating-point inaccuracies at boundaries [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method implements a voxel traversal algorithm using an AABB test, calculates steps, deltas, and t_max values, and iterates through voxels [S1, unofficial].

## Sources
[S1] AThilenius/fast-voxel-traversal-rs, https://github.com/AThilenius/fast-voxel-traversal-rs, fetched 2026-10-24 (unofficial)
