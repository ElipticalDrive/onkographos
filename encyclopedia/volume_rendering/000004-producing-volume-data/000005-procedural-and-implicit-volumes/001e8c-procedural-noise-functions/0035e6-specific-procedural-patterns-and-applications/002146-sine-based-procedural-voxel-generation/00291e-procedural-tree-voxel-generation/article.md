# Procedural Tree Voxel Generation

Also called: L-system-like recursion, L-systems, Lindenmayer systems, Recursive Tree Branching, Tree Voxel Generation

## Overview
Procedural Tree Voxel Generation is a method for creating volumetric tree structures by recursively branching segments represented as spheres of varying thickness [S1, unofficial]. It adapts L-system principles to voxel space, where a single recursive rule generates complex, self-similar shapes by spawning sub-branches with random directional offsets and scaling factors [S2, unofficial]. Unlike shape grammars that output basic shapes directly, this method requires interpreting the recursive production rules to draw voxelized primitives [S2, unofficial]. It is distinguished from sibling methods like Simple Heightmap Voxel Generation by its hierarchical, branching topology rather than surface elevation, and from Asymmetric Wing Stroke by its focus on organic, fractal-like growth patterns [S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- `root1`: Starting 3D position (vector).
- `root2`: Ending 3D position (vector) of the current segment.
- `thicknessA`: Radius of the sphere at `root1`.
- `thicknessB`: Radius of the sphere at `root2`.
- `iteration`: Current recursion depth (integer).
- `vx, vy, vz`: Dimensions of the voxel grid.
- `voxel`: Bit-packed array storing occupancy.
- `voxel_col1, voxel_col2`: Arrays storing color data.

**Outputs:**
- Modified `voxel` array with tree geometry.
- Modified `voxel_col` arrays with color data.

### Data Structures
The voxel data is stored in a bit-packed format to save memory.
- `voxel`: `char*` array of size $(vx/8) \times vy \times vz$ bytes.
- Each byte stores 8 voxels along the X-axis.
- Bit $k$ in a byte corresponds to voxel coordinate $x$ where $k = x \pmod 8$.
- Offset calculation for voxel $(x, y, z)$:
  $$ ofs = \lfloor x/8 \rfloor + y \cdot \lfloor vx/8 \rfloor + z \cdot (\lfloor vx/8 \rfloor \cdot vy) $$
- `voxel_col1`, `voxel_col2`: `uchar*` arrays of size $vx \times vy \times vz$, storing color indices or values for each voxel.

### Algorithm Steps

1.  **Base Case Check**: If `iteration > 5` or `thicknessA < 2`, terminate recursion for this branch [S1, unofficial].
2.  **Draw Segment**:
    - Calculate the midpoint between `root1` and `root2`:
      $$ mid = \frac{root1 + root2}{2} $$
    - Calculate the midpoint thickness:
      $$ thicknessM = \frac{thicknessA + thicknessB}{2} $$
    - Draw a sphere at `mid` with radius `thicknessM` and color index 2 [S1, unofficial].
3.  **Add Detail**:
    - Calculate an offset vector using sine/cosine functions of an index `iM` (implementation-specific detail index) [S1, unofficial].
    - Draw a smaller sphere at `mid + offset` with radius `thicknessM / 3` and color index 3 [S1, unofficial].
4.  **Recursive Branching**:
    - Loop `branches` from 0 to 3 [S1, unofficial].
    - Generate a random direction vector `rnd` and normalize it [S1, unofficial].
    - Adjust the Y component to bias upward based on iteration:
      $$ rnd.y += 0.6 \cdot iteration - 0.5 $$
    - Calculate the delta vector of the current segment:
      $$ delta = root2 - root1 $$
    - Calculate the half-length of the current segment:
      $$ length = \|delta\| \cdot 0.5 $$
    - Scale and offset the branch vector:
      $$ branch = rnd \cdot \frac{length}{\|rnd\|} + delta \cdot 0.6 $$
      *(Note: Source [S1, unofficial] specifies `branch.length()` in denominator, but `rnd` is normalized, so $\|rnd\|=1$. The formula effectively scales the random direction by half the segment length and adds 60% of the current segment's direction vector)* [S1, unofficial].
    - Calculate new thickness for the child branch:
      $$ thicknessNew = thicknessA \cdot 0.4 $$
    - Recursively call `tree` with the new start point `root2` and end point `root2 + branch`, using `thicknessNew` and `iteration + 1` [S1, unofficial].
5.  **Decrement Iteration**: After the loop, decrement `iteration` if managed as a shared state, though typically `iteration` is passed by value in recursion [S1, unofficial].

### Sphere Voxelization
To draw a sphere at position `pos` with radius `r`:
1.  Iterate $x$ from $\lfloor pos.x - r \rfloor$ to $\lceil pos.x + r \rceil$.
2.  Iterate $y$ from $\lfloor pos.y - r \rfloor$ to $\lceil pos.y + r \rceil$.
3.  Iterate $z$ from $\lfloor pos.z - r \rfloor$ to $\lceil pos.z + r \rceil$.
4.  Calculate squared distance:
    $$ distSq = (x - pos.x)^2 + (y - pos.y)^2 + (z - pos.z)^2 $$
5.  If $distSq \le r^2$:
    - Set the bit in `voxel`:
      $$ voxel[ofs] \ |= \ (1 \ll (x \pmod 8)) $$
    - Set color in `voxel_col1` and `voxel_col2` at $(x, y, z)$ [S1, unofficial].

### Mipmap Generation
After the tree is generated, a mipmap level can be created:
1.  Iterate over each voxel in the current tree level.
2.  Check the 2x2x2 sub-voxels corresponding to the current voxel.
3.  If any sub-voxel is set, set the corresponding bit in the mipmap voxel.
4.  Copy color bits if present [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def draw_sphere(pos, radius, color_idx, voxel, voxel_col1, voxel_col2, vx, vy, vz):
    """Draws a sphere into the voxel grid."""
    r = int(radius)
    # Define bounds
    x_min = max(0, int(pos.x - radius))
    x_max = min(vx - 1, int(pos.x + radius))
    y_min = max(0, int(pos.y - radius))
    y_max = min(vy - 1, int(pos.y + radius))
    z_min = max(0, int(pos.z - radius))
    z_max = min(vz - 1, int(pos.z + radius))
    
    # Precompute radius squared
    r_sq = radius * radius
    
    for x in range(x_min, x_max + 1):
        # Precompute x offset and bit mask
        ofs_base = (x >> 3) + (y_min * (vx >> 3)) + (z_min * (vx >> 3) * vy)
        bit_mask = 1 << (x & 7)
        
        for y in range(y_min, y_max + 1):
            ofs_y_offset = (y - y_min) * (vx >> 3)
            for z in range(z_min, z_max + 1):
                # Calculate squared distance
                dx = x - pos.x
                dy = y - pos.y
                dz = z - pos.z
                dist_sq = dx*dx + dy*dy + dz*dz
                
                if dist_sq <= r_sq:
                    ofs = ofs_base + ofs_y_offset + (z - z_min) * (vx >> 3) * vy
                    voxel[ofs] |= bit_mask
                    voxel_col1[x + y * vx + z * vx * vy] = color_idx
                    voxel_col2[x + y * vx + z * vx * vy] = color_idx

def get_rnd():
    """Generates a random 3D vector."""
    return [random.uniform(-1, 1), random.uniform(-1, 1), random.uniform(-1, 1)]

def normalize(v):
    """Normalizes a vector."""
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0:
        return [0, 0, 0]
    return [v[0]/length, v[1]/length, v[2]/length]

def tree(root1, root2, thicknessA, thicknessB, iteration, voxel, voxel_col1, voxel_col2, vx, vy, vz):
    """
    Recursive function to generate tree branches.
    root1, root2: 3D coordinates [x, y, z]
    thicknessA, thicknessB: float radii
    iteration: int
    """
    # Base case
    if iteration > 5 or thicknessA < 2:
        # Draw leaf sphere at root1
        draw_sphere(root1, 8, 2, voxel, voxel_col1, voxel_col2, vx, vy, vz)
        return

    # Increment iteration for children
    next_iteration = iteration + 1
    
    # Calculate midpoint
    mid = [(root1[i] + root2[i]) / 2.0 for i in range(3)]
    thicknessM = (thicknessA + thicknessB) / 2.0
    
    # Draw main segment sphere
    draw_sphere(mid, thicknessM, 2, voxel, voxel_col1, voxel_col2, vx, vy, vz)
    
    # Add detail sphere
    # Note: iM is not defined in source, using a placeholder index or derived from iteration
    # Source says "using sine/cosine of iM". We assume iM is a local detail index.
    # For implementation, we might use iteration or a hash. 
    # Since source doesn't define iM, we use a deterministic offset based on iteration for reproducibility.
    iM = iteration 
    offset_x = math.sin(iM) * (thicknessM / 3.0)
    offset_y = math.cos(iM) * (thicknessM / 3.0)
    offset_z = math.sin(iM + 1) * (thicknessM / 3.0)
    detail_pos = [mid[i] + [offset_x, offset_y, offset_z][i] for i in range(3)]
    draw_sphere(detail_pos, thicknessM / 3.0, 3, voxel, voxel_col1, voxel_col2, vx, vy, vz)

    # Recursive branching
    delta = [root2[i] - root1[i] for i in range(3)]
    delta_len = math.sqrt(delta[0]**2 + delta[1]**2 + delta[2]**2)
    half_length = delta_len * 0.5
    
    for branches in range(4): # 0 to 3
        rnd = get_rnd()
        rnd = normalize(rnd)
        
        # Adjust Y component
        rnd[1] += 0.6 * iteration - 0.5
        
        # Scale and offset branch vector
        # Source: branch = rnd * length / branch.length() + delta*0.6
        # Since rnd is normalized, branch.length() is 1.
        branch_vec = [rnd[i] * half_length + delta[i] * 0.6 for i in range(3)]
        
        new_root2 = [root2[i] + branch_vec[i] for i in range(3)]
        thicknessNew = thicknessA * 0.4
        
        tree(root2, new_root2, thicknessA, thicknessNew, next_iteration, voxel, voxel_col1, voxel_col2, vx, vy, vz)

def generate_tree(vx, vy, vz):
    """
    Initializes voxel data and starts tree generation.
    """
    # Allocate memory
    voxel_size = (vx // 8) * vy * vz
    voxel = bytearray(voxel_size)
    voxel_col1 = bytearray(vx * vy * vz)
    voxel_col2 = bytearray(vx * vy * vz)
    
    # Initial trunk parameters
    # Source implies starting with a trunk. 
    # Example: root1 at (0,0,0), root2 at (0,10,0), thickness 10
    root1 = [0.0, 0.0, 0.0]
    root2 = [0.0, 10.0, 0.0]
    thicknessA = 10.0
    thicknessB = 8.0
    iteration = 0
    
    tree(root1, root2, thicknessA, thicknessB, iteration, voxel, voxel_col1, voxel_col2, vx, vy, vz)
    
    return voxel, voxel_col1, voxel_col2
```

### Parameters
- `iteration`: Static counter for recursion depth. Base case triggers if `iteration > 5` or `thickness < 2` [S1, unofficial].
- `iso`: Isolation threshold for loading voxel data from file, default 168 [S1, unofficial]. (Note: This parameter is mentioned in the context of file I/O, not the generation algorithm itself).
- `thicknessA`: Initial thickness of the trunk.
- `thicknessB`: Thickness at the end of the trunk segment.
- `branches`: Number of sub-branches spawned per node (fixed at 4 in the source loop) [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Floating point errors in distance calculations may cause gaps or overlaps in sphere voxelization. Using squared distance comparisons avoids square roots [S1, unofficial].
- **Boundary Checks**: Voxel coordinates must be clamped to the grid dimensions $[0, vx-1] \times [0, vy-1] \times [0, vz-1]$ to prevent out-of-bounds memory access [S1, unofficial].
- **Degenerate Input**: If `root1` equals `root2`, the delta vector is zero, resulting in no branch direction. The normalization of `rnd` handles this, but the branch vector becomes purely random, potentially causing instability.
- **Stack Overflow**: Deep recursion (e.g., `iteration > 5`) can lead to stack overflow if not managed. The source limits iteration to 5 [S1, unofficial].

## Subtopics

### L-System Principles
L-systems use production rules to model plant growth. Unlike shape grammars, L-systems require interpretation of the output string to produce the final result [S2, unofficial]. They are well-suited for self-similar shapes like trees [S2, unofficial].

### Self-Sensitive L-Systems
Used in city layout generation, these L-systems change under local constraints to avoid overlapping streets or parallel roads [S2, unofficial]. This concept can be adapted for tree generation to prevent branch collision.

### Mipmap Generation for Trees
A post-processing step where 2x2x2 sub-voxels are checked to generate lower-resolution levels of the tree voxel data [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
- **Time**: $O(N)$ where $N$ is the number of voxels in the bounding box of the tree. The number of spheres drawn is proportional to the number of branches, which grows exponentially with `iteration` (base 4). Each sphere voxelization is $O(r^3)$.
- **Memory**: $O(vx \cdot vy \cdot vz / 8)$ for the bit-packed voxel array, plus $O(vx \cdot vy \cdot vz)$ for color arrays [S1, unofficial].

### Measured Performance
- Not given in the sources.

### Scaling
- The tree size scales with the initial thickness and the number of iterations. Increasing `iteration` exponentially increases the number of branches and voxels [S1, unofficial].
- The scaling factor for thickness ($0.4$) and length (implicit in `half_length`) controls the fractal dimension [S1, unofficial].

### Strengths and Limitations
- **Strengths**: Simple implementation, generates organic-looking structures, low memory footprint due to bit-packing [S1, unofficial].
- **Limitations**: Limited to spherical primitives, no internal structure (hollow branches), random variation may lead to non-physical overlaps [S1, S2, unofficial].

### When to Use
- Use for background vegetation, low-poly trees, or when procedural generation is required without external assets [S1, S2, unofficial].
- Not suitable for high-fidelity, detailed tree models requiring complex geometry or internal structure [S2, unofficial].

## Background

### Origin
The method is based on L-systems, introduced by Aristid Lindenmayer in 1968 to model plant development [S2, unofficial]. The specific voxel implementation is derived from the OpenGL Voxel Splatting project by Brian Paul (2007) [S1, unofficial].

### History and Development
L-systems have been widely used in procedural architectural modeling and city layout generation, such as by Parish & Müller (2001) [S2, unofficial]. The voxel-based approach adapts these concepts to volumetric rendering, using sphere splatting to represent branches [S1, unofficial].

### Usage
- Procedural vegetation in voxel-based games and simulations [S1, unofficial].
- Architectural modeling and city layout generation using L-systems [S2, unofficial].

## Sources

[S1] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
[S2] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
