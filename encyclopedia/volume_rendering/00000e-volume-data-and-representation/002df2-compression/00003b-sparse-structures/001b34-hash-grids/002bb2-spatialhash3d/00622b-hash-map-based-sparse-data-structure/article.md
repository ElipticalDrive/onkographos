# Hash Map-based Sparse Data Structure

Also called: Fluid Surface Data Structure, Morton code based SPH neighbor search, Rendering Data Structure (Implicit Surface Cell Hash Map), Single-level SPH Data Structure with Hash Map, Sparse Data Structure, Sparse cell grid with hash map, Surface Cell Hash Map

## Overview
A hash map-based sparse data structure that maps spatial particle positions to memory locations using a compact cell list sorted by Morton codes, designed to reduce Smoothed Particle Hydrodynamics (SPH) neighbor search complexity from $O(n^2)$ to $O(n \cdot c)$ [S1]. It distinguishes itself from its sibling [Perfect Spatial Hashing](8448-perfect-spatial-hashing.md) by supporting variable cell sizes and adaptive resolutions through multi-level structures, and from [Reprojected Spatial Hash Map Indices](5180-reprojected-spatial-hash-map-indices.md) by explicitly managing surface detection for ray tracing [S1]. It solves the problem of efficient spatial lookup in highly adaptive particle simulations where dense grids are memory-prohibitive.

## How it works
### Inputs and Outputs
**Inputs:**
*   Particle positions $x_i$ and rest volumes $V_i$.
*   Simulation domain bounds $D_{min}, D_{max}$.
*   Target hash table size $n_{hash}$.
*   Mode: Simulation (neighbor search) or Rendering (surface intersection).

**Outputs:**
*   **Simulation Structure:** Hash map $H$ with `begin` and `length` arrays, and compacted cell list $C_{begin, compact}$ allowing $O(1)$ cell lookup for any coordinate.
*   **Rendering Structure:** Hash map $H_R$ storing Morton codes or collision markers for cells containing fluid surfaces.

### Data Structures
1.  **Particle List:** Array of particles sorted by Morton code $Z_i$.
2.  **Cell List ($C$):** Array of length $n_{particles} + 1$.
    *   $C[i]$ stores the index of the first particle in cell $i$, or $-1$ if empty, or $n_{particles}$ if end of list.
3.  **Compacted Cell List ($C_{begin, compact}$):** Array of length $n_{occupied} + 1$.
    *   Stores the indices into the sorted particle list where each occupied cell begins.
4.  **Hash Map ($H$):**
    *   `H_begin`: Array of size $n_{hash}$. Stores the index in $C_{begin, compact}$ where cells mapping to hash bucket $k$ start.
    *   `H_length`: Array of size $n_{hash}$. Stores the number of cells mapping to hash bucket $k$.
5.  **Rendering Hash Map ($H_R$):**
    *   Stores either a Morton code $Z_q$ (if the cell contains surface particles and no collision occurred) or a collision marker.

### Parameters
*   $N_h$: Number of neighbors for cubic spline kernel, typically 50 [S1].
*   $n_{hash}$: Smallest prime number larger than the maximum number of particles in the simulation [S1].
*   Hash Primes (Simulation only): $p_1 = 73,856,093$, $p_2 = 19,349,663$, $p_3 = 83,492,791$ [S1].
*   Morton Code Width: 30-bit (non-adaptive) or 60-bit (adaptive) [S1]. 32-bit allows 10 bits/dimension (1024 cells/dim); 64-bit allows 21 bits/dimension (2,097,152 cells/dim) [S1].

