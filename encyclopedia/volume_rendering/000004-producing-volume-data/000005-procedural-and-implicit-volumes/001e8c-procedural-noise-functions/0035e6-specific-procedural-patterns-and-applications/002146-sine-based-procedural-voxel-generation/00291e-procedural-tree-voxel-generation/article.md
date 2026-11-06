# Procedural Tree Voxel Generation

Also called: L-system-like recursion, Recursive Tree Branching, Tree Voxel Generation, branch

## Overview
Procedural Tree Voxel Generation is a recursive algorithm that constructs tree structures by drawing tapered branches as sequences of spheres and spawning sub-branches at intermediate points [S1, unofficial]. It differs from its parent, [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md), by introducing hierarchical branching logic and variable thickness, rather than relying solely on periodic functions for shape definition [G]. It is distinct from [Simple Heightmap Voxel Generation](4649-simple-heightmap-voxel-generation.md) which generates terrain, and [Asymmetric Wing Stroke](5094-asymmetric-wing-stroke.md) which generates aerodynamic surfaces [G].

## How it works
**Inputs:**
- `root1`: Start position of the branch (vec3) [S1, unofficial].
- `root2`: End position of the branch (vec3) [S1, unofficial].
- `thicknessA`: Radius at `root1` (float) [S1, unofficial].
- `thicknessB`: Radius at `root2` (float) [S1, unofficial].
- `iteration`: Current recursion depth (int) [S1, unofficial].
- `iA`, `iM`: Integer indices for detail sphere offsets (int) [S1, unofficial].

**Outputs:**
- Voxel grid updates: Sets bits in the global voxel array corresponding to the branch geometry [S1, unofficial].
- Recursive calls: Generates sub-branches [S1, unofficial].

**Data Structures:**
The voxel data is stored in a class `Tree` with the following fields [S1, unofficial]:
- `char* voxel`: Bit-packed occupancy data. Size: `(vx/8)*vy*vz` bytes [S1, unofficial].
- `uchar* voxel_col1`: First color channel data. Size: `vx*vy*vz` bytes [S1, unofficial].
- `uchar* voxel_col2`: Second color channel data. Size: `vx*vy*vz` bytes [S1, unofficial].
- `vx, vy, vz`: Dimensions of the voxel grid (int) [S1, unofficial].

Bit packing layout: The bit `(x & 7)` in `voxel[ofs]` corresponds to world coordinate `(x, y, z)`, where `ofs = (x >> 3) + y * vx8 + z * vx8vy` [S1, unofficial]. Here `vx8 = vx / 8` and `vx8vy = vx8 * vy` [G].

**Parameters:**
- `iso`: Isolation threshold for loading voxel data from file, default 168 [S1, unofficial].
- `iteration`: Static counter for recursion depth. Base case triggers if `iteration > 5` or `thickness < 2` [S1, unofficial].

**Algorithm Steps:**
1. **Base Case Check:** If `iteration > 5` or `thicknessA < 2`, draw a leaf sphere at `root1` with radius 8 and color 2. Decrement `iteration` and return [S1, unofficial].
2. **Increment Iteration:** Increment `iteration` [S1, unofficial].
3. **Calculate Segment Properties:** Calculate delta vector `delta = root2 - root1` and half length `length = delta.length() * 0.5f` [S1, unofficial]. Calculate middle thickness `thicknessM = (thicknessA + thicknessB) / 2` [S1, unofficial].
4. **Draw Main Segment:** Calculate middle point `middle = (root1 + root2) / 2 + random_vector * length * 0.1f`, where `random_vector` is a normalized random direction [S1, unofficial]. Draw a sphere at `middle` with radius `thicknessM` [S1, unofficial].
5. **Add Detail Sphere:** Calculate offset vector using sine/cosine of index `iM`: `offset = (sin(iM), cos(iM), 0) * (thicknessM / 3)` [S1, unofficial]. Draw a smaller sphere at `middle + offset` with radius `thicknessM / 3` and color 3 [S1, unofficial].
6. **Recursive Branching:** Loop `branches` from 0 to 3 [S1, unofficial]. Generate random direction `rnd` using a standard random direction generator. Normalize `rnd` [S1, unofficial]. Adjust Y component: `rnd.y += 0.6f * iteration - 0.5f` [S1, unofficial]. Scale and offset the branch vector: `branch_vec = rnd * length + delta * 0.6f` [S1, unofficial]. Calculate new thickness: `thicknessNew = thicknessA * 0.4f` [S1, unofficial]. Recursively call `tree(root2, root2 + branch_vec, thicknessNew)` [S1, unofficial]. Decrement `iteration` after the loop [S1, unofficial].

