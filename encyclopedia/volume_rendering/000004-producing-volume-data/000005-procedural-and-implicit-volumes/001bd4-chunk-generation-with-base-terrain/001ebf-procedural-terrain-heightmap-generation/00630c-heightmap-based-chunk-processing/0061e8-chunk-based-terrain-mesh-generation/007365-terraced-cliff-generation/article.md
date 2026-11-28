# Terraced Cliff Generation

Also called: Map-Edge Border Construction

## Overview
Terraced Cliff Generation constructs a 3D terrain mesh and corresponding physics colliders for linear "runs" (segments between two points) by creating stepped strata that rise outward from the map center. It differs from general procedural terrain by generating geometry specifically for boundary edges using a fixed number of terraces and coarse voxel discretization, backed by merged fixed cuboid colliders for efficient physics simulation [S1, unofficial].

## How it works
**Inputs:**
- Start point $(f_x, f_z)$ and end point $(t_x, t_z)$ in world coordinates.
- `height`: Total vertical rise of the cliff.
- `depth`: Total inward width of the cliff structure.
- `T`: Number of terraces (fixed at 6 in implementation, comment suggests $\le 8$) [S1, unofficial].
- `cv`: Voxel size constant (fixed at 0.5) [S1, unofficial].
- `W.skinThickness`: Thickness of the buried foundation layer.
- `B.wallHeight`: Height of the clearance wall.

**Outputs:**
- A voxel mesh representing the cliff geometry.
- A set of fixed rigid body colliders for physics.

**Data Structures:**
- Voxel Grid: 3D array of dimensions $[n_lx, n_ly, n_lz]$.
- Collider List: Array of cuboid definitions with position, half-extents, and material properties.

**Parameters:**
- `T = 6`
- `cv = 0.5`
- `skinThickness`: Not given in the sources.
- `wallHeight`: Not given in the sources.

**Steps:**

1.  **Compute Run Vector and Orientation:**
    Calculate the run vector components:
    $$ dx = t_x - f_x $$
    $$ dz = t_z - f_z $$
    Calculate the length $L$:
    $$ L = \sqrt{dx^2 + dz^2} $$
    Compute the unit tangent vector $(u_x, u_z)$:
    $$ u_x = \frac{dx}{L}, \quad u_z = \frac{dz}{L} $$
    Compute the initial outward normal candidate $(n_x, n_z)$:
    $$ n_x = -u_z, \quad n_z = u_x $$
    Calculate the midpoint $(m_x, m_z)$:
    $$ m_x = \frac{f_x + t_x}{2}, \quad m_z = \frac{f_z + t_z}{2} $$
    Check if the normal points towards the map center (assumed origin). If $n_x m_x + n_z m_z < 0$, flip the direction to ensure terraces face away from the map center:
    $$ u_x = -u_x, \quad u_z = -u_z $$
    $$ n_x = u_z, \quad n_z = -u_x $$
    Set the origin for the cliff to the end point $(t_x, t_z)$ if flipped, otherwise $(f_x, f_z)$? The source states: "set origin to the end point (originX = tx, originZ = tz) so terraces face away from the map center" only if the condition is met. If not met, the origin remains the start point? The source implies the origin is the point from which the cliff extends. Let $origin_x, origin_z$ be the chosen origin.

2.  **Calculate Voxel Grid Dimensions:**
    $$ n_lx = \max(1, \text{round}(L / cv)) $$
    $$ n_ly = \max(1, \text{round}(height / cv)) $$
    $$ n_lz = \max(1, \text{round}(depth / cv)) $$
    The voxel grid has dimensions $[n_lx, n_ly, n_lz]$ with voxel size $cv$ and origin at bedrock level $(0,0,0)$ in local space.

3.  **Define Terrace Height Function:**
    The height of the terrace top at a given layer index $lz$ is determined by the vertical position $zpos$ (which corresponds to the local Z-axis in the voxel grid, representing depth/inward distance):
    $$ zpos = (lz + 0.5) \cdot cv $$
    $$ t = \min(T - 1, \lfloor (zpos / depth) \cdot T \rfloor) $$
    $$ \text{terraceTop}(lz) = \frac{height \cdot (t + 1)}{T} $$