### Formulas
*   **Support Radius:** $h_i = \sqrt[3]{\frac{3 N_h V_i}{4 \pi}}$ [S1].
*   **Effective Support Extent:** $h_i = \max(h_{ii}, h_{xi}, \max(h_{ii}, h_{yi}, \max(h_{ii}, h_{zi})))$ [S1].
*   **Max Cell Size:** $C_{max} = \max\{h_0, \dots, h_{n-1}\}$ [S1].
*   **Domain Bounds:** $D_{min} = \min\{x_0, \dots, x_{n-1}\}$, $D_{max} = \max\{x_0, \dots, x_{n-1}\}$ [S1].
*   **Normalized Coordinate:** $\bar{x} = \frac{x - D_{min}}{C_{max}}$ [S1].
*   **Linear Index:** $L(\bar{x}) = \bar{x}_x + D_x \bar{x}_y + D_y \bar{x}_z$ [S1].
*   **Morton Code:** $Z(\bar{x})$ interleaves binary bits of $\bar{x}_x, \bar{x}_y, \bar{x}_z$ [S1].
*   **Simulation Hash:** $H(\bar{x}) = (p_1 \bar{x}_x + p_2 \bar{x}_y + p_3 \bar{x}_z) \pmod{n_{hash}}$ [S1].
*   **Rendering Hash:** $H(\bar{x}) = Z(\bar{x}) \pmod{n_{hash}}$ [S1].

### Steps

#### 1. Simulation Data Structure Construction
1.  **Calculate Bounds and Cell Size:** Compute $D_{min}, D_{max}$ and $C_{max}$ from particle positions and support radii [S1].
2.  **Normalize and Sort:** For each particle $i$, compute $\bar{x}_i$ and Morton code $Z_i = Z(\bar{x}_i)$. Sort all particles by $Z_i$ [S1].
3.  **Build Cell List $C$:** Create array $C$ of size $n_{particles} + 1$:
    $$
    C[i] = \begin{cases} 
    i & \text{if } i=0 \lor Z_i \neq Z_{i-1} \\
    -1 & \text{if } Z_i = Z_{i-1} \\
    n_{particles} & \text{else}
    \end{cases}
    $$
    [S1].
4.  **Compact Cell List:** Remove all $-1$ entries from $C$ to create $C_{begin, compact}$ of length $n_{occupied} + 1$ [S1].
5.  **Sort for Hashing:** Sort the occupied cells based on their hash values $H(\bar{x})$ derived from their Morton codes or linear coordinates [S1].
6.  **Build Hash Map $H$:**
    *   Initialize `H_begin` and `H_length` to 0.
    *   For each occupied cell $i$ in the sorted list:
        *   Calculate $H_i = H(\bar{x}_i)$.
        *   If $i=0$ or $H_i \neq H_{i-1}$, set `H_begin`[$H_i$] = $i$ [S1].
        *   If $i=n_{occupied}$ or $H_i \neq H_{i+1}$, set `H_length`[$H_i$] = $i - \text{H\_begin}[H_i] - 1$ [S1].

#### 2. Rendering Data Structure Construction
1.  **Identify Surface Particles:** Calculate signed surface distance $\phi_i$ for all particles [S1]. Mark particle $i$ as surface if $\phi_i < r_i$ [S1].
2.  **Build Surface Cell List $R$:** Iterate over occupied cells $C_{compact}$. For cell $i$, let $c_0$ be the first particle in the cell ($c_0 = C_{begin, compact}[i]$).
    $$
    R[i] = \begin{cases} 
    Z(x_{c0}) & \text{if } \exists c \in C_i \text{ such that } c \text{ is a surface particle} \\
    -1 & \text{else}
    \end{cases}
    $$
    [S1].
3.  **Compact Surface List:** Compact $R$ to remove $-1$ entries, yielding list $F$ of cells containing surface particles [S1].
4.  **Build Rendering Hash Map $H_R$:**
    *   Sort $F$ by hash key $H_q = Z(q) \pmod{n_{hash}}$ [S1].
    *   Initialize $H_R$ entries as empty.
    *   For each cell $q \in F$:
        *   Calculate $H_q$.
        *   If $H_R[H_q]$ is empty, store $Z_q$ (Morton code of the cell) [S1].
        *   If $H_R[H_q]$ is occupied, mark as collision [S1].
