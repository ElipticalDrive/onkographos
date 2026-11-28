# Ray Marching with Octant Masking

Also called: Ray march, Ray struct

## Overview
Ray Marching with Octant Masking advances a ray origin in discrete steps through a voxel grid, using a precomputed octant mask to track traversal direction and state [S1, unofficial]. It builds on standard ray-grid traversal principles by maintaining a fixed-step march loop rather than analytically computing exact boundary crossings [S5][G].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, and a step amount [S1, unofficial].
**Outputs:** Updated ray origin, traversal state, and visibility mask [S1, unofficial].

**Data Structures:**
The core data structure is the `Ray` struct, which contains `origin` (Vector), `direction` (Vector), `negdir` (Vector, unused in the reference implementation), and `init_origin` (Vector, private) [S1, unofficial].

**Algorithm Steps:**
1. Initialize `init_origin` with the starting ray position [S1, unofficial].
2. Clamp any direction component with magnitude below $\epsilon$ to $\pm\epsilon$ with the same sign to prevent division by zero or degenerate cases [S1, unofficial].
3. Compute an 8-bit `octant_mask` based on the direction signs: bit 2 (value 4) is set if `direction.x >= 0`, bit 1 (value 2) if `direction.y >= 0`, and bit 0 (value 1) if `direction.z >= 0` [S1, unofficial].
4. Advance the ray origin by $\mathbf{o} \leftarrow \mathbf{o} + \mathbf{d} \times \text{amount}$ using the `march` function [S1, unofficial].
5. Track traversal progress using `square_distance()`, which computes $(\mathbf{o} - \mathbf{init\_origin})^2$ to check against maximum bounds [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, origin, direction):
        self.origin = origin.copy()
        self.direction = direction.copy()
        self.negdir = direction.copy()  # Unused in reference implementation
        self.init_origin = origin.copy()
        self._clamp_direction()

    def _clamp_direction(self):
        epsilon = 1e-6
        for i in range(3):
            if abs(self.direction[i]) < epsilon:
                self.direction[i] = epsilon if self.direction[i] >= 0 else -epsilon

    def octant_mask(self):
        mask = 0
        if self.direction[0] >= 0: mask |= 4
        if self.direction[1] >= 0: mask |= 2
        if self.direction[2] >= 0: mask |= 1
        return mask

    def march(self, amount):
        self.origin = self.origin + self.direction * amount
        return self

    def square_distance(self):
        return sum((self.origin[i] - self.init_origin[i]) ** 2 for i in range(3))
```

**Edge Cases:**
Zero direction components are clamped to $\pm\epsilon$ to avoid numerical instability [S1, unofficial]. Boundary checks rely on external logic comparing `square_distance()` against a maximum squared distance, as the core `march` function does not inherently validate grid bounds [S1, unofficial].

## Subtopics

### [Ray Marching Voxel Textures with Neighbor-Sensitive Masking](4995-ray-marching-voxel-textures-with-neighbor-sensitive-masking.md)
This method renders dense voxel data by combining geometry shader instancing for component-level bounding boxes with ray marching through dense volume textures for the interior. It sets itself apart from pure instancing or static vertex buffers by using a neighbor-sensitive bitmask to determine voxel visibility based on adjacent components, effectively hiding internal faces without storing dense geometry for every voxel. This hybrid approach allows for dynamic updates without recomputing complex meshes while avoiding the high vertex count of instanced cubes. It is chosen when dynamic voxel updates are required without heavy mesh recomputation, and it builds on [Ray Marching with Octant Masking](2488-ray-marching-with-octant-masking.md) by replacing octant-based traversal with uniform grid ray marching through a 2D array texture.

## Operating characteristics
- Time complexity is $O(N)$ where $N$ is the number of steps, with $O(1)$ work per step [G].
- Memory complexity is $O(1)$ auxiliary space for the ray state [G].
- Measured performance with specific hardware, resolution, and scene: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths include simple implementation, low overhead per step, and easy integration with GPU shaders [G].
- Limitations include fixed step size potentially causing aliasing or missed features if too large, and lower efficiency than analytical traversal for sparse data since it samples uniformly along the path [G].
- When to use: dense voxel rendering where simplicity and uniform sampling are prioritized over precise boundary detection [G].

## Background
The method is documented in the open-source project `zazen` by Gabriel Muller, which implements a voxel ray marcher using this specific octant masking technique [S1, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