4.  **Define Voxel Presence Function:**
    For a voxel at local coordinates $(u, y, z)$ (where $u$ is along the run, $y$ is height, $z$ is depth/inward):
    - Convert local $y$ to world Y: $w_y = y \cdot cv$.
    - If $w_y < W.skinThickness$, the voxel is solid (1) (buried foundation).
    - Else, check against the terrace top. The source uses `topAt((t + 0.5) * cv)` where `t` is the inward distance index. In the voxel grid context, the inward distance is represented by the local Z coordinate. Let $z_{local}$ be the local Z index. The inward distance from the outer face is $z_{local} \cdot cv$.
    - The source formula for `solid(u, y, t)` uses `t` as inward distance. In the voxel grid, the inward distance for a voxel at index $lz$ is roughly $lz \cdot cv$. The source says: `return 1 if wy < topAt((t + 0.5) * cv)`. Here `t` in `topAt` refers to the inward distance index.
    - Let's align with the voxel grid definition:
      $$ \text{solid}(u, y, z) = \begin{cases} 1 & \text{if } y \cdot cv < W.skinThickness \\ 1 & \text{if } y \cdot cv < \text{terraceTop}(z) \\ 0 & \text{otherwise} \end{cases} $$
      Note: The source mentions `topAt` and `terraceTop` are effectively the same logic. `terraceTop` is defined per layer $lz$.

5.  **Generate Voxel Mesh:**
    - Iterate over $u \in [0, n_lx)$, $y \in [0, n_ly)$, $z \in [0, n_lz)$.
    - Determine if the voxel is solid using the presence function.
    - If solid, assign a color using `jitteredColor` based on a seed derived from edge orientation to ensure consistent but varied coloring. The source mentions `jitterBucket(x + seed, y, z)` for mergeKey.
    - Construct the mesh geometry from the solid voxels.

6.  **Create Transformation Matrix:**
    - Basis vectors:
      - X-axis (along run): $(u_x, 0, u_z)$
      - Y-axis (up): $(0, 1, 0)$
      - Z-axis (outward normal): $(n_x, 0, n_z)$
    - Convert this basis to a quaternion $q$.
    - Apply rotation $q$ and translation $(origin_x, 0, origin_z)$ to the voxel mesh.