5.  **Expand Neighbors:** For every cell $i \in F$:
    *   Calculate AABB of cell $i$ based on contained particles [S1].
    *   Check surrounding cells $N_i$.
    *   If neighbor $n$ overlaps $AABB_i$ and $n \notin F$:
        *   If hash entry for $n$ is empty, mark $n$ as potentially having fluid surface [S1].
        *   If hash entry is occupied, mark as collision [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_simulation_structure(particles, n_hash):
    """
    Builds the hash map-based sparse data structure for SPH neighbor search.
    particles: list of objects with pos (x,y,z), vol, h (support radius)
    n_hash: int, smallest prime > len(particles)
    Returns: H_begin, H_length, C_begin_compact, sorted_particles
    """
    n_particles = len(particles)
    
    # 1. Calculate bounds and max cell size
    D_min = min(p.pos for p in particles) # Component-wise min
    D_max = max(p.pos for p in particles) # Component-wise max
    C_max = max(p.h for p in particles)
    
    # 2. Normalize and compute Morton codes
    # Note: Morton code implementation depends on bit width (30/60 bit)
    # Here we assume a function morton_encode(x_norm)
    for p in particles:
        x_norm = (p.pos - D_min) / C_max
        p.z_code = morton_encode(x_norm)
        
    # 3. Sort particles by Morton code
    sorted_particles = sorted(particles, key=lambda p: p.z_code)
    
    # 4. Build Cell List C
    # C[i] is the index in sorted_particles where cell i starts
    C = [0] * (n_particles + 1)
    for i in range(n_particles):
        if i == 0 or sorted_particles[i].z_code != sorted_particles[i-1].z_code:
            C[i] = i
        else:
            C[i] = -1
    C[n_particles] = n_particles
    
    # 5. Compact C to get C_begin_compact
    # Keep only indices where C[i] != -1
    C_begin_compact = [idx for idx in C if idx != -1]
    C_begin_compact.append(n_particles) # End marker
    
    # 6. Prepare for Hash Map
    # We need to hash the cells. A cell is identified by its Morton code.
    # We create a list of (hash_key, cell_index_in_compact)
    hash_entries = []
    for idx, cell_start_idx in enumerate(C_begin_compact[:-1]): # Exclude end marker
        # Get the first particle in this cell to determine its Morton code
        p = sorted_particles[cell_start_idx]
        z_code = p.z_code
        
        # Choose hash function based on mode. 
        # For simulation, use prime-based hash for fewer collisions.
        # H(x) = (p1*x_x + p2*x_y + p3*x_z) % n_hash
        # However, the source notes H(x) = Z(x) % n_hash for rendering, 
        # and prime-based for simulation. Let's use prime-based for simulation.
        # We need integer coordinates for the prime hash.
        # x_norm is float. We need to reverse normalize or use integer coords.
        # Source says: H(x_bar) = (p1 x_bar_x + ...) % n_hash
        # x_bar is normalized.
        x_bar = p.pos # Wait, source uses x_bar in formula 16. 
        # Actually, source formula 16 uses x_bar. 
        # But x_bar is normalized. Let's assume integer coordinates are used 
        # for the prime hash to avoid float issues, or normalized floats.
        # Source text: "H(x_bar) = (p1 x_bar_x + ...) % n_hash". 
        # We will use the normalized coordinates as floats.
        
        p1, p2, p3 = 73856093, 19349663, 83492791
        h_key = (p1 * x_bar.x + p2 * x_bar.y + p3 * x_bar.z) % n_hash
        
        hash_entries.append((h_key, idx))
        
    # Sort by hash key
    hash_entries.sort(key=lambda x: x[0])
    
    # 7. Build H_begin and H_length
    H_begin = [0] * n_hash
    H_length = [0] * n_hash
    
    if not hash_entries:
        return H_begin, H_length, C_begin_compact, sorted_particles
        
    for i, (h_key, cell_idx) in enumerate(hash_entries):
        if i == 0 or h_key != hash_entries[i-1][0]:
            H_begin[h_key] = cell_idx
            
    for i, (h_key, cell_idx) in enumerate(hash_entries):
        if i == len(hash_entries) - 1 or h_key != hash_entries[i+1][0]:
            H_length[h_key] = i - H_begin[h_key]
            
    return H_begin, H_length, C_begin_compact, sorted_particles

def query_simulation(H_begin, H_length, C_begin_compact, sorted_particles, target_pos, D_min, C_max):
    """
    Finds particles in the cell containing target_pos.
    """
    x_bar = (target_pos - D_min) / C_max
    # Prime hash
    p1, p2, p3 = 73856093, 19349663, 83492791
    h_key = (p1 * x_bar.x + p2 * x_bar.y + p3 * x_bar.z) % len(H_begin)
    
    begin_idx = H_begin[h_key]
    length = H_length[h_key]
    
    if length == 0:
        return []
        
    # Iterate through cells in this bucket
    # Note: C_begin_compact stores indices into sorted_particles
    # But H_begin points to index in C_begin_compact? 
    # Source Algorithm 1 says: iterate through cells starting at H_begin[H(x_bar)]
    # H_begin stores index in the sorted hash_entries list? 
    # No, H_begin stores index in C_begin_compact? 
    # Source: "H_begin[H_i] = i, if i=0 or H_i != H_{i-1}" where i is index in sorted occupied cells.
    # So H_begin points to index in C_begin_compact.
    
    # However, we need to verify Z_c == Z_j.
    # The cells in the bucket are C_begin_compact[begin_idx : begin_idx + length]
    
    # We need the Morton code of the target cell
    target_z = morton_encode(x_bar)
    
    found_particles = []
    for k in range(begin_idx, begin_idx + length):
        cell_start_idx = C_begin_compact[k]
        # Get the Morton code of this cell (from the first particle)
        if cell_start_idx < len(sorted_particles):
            cell_z = sorted_particles[cell_start_idx].z_code
            if cell_z == target_z:
                # Found the correct cell
                # Find end of this cell
                next_cell_start = C_begin_compact[k+1] if k+1 < len(C_begin_compact) else len(sorted_particles)
                for p_idx in range(cell_start_idx, next_cell_start):
                    found_particles.append(sorted_particles[p_idx])
                break
                
    return found_particles

def build_rendering_structure(particles, C_begin_compact, sorted_particles, n_hash, D_min, C_max):
    """
    Builds the rendering data structure for surface ray tracing.
    """
    # 1. Identify surface particles
    surface_particles = []
    for p in particles:
        # phi calculation is external [HS13]
        if p.phi < p.r:
            surface_particles.append(p)
            
    # 2. Build list R of surface cells
    # R[i] = Z(x_c0) if cell i has surface particle, else -1
    R = []
    for idx, cell_start_idx in enumerate(C_begin_compact[:-1]):
        has_surface = False
        next_cell_start = C_begin_compact[idx+1]
        for p_idx in range(cell_start_idx, next_cell_start):
            p = sorted_particles[p_idx]
            if p.phi < p.r:
                has_surface = True
                break
        if has_surface:
            R.append(sorted_particles[cell_start_idx].z_code)
        else:
            R.append(-1)
            
    # 3. Compact R to F
    F = [z for z in R if z != -1]
    
    # 4. Build H_R
    # Sort F by hash key H(q) = Z(q) % n_hash
    # Since Z(q) is the Morton code, and we want spatial locality, we use Morton hash
    hash_entries = []
    for z_code in F:
        h_key = z_code % n_hash
        hash_entries.append((h_key, z_code))
        
    hash_entries.sort(key=lambda x: x[0])
    
    H_R_begin = [0] * n_hash
    H_R_length = [0] * n_hash
    H_R_data = [None] * n_hash # Stores Z_q or Collision Marker
    
    if not hash_entries:
        return H_R_begin, H_R_length, H_R_data
        
    for i, (h_key, z_code) in enumerate(hash_entries):
        if i == 0 or h_key != hash_entries[i-1][0]:
            H_R_begin[h_key] = i
            
    for i, (h_key, z_code) in enumerate(hash_entries):
        if i == len(hash_entries) - 1 or h_key != hash_entries[i+1][0]:
            H_R_length[h_key] = i - H_R_begin[h_key]
            
    # 5. Embed cell info or mark collisions
    # If length is 1, store Z_q. If > 1, mark collision.
    for i, (h_key, z_code) in enumerate(hash_entries):
        # Determine if this is the last entry for this hash key
        is_last = (i == len(hash_entries) - 1) or (hash_entries[i+1][0] != h_key)
        
        if is_last:
            # Check how many entries map to this key
            begin_idx = H_R_begin[h_key]
            length = H_R_length[h_key]
            
            if length == 1:
                H_R_data[h_key] = z_code
            else:
                H_R_data[h_key] = "COLLISION"
                
    # 6. Expand neighbors
    # For each cell in F, check neighbors. 
    # This requires AABB calculation which depends on particle positions.
    # We assume a function get_cell_aabb(cell_z) exists.
    # And a function get_neighbor_cells(cell_z) exists.
    
    # To implement this fully, we need to map Morton codes back to cell indices 
    # or store AABBs in a separate structure. 
    # The source says: "For every cell i in F, check surrounding cells Ni."
    # We will assume we have a list of surface cells with their AABBs.
    
    surface_cells_with_aabb = []
    for z_code in F:
        # Find the cell index in C_begin_compact to get particles
        # This is inefficient, but for the sake of the algorithm description:
        # We need to find which cell in C_begin_compact has this Morton code.
        # In practice, we would store this mapping.
        pass 
        
    # Simplified neighbor expansion logic as per source description:
    # If neighbor n overlaps AABB_i and n not in F:
    #   If hash entry empty, mark n as potential surface.
    #   If hash entry occupied, mark as collision.
    
    # This step modifies H_R_data.
    # We iterate through F.
    for z_code in F:
        # Get AABB for this cell
        # aabb = get_aabb_for_morton(z_code) 
        # This requires knowing the cell coordinates.
        # x_bar = morton_decode(z_code)
        # cell_pos = x_bar * C_max + D_min
        # aabb = cell_pos +/- C_max/2
        
        # Get neighbors
        # neighbors = get_3x3x3_neighbors(z_code)
        
        for n_z in neighbors:
            if n_z == z_code: continue
            
            # Check if n_z is in F
            # We can use H_R to check.
            h_n = n_z % n_hash
            if H_R_length[h_n] == 0:
                # Empty. Mark as potential surface.
                # But we must ensure we don't overwrite a valid Z_q.
                # Source: "If n not in F and hash entry empty, set entry to indicate n as potentially having fluid surface."
                # How to indicate? The source says "mark the entry".
                # It implies we might store a special value or just rely on the fact that 
                # if it's not empty and not collision, it's a Z_q.
                # If we mark it, we might overwrite a Z_q? 
                # Source: "If n not in F ... If entry occupied, mark as collision."
                # This implies we only write to empty entries.
                H_R_data[h_n] = "POTENTIAL_SURFACE" # Placeholder for marker
            elif H_R_data[h_n] == "COLLISION":
                pass # Already collision
            elif H_R_data[h_n] == "POTENTIAL_SURFACE":
                pass # Already marked
            else:
                # It contains a Z_q. This means n IS in F.
                # Source: "If n in F using H_R, in which case nothing needs to be done."
                pass
                
    return H_R_begin, H_R_length, H_R_data

def query_rendering(H_R_begin, H_R_length, H_R_data, target_pos, D_min, C_max):
    """
    Queries if a cell contains fluid surface.
    """
    x_bar = (target_pos - D_min) / C_max
    z_q = morton_encode(x_bar)
    h_q = z_q % len(H_R_data)
    
    if H_R_length[h_q] == 0:
        return "NO_SURFACE"
        
    data = H_R_data[h_q]
    
    if data == "COLLISION":
        return "POTENTIAL_SURFACE"
    elif data == "POTENTIAL_SURFACE":
        return "POTENTIAL_SURFACE"
    elif isinstance(data, int): # Morton code
        if data == z_q:
            return "HAS_SURFACE"
        else:
            # Collision with a different cell's Morton code
            return "POTENTIAL_SURFACE"
            
    return "NO_SURFACE"
```

### Edge Cases and Pitfalls
*   **Hash Collisions:** An empty cell can map to the same hash entry as an occupied cell. This is handled by checking $Z_c = Z_j$ for the candidate particle index $j$ during simulation queries [S1]. In rendering, collisions are explicitly marked [S1].
*   **Precision:** Floating-point normalization errors can lead to incorrect Morton codes. Use consistent normalization [S1].
*   **Boundary Conditions:** Particles moving outside the simulation domain require domain expansion or clamping. Dense structures require known a priori bounds; sparse structures handle dynamic bounds better [S1].
*   **Adaptive Resolution:** Multi-level structures are required to handle varying cell sizes. A single cell size $C_{max}$ is used for the base structure, but additional levels are built for finer resolutions [S1].

## Subtopics

### Multi-Level Memory Structures
Extends the single-level hash map to support adaptive SPH by creating multiple data structures for different cell sizes, utilizing the self-similarity of the Morton code ordering. Choose when particle density varies significantly. [S1]

### Surface Cell Hash Map
A specialized rendering structure that implicitly marks cells containing fluid surfaces to avoid checking empty cells during ray casting. Choose when direct ray tracing of fluid surfaces is required. [S1]

## Operating characteristics
*   **Time Complexity:** Reduces particle accesses from $O(n^2)$ to $O(n \cdot c)$ where $c$ is the number of particles accessed per particle [S1]. Hash map queries are $O(1)$ on average.
*   **Memory Complexity:** Constant memory consumption regardless of simulation domain size, particle resolution, or particle distribution [S1]. Memory scales with the number of occupied cells, not the domain volume.
*   **Performance:** Significant performance improvement for spatially adaptive fluid simulations and direct ray tracing with little memory overhead [S1].
*   **Strengths:** Efficient neighbor search, low memory usage, supports adaptive resolution, enables direct ray tracing.
*   **Limitations:** Hash collisions require verification (simulation) or result in conservative surface detection (rendering). Morton code generation can be computationally expensive.
*   **When to Use:** Highly adaptive SPH simulations, real-time fluid rendering, scenarios with large empty spaces.
*   **When Not to Use:** Uniform density simulations where dense grids are more cache-friendly, or when exact surface reconstruction without potential false positives is critical without additional checks.

## Background
The hash map-based sparse data structure was introduced by Winchenbach and Kolb in 2020 [S1]. It was developed to address the limitations of dense data structures in Smoothed Particle Hydrodynamics (SPH) simulations, particularly for adaptive resolutions and real-time rendering. The method builds on earlier work by Green [Gre10] and Teschner et al. [THM*03], combining Morton codes with hash maps to achieve efficient spatial indexing. It is used in computer graphics for fluid simulation and rendering, specifically in the context of the Eurographics Computer Graphics Forum publication [S1].

## Sources
[S1] Multi‐Level Memory Structures for Simulating and Rendering Smoothed Particle Hydrodynamics, https://doi.org/10.1111/cgf.14090, fetched 2026-10-28