**Sphere Voxelization:**
For each sphere with center `pos` and radius `r`:
1. Iterate `x` from `pos.x - r` to `pos.x + r`, `y` from `pos.y - r` to `pos.y + r`, `z` from `pos.z - r` to `pos.z + r` [S1, unofficial].
2. Check if `(x - pos.x)^2 + (y - pos.y)^2 + (z - pos.z)^2 <= r^2` [S1, unofficial].
3. If true, set bit `(1 << (x & 7))` in `voxel[ofs]` where `ofs = (x >> 3) + y * vx8 + z * vx8vy` [S1, unofficial].
4. Set color bits in `voxel_col1` and `voxel_col2` if present [S1, unofficial].

**Mipmap Generation:**
1. For each voxel in the current tree, check its 2x2x2 sub-voxels [S1, unofficial].
2. If any sub-voxel is set, set the corresponding bit in the mipmap voxel [S1, unofficial].
3. Copy color bits if present [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def get_normalized_random_vector():
    """Returns a normalized random direction vector [G]."""
    v = [random.uniform(-1.0, 1.0) for _ in range(3)]
    length = math.sqrt(sum(c*c for c in v))
    return [c / length for c in v]

def voxel_sphere(pos, radius, color_idx, voxel, vx, vy, vz, vx8, vx8vy, voxel_col1, voxel_col2):
    r_int = int(radius)
    for x in range(int(pos[0] - radius), int(pos[0] + radius) + 1):
        for y in range(int(pos[1] - radius), int(pos[1] + radius) + 1):
            for z in range(int(pos[2] - radius), int(pos[2] + radius) + 1):
                dx = x - pos[0]
                dy = y - pos[1]
                dz = z - pos[2]
                if dx*dx + dy*dy + dz*dz <= radius*radius:
                    ofs = (x >> 3) + y * vx8 + z * vx8vy
                    bit = 1 << (x & 7)
                    voxel[ofs] |= bit
                    if voxel_col1 is not None:
                        voxel_col1[x + y * vx + z * vx * vy] = color_idx

def branch(root1, root2, thicknessA, thicknessB, iteration, iM):
    # Base case
    if iteration > 5 or thicknessA < 2:
        voxel_sphere(root1, 8.0, 2, voxel, vx, vy, vz, vx8, vx8vy, voxel_col1, voxel_col2)
        return

    # Increment iteration
    iteration += 1

    # Calculate segment properties
    delta = [root2[i] - root1[i] for i in range(3)]
    length = math.sqrt(sum(d*d for d in delta)) * 0.5
    thicknessM = (thicknessA + thicknessB) / 2.0

    # Draw main segment sphere
    random_vector = get_normalized_random_vector()
    middle = [(root1[i] + root2[i]) / 2.0 + random_vector[i] * length * 0.1 for i in range(3)]
    voxel_sphere(middle, thicknessM, 0, voxel, vx, vy, vz, vx8, vx8vy, voxel_col1, voxel_col2)

    # Add detail sphere
    offset = [math.sin(iM), math.cos(iM), 0.0]
    offset = [o * (thicknessM / 3.0) for o in offset]
    middle_offset = [middle[i] + offset[i] for i in range(3)]
    voxel_sphere(middle_offset, thicknessM / 3.0, 3, voxel, vx, vy, vz, vx8, vx8vy, voxel_col1, voxel_col2)

    # Recursive branching
    thicknessNew = thicknessA * 0.4
    for b in range(4):
        rnd = get_normalized_random_vector()
        rnd[1] += 0.6 * iteration - 0.5
        branch_vec = [rnd[i] * length + delta[i] * 0.6 for i in range(3)]
        tree(root2, [root2[i] + branch_vec[i] for i in range(3)], thicknessNew)
    
    # Decrement iteration after loop
    iteration -= 1

def tree(root1, root2, thickness):
    # Wrapper or main entry, manages iteration state
    # Assuming iteration is managed externally or passed
    branch(root1, root2, thickness, thickness, iteration, 0)

def generate_mipmaps(voxel, vx, vy, vz, vx8, vx8vy):
    # Check 2x2x2 sub-voxels
    # Implementation details for mipmap generation not fully specified in source
    # Placeholder for logic: if any sub-voxel set, set parent bit
    pass
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Proportional to the number of spheres drawn times the volume of each sphere's bounding box. Bit-packed storage reduces memory footprint by 8x compared to byte-per-voxel [S1, unofficial].
- **Strengths:** Generates complex, organic tree structures with minimal input parameters. Efficient bit-packing allows large grids [S1, unofficial].
- **Limitations:** Recursion depth limited to 5 in the source implementation. Randomness leads to non-deterministic results unless seeded. No collision detection or physical constraints [S1, unofficial].
- **When to use:** When organic, randomized tree structures are needed for voxel-based environments [S1, unofficial].
- **When not to use:** When deterministic, perfectly symmetric trees are required, or when performance constraints prohibit exponential growth [G].

## Background
The method originates from the OpenGL Voxel Splatting project by Brian Paul (2007), which demonstrated real-time voxel rendering techniques. The specific tree generation algorithm was included as a procedural example within this project, utilizing recursive branching and sphere splatting to create voxelized trees [S1, unofficial].

## Sources
[S1] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