7.  **Create Physics Colliders:**
    - Create a fixed rigid body at $(origin_x, 0, origin_z)$ with rotation $q$.
    - **Clearance Wall:** Create one tall collider at the outer face.
      - Extends from bedrock ($y=0$) to $W.skinThickness + B.wallHeight$.
      - This is the only invisible part.
    - **Terrace Colliders:** For each terrace $k$ from $0$ to $T-1$:
      - Calculate height $h_{top}$:
        $$ h_{top} = \frac{height \cdot (k + 1)}{T} $$
      - Calculate depth bounds $z_{t0}, z_{t1}$:
        $$ z_{t0} = \frac{k}{T} \cdot depth $$
        $$ z_{t1} = \frac{k + 1}{T} \cdot depth $$
      - Calculate half-extents:
        - $halfX = \frac{n_lx \cdot cv}{2}$
        - $halfY = h_{top} / 2$
        - $halfZ = (z_{t1} - z_{t0}) / 2$
      - Calculate center position in local space (before rotation):
        - $c_x = halfX$
        - $c_y = h_{top} / 2$
        - $c_z = (z_{t0} + z_{t1}) / 2$
      - Create a cuboid collider with these half-extents, translated to $(c_x, c_y, c_z)$, with friction 0.9 and restitution 0.0.
      - Attach this collider to the fixed rigid body.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_terraced_cliff(fx, fz, tx, tz, height, depth, T=6, cv=0.5, skin_thickness=0.0, wall_height=0.0):
    # 1. Compute Run Vector and Orientation
    dx = tx - fx
    dz = tz - fz
    L = math.hypot(dx, dz)
    if L == 0:
        return None, [] # Degenerate case

    ux = dx / L
    uz = dz / L
    
    # Initial outward normal candidate
    nx = -uz
    nz = ux
    
    # Midpoint
    mx = (fx + tx) / 2
    mz = (fz + tz) / 2
    
    # Check orientation relative to map center (assumed origin)
    if nx * mx + nz * mz < 0:
        ux = -ux
        uz = -uz
        nx = uz
        nz = -ux
        origin_x = tx
        origin_z = tz
    else:
        origin_x = fx
        origin_z = fz
        
    # 2. Calculate Voxel Grid Dimensions
    n_lx = max(1, round(L / cv))
    n_ly = max(1, round(height / cv))
    n_lz = max(1, round(depth / cv))
    
    # 3. Define Terrace Height Function
    def terrace_top(lz):
        zpos = (lz + 0.5) * cv
        t = min(T - 1, int((zpos / depth) * T))
        return (height * (t + 1)) / T
    
    # 4. Generate Voxel Data
    voxels = []
    for u in range(n_lx):
        for y in range(n_ly):
            for z in range(n_lz):
                wy = y * cv
                if wy < skin_thickness:
                    solid = 1
                else:
                    top = terrace_top(z)
                    solid = 1 if wy < top else 0
                
                if solid:
                    # Color logic placeholder
                    color = get_jittered_color(u, y, z)
                    voxels.append((u, y, z, color))
                    
    # 5. Create Transformation Matrix (Quaternion)
    basis_x = (ux, 0, uz)
    basis_y = (0, 1, 0)
    basis_z = (nx, 0, nz)
    quaternion = quaternion_from_basis(basis_x, basis_y, basis_z)
    
    # 6. Create Physics Colliders
    colliders = []
    
    # Clearance Wall
    wall_h = skin_thickness + wall_height
    colliders.append({
        'type': 'cuboid',
        'position': (origin_x, wall_h / 2, origin_z), # Assuming wall is at outer face, local Z=0?
        'half_extents': (n_lx * cv / 2, wall_h / 2, depth / 2), # Approximation
        'friction': 0.9,
        'restitution': 0.0
    })
    
    # Terrace Colliders
    for k in range(T):
        h_top = (height * (k + 1)) / T
        zt0 = (k / T) * depth
        zt1 = ((k + 1) / T) * depth
        
        half_x = (n_lx * cv) / 2
        half_y = h_top / 2
        half_z = (zt1 - zt0) / 2
        
        center_x = half_x
        center_y = h_top / 2
        center_z = (zt0 + zt1) / 2
        
        # Transform center to world space
        world_center = transform_point((center_x, center_y, center_z), quaternion, (origin_x, 0, origin_z))
        
        colliders.append({
            'type': 'cuboid',
            'position': world_center,
            'half_extents': (half_x, half_y, half_z),
            'friction': 0.9,
            'restitution': 0.0
        })
        
    return voxels, colliders
```

**Edge Cases and Pitfalls:**
- **Zero Length Run:** If $L=0$, the cliff cannot be generated. The code should handle this degenerate case.
- **Precision:** Floating point comparisons for voxel presence should be consistent.
- **Orientation:** The orientation flip logic is critical for ensuring terraces face away from the map center. Incorrect orientation leads to incorrect geometry.
- **Collider Alignment:** The clearance wall position and dimensions are approximated in the pseudocode based on the description. Exact alignment with the voxel mesh and terrace colliders is required for seamless physics.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels $O(n_lx \cdot n_ly \cdot n_lz)$ for mesh generation. Collider generation is $O(T)$, which is constant.
- **Memory Complexity:** Linear with respect to the number of voxels for the mesh data. Collider data is constant size.
- **Performance:** The use of coarse voxels ($cv=0.5$) and fixed number of terraces ($T=6$) keeps the geometry simple and efficient for real-time rendering and physics.
- **Strengths:** Simple, deterministic, efficient for boundary generation.
- **Limitations:** Fixed structure, not adaptable to arbitrary terrain shapes. Only suitable for linear runs.

## Background
This method is part of the "voxel-demolition" project, a game engine implementation. It is used to generate boundary cliffs for the game map, providing both visual structure and physics collision for the game world.

## Sources
[S1] JerzySukiennik/voxel-demolition, https://github.com/JerzySukiennik/voxel-demolition, fetched 2026-10-28 (unofficial)
