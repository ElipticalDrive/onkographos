# Marching Slices with Slab-based Caching

Also called: Marching Slices, Slab-based Marching Cubes

## Overview
Marching Slices is an isosurface extraction algorithm that polygonizes a volume one slice at a time to minimize working memory usage by caching only the data necessary for the current slab [S1]. Unlike standard Marching Cubes which requires storing state for the entire volume or all slices simultaneously, this method uses slab-based caching to discard unused data, making it suitable for large volumes where memory is constrained [S1]. It sets itself apart from surface-following marching cubes by reducing memory footprint and improving cache locality, and from hash-map based approaches by avoiding lookup overhead [S1].

## How it works
The algorithm operates on a discretized domain, typically a regular grid, treating the volume as a vertical array of slabs [S1]. It traverses connected cubes containing the isosurface, ensuring that all connected cubes in the current slice are processed before moving to the next, with the ability to march in arbitrary forward or reverse order [S1].

### Inputs and Outputs
- **Input**: A 3D scalar field defined on a grid of size $N_x \times N_y \times N_z$. An isovalue threshold.
- **Output**: A stream of triangles (vertices and indices) representing the isosurface, flushed to the rendering buffer [S1].

### Data Structures
The algorithm uses a sparse set of data structures over the volume [S1].

1.  **Block Structure**: The volume is treated as a vertical array of $N_z$ slabs, numbered $0$ to $N_z-1$ (where $z$ increases upwards) [S1]. Each slab contains $N_x \times N_y \times 1$ cubes. Each slab is bounded above and below by a 2D slice. Slices are numbered $0$ to $N_z$, with adjacent slabs sharing slices [S1].
2.  **Todo List**: For each of the $N_z$ slabs, a linked list containing integer tuples $(x, y)$ representing cubes in that slab that may need to be rendered [S1].
3.  **Slice Cache**: Allocated for the slice above and below a slab currently being polygonized. Each cache consists of:
    -   A 2D corner index array of size $(N_x + 1) \times (N_y + 1)$. Each element stores an index to retrieve a computed corner from the corner cache (or 0 if not computed) and two boolean "done" flags (`done_above`, `done_below`) indicating whether the cubes above and below the slice at that location have been rendered [S1].
    -   A dynamically resizing vector of corner values (corner cache) storing the actual 3D coordinates of interpolated corners [S1].

### Parameters
-   **Slices in Memory**: The typical number of slices kept in memory at the same time is three (the current slab's top and bottom slices, plus potentially one more depending on traversal) [S1].

### Algorithm Steps
1.  **Initialization**: Initialize all slab todo lists as empty. Allocate a common memory pool for slice caches [S1].
2.  **Seeding**: Find a seed cube containing the isosurface. Add its $(x, y)$ coordinate to the todo list of its corresponding slab [S1].
3.  **Processing Loop**: While any slab's todo list is not empty:
    a.  Select a slab $S$ with a non-empty todo list.
    b.  Allocate slice caches for the slice above $S$ (Slice $S+1$) and the slice below $S$ (Slice $S$) if they are not already allocated [S1].
    c.  While slab $S$'s todo list is not empty:
        i.  Pop a cube $(x, y)$ from slab $S$'s todo list.
        ii. Check if this cube has already been processed using the done flags in the adjacent slice caches. If done, continue [S1].
        iii. Perform Marching Cubes on the cube $(x, y)$ in slab $S$.
        iv. For each of the 8 corners of the cube:
            -   If the corner value is not in the corner cache, compute it and add to the corner cache. Store the index in the corner index array of the relevant slice cache [S1].
            -   Mark the corner as "done" in the appropriate direction flags in the corner index array [S1].
        v.  For each of the 6 faces of the cube, check if the adjacent cube (in the $\pm x, \pm y, \pm z$ directions) needs to be processed:
            -   If the adjacent cube is in a different slab (e.g., $\pm z$), check the `done` flag in the shared slice cache. If not done, add the adjacent cube's $(x, y)$ to the todo list of the adjacent slab [S1].
            -   If the adjacent cube is in the same slab (e.g., $\pm x, \pm y$), check the `done` flag in the current slab's adjacent slice caches. If not done, add the adjacent cube's $(x, y)$ to the current slab's todo list [S1].
        vi. Mark the current cube $(x, y)$ as done in the relevant slice caches to prevent re-processing [S1].
    d.  After slab $S$'s todo list is empty:
        i.  If the todo list of the slab above $S$ is empty, deallocate the slice cache above $S$ and return memory to the pool [S1].
        ii. If the todo list of the slab below $S$ is empty, deallocate the slice cache below $S$ and return memory to the pool [S1].
4.  **Completion**: When all slab todo lists are empty, flush the index and vertex buffers to the video card [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class SliceCache:
    def __init__(self, nx, ny):
        self.nx = nx
        self.ny = ny
        # 2D corner index array: (nx+1) x (ny+1)
        # Each element: (index, done_above, done_below)
        self.corner_index = [[(0, False, False) for _ in range(ny + 1)] for _ in range(nx + 1)]
        self.corner_cache = [] # List of 3D vectors

    def get_or_compute_corner(self, x, y, z, field, isovalue):
        # Check if corner (x,y,z) is already in cache
        # Note: x, y are slice coordinates, z is slab index
        # The corner index array is 2D, indexed by (x, y)
        # We need to map (x,y,z) to the correct slice cache
        # This is handled by the caller passing the correct slice cache
        pass

    def mark_done(self, x, y, direction):
        # direction: 'above' or 'below'
        if direction == 'above':
            self.corner_index[x][y] = (self.corner_index[x][y][0], True, self.corner_index[x][y][2])
        else:
            self.corner_index[x][y] = (self.corner_index[x][y][0], self.corner_index[x][y][1], True)

    def is_done(self, x, y, direction):
        if direction == 'above':
            return self.corner_index[x][y][1]
        else:
            return self.corner_index[x][y][2]

class Slab:
    def __init__(self, slab_index, nx, ny):
        self.slab_index = slab_index
        self.todo_list = [] # List of (x, y) tuples
        self.nx = nx
        self.ny = ny

def marching_slices(field, isovalue, nx, ny, nz):
    # Initialize slabs
    slabs = [Slab(i, nx, ny) for i in range(nz)]
    
    # Slice caches: indexed by slice number 0 to nz
    # Each slice cache is shared between slab i-1 and slab i
    slice_caches = [None] * (nz + 1)
    
    # Find initial seed
    seed_cube = find_seed_cube(field, isovalue, nx, ny, nz)
    if seed_cube is None:
        return []
    
    x, y, z = seed_cube
    slabs[z].todo_list.append((x, y))
    
    triangles = []
    
    while any(len(s.todo_list) > 0 for s in slabs):
        # Find a slab with work
        current_slab = None
        for s in slabs:
            if len(s.todo_list) > 0:
                current_slab = s
                break
        
        if current_slab is None:
            break
            
        slab_idx = current_slab.slab_index
        
        # Allocate slice caches if needed
        # Slice below slab_idx is slice_idx = slab_idx
        # Slice above slab_idx is slice_idx = slab_idx + 1
        slice_below_idx = slab_idx
        slice_above_idx = slab_idx + 1
        
        if slice_caches[slice_below_idx] is None:
            slice_caches[slice_below_idx] = SliceCache(nx, ny)
        if slice_caches[slice_above_idx] is None:
            slice_caches[slice_above_idx] = SliceCache(nx, ny)
            
        cache_below = slice_caches[slice_below_idx]
        cache_above = slice_caches[slice_above_idx]
        
        while len(current_slab.todo_list) > 0:
            x, y = current_slab.todo_list.pop()
            
            # Process cube (x, y, slab_idx)
            # Get corner values
            corners = []
            for dx in [0, 1]:
                for dy in [0, 1]:
                    for dz in [0, 1]:
                        cx, cy, cz = x + dx, y + dy, z + dz
                        # Determine which slice cache this corner belongs to
                        # Corner (cx, cy, cz) is on slice cz
                        # But the cube is in slab_idx, so corners are at z=slab_idx and z=slab_idx+1
                        # Actually, corners are at integer coordinates.
                        # Cube (x,y,z) has corners at (x,y,z) to (x+1,y+1,z+1)
                        # Slice z is the bottom of slab z. Slice z+1 is the top.
                        # So corner (cx, cy, cz) is on slice cz.
                        
                        # Check if corner is already computed
                        # We need to store the index in the cache for slice cz
                        # But we only have cache_below (slice z) and cache_above (slice z+1)
                        # If cz == z, use cache_below
                        # If cz == z+1, use cache_above
                        
                        if cz == slab_idx:
                            cache = cache_below
                        elif cz == slab_idx + 1:
                            cache = cache_above
                        else:
                            # This should not happen for a single cube
                            raise Exception("Invalid corner z")
                            
                        idx, done_above, done_below = cache.corner_index[cx][cy]
                        if idx == 0:
                            # Compute corner
                            val = field[cx][cy][cz]
                            # Linear interpolation to find intersection with isovalue
                            # This is a simplified corner computation
                            # In reality, Marching Cubes interpolates on edges
                            # Here we assume corner values are stored for the vertices
                            # And we interpolate on edges for the surface
                            # The cache stores the vertex position
                            corner_pos = compute_corner_position(cx, cy, cz, field, isovalue)
                            cache.corner_cache.append(corner_pos)
                            new_idx = len(cache.corner_cache)
                            cache.corner_index[cx][cy] = (new_idx, done_above, done_below)
                            idx = new_idx
                        corners.append(idx)
            
            # Perform Marching Cubes lookup
            # Determine case based on sign of field values at corners
            case = determine_marching_cubes_case(corners, field, isovalue)
            
            # Generate triangles
            # This part depends on the specific Marching Cubes configuration
            # For simplicity, we assume a standard lookup
            triangles.extend(generate_triangles(case, corners, cache_below, cache_above))
            
            # Mark cube as done
            # The cube itself doesn't have a "done" flag, but the corners do
            # We mark the corners as done for the cubes above and below
            # For cube (x,y,z), the corners are at z and z+1
            # We need to mark that the cubes adjacent to these corners have been processed
            
            # Mark corners as done for the cube above (slab_idx+1) and below (slab_idx)
            # This is tricky because the done flags are per-corner, per-direction
            # We mark that the cube (x,y,z) has been processed
            # This prevents re-processing the same cube
            
            # Actually, the done flags are for the cubes ABOVE and BELOW the slice
            # So for corner (cx, cy, cz), done_above means cube (cx, cy, cz) is done?
            # No, done_above means the cube above the slice at (cx, cy) is done
            # The cube above slice cz is slab cz (if we consider slab z bounded by slices z and z+1)
            # So corner (cx, cy, cz) is on slice cz.
            # The cube above slice cz is slab cz (if cz < nz)
            # The cube below slice cz is slab cz-1 (if cz > 0)
            
            # So for each corner, we mark the cube above and below as done
            for dx in [0, 1]:
                for dy in [0, 1]:
                    for dz in [0, 1]:
                        cx, cy, cz = x + dx, y + dy, z + dz
                        if cz == slab_idx:
                            cache = cache_below
                            # Cube below is slab_idx-1, cube above is slab_idx
                            # Mark cube above (slab_idx) as done for this corner
                            cache.mark_done(cx, cy, 'above')
                            # Mark cube below (slab_idx-1) as done for this corner
                            if slab_idx > 0:
                                cache.mark_done(cx, cy, 'below')
                        elif cz == slab_idx + 1:
                            cache = cache_above
                            # Cube below is slab_idx, cube above is slab_idx+1
                            # Mark cube below (slab_idx) as done for this corner
                            cache.mark_done(cx, cy, 'below')
                            # Mark cube above (slab_idx+1) as done for this corner
                            if slab_idx + 1 < nz:
                                cache.mark_done(cx, cy, 'above')
            
            # Add neighbors to todo lists
            # Neighbors in x, y direction are in the same slab
            # Neighbors in z direction are in adjacent slabs
            
            # Check 6 neighbors
            neighbors = [
                (x-1, y, z, 'left'),
                (x+1, y, z, 'right'),
                (x, y-1, z, 'front'),
                (x, y+1, z, 'back'),
                (x, y, z-1, 'down'),
                (x, y, z+1, 'up')
            ]
            
            for nx_val, ny_val, nz_val, direction in neighbors:
                if not (0 <= nx_val < nx and 0 <= ny_val < ny and 0 <= nz_val < nz):
                    continue
                
                # Determine which slice cache to check for done flags
                # If neighbor is in same slab, check cache_below and cache_above
                # If neighbor is in slab below, check cache_below
                # If neighbor is in slab above, check cache_above
                
                if nz_val == slab_idx:
                    # Same slab
                    # Check if this cube has been processed
                    # We need to check if the cube (nx_val, ny_val, nz_val) is done
                    # This is stored in the slice caches
                    # The cube (nx_val, ny_val, slab_idx) is bounded by slices slab_idx and slab_idx+1
                    # So we check cache_below for 'above' flag and cache_above for 'below' flag
                    # Actually, the done flags are per-corner
                    # We need to check if ANY corner of the cube is done?
                    # No, the done flags are for the cube itself
                    # The algorithm marks the cube as done by marking the corners
                    # But how do we know if the cube is done?
                    # The done flags are for the cubes ABOVE and BELOW the slice
                    # So for cube (x,y,z), we check the done flags in the slice caches
                    # This is complex. Let's simplify:
                    # We add the neighbor to the todo list if it hasn't been processed
                    # We can check if the neighbor is in the todo list or has been processed
                    # For simplicity, we assume that if the neighbor is not in the todo list and not processed, we add it
                    # But we need to avoid duplicates
                    # The done flags prevent re-adding to the todo list
                    
                    # Check cache_below for the cube above (slab_idx)
                    # Check cache_above for the cube below (slab_idx)
                    # This is getting complicated. Let's follow the source logic:
                    # "if do above and ! slice above [x,y] done above then push to slab above todo list"
                    # This implies that the done flags are for the cubes, not the corners
                    # But the source says "done flags to indicate whether the cubes above and below the slice at that location have been rendered"
                    # So for corner (x,y), done_above means the cube above slice (x,y) is done
                    # The cube above slice z is slab z
                    # So for corner (x,y,z), done_above means slab z is done at (x,y)
                    
                    # So for neighbor (nx_val, ny_val, slab_idx), we check:
                    # cache_below.corner_index[nx_val][ny_val].done_above (cube above slice slab_idx is slab_idx)
                    # cache_above.corner_index[nx_val][ny_val].done_below (cube below slice slab_idx+1 is slab_idx)
                    
                    if not cache_below.corner_index[nx_val][ny_val][1]: # done_above
                        current_slab.todo_list.append((nx_val, ny_val))
                    if not cache_above.corner_index[nx_val][ny_val][2]: # done_below
                        current_slab.todo_list.append((nx_val, ny_val))
                        
                elif nz_val == slab_idx - 1:
                    # Neighbor in slab below
                    # Check cache_below for done_below flag
                    if not cache_below.corner_index[nx_val][ny_val][2]: # done_below
                        slabs[nz_val].todo_list.append((nx_val, ny_val))
                        
                elif nz_val == slab_idx + 1:
                    # Neighbor in slab above
                    # Check cache_above for done_above flag
                    if not cache_above.corner_index[nx_val][ny_val][1]: # done_above
                        slabs[nz_val].todo_list.append((nx_val, ny_val))
                        
        # Deallocate slice caches if todo lists are empty
        if len(slabs[slice_below_idx].todo_list) == 0:
            slice_caches[slice_below_idx] = None
        if len(slabs[slice_above_idx].todo_list) == 0:
            slice_caches[slice_above_idx] = None
            
    return triangles

def find_seed_cube(field, isovalue, nx, ny, nz):
    # Find a cube where the isosurface passes through
    for z in range(nz):
        for y in range(ny):
            for x in range(nx):
                # Check if isosurface passes through cube (x,y,z)
                # This requires checking the 8 corners
                # Simplified: check if min and max values straddle isovalue
                vals = [field[x+dx][y+dy][z+dz] for dx in [0,1] for dy in [0,1] for dz in [0,1]]
                if min(vals) <= isovalue <= max(vals):
                    return (x, y, z)
    return None

def compute_corner_position(x, y, z, field, isovalue):
    # Return the 3D position of the corner
    # In a regular grid, this is just (x, y, z)
    return (x, y, z)

def determine_marching_cubes_case(corners, field, isovalue):
    # Determine the case index for Marching Cubes
    # This is a placeholder for the actual logic
    return 0

def generate_triangles(case, corners, cache_below, cache_above):
    # Generate triangles based on the case
    # This is a placeholder for the actual logic
    return []
```

### Edge Cases and Pitfalls
-   **Field Values Unknown A-Priori**: The algorithm cannot march back-to-front. It must traverse connected cubes in arbitrary order [S1].
-   **Seeding Failures**: Seeding algorithms may fail to find all local minima, causing the algorithm to step downwards to render connected cubes in slabs below that were missed. This requires storing additional slices for the abandoned frontier and breaks the unidirectional order of polygonization [S1].
-   **Memory Management**: Cached data not reused must be discarded, and redundant visits to completed cubes must be avoided using the done flags [S1].

## Operating characteristics
-   **Memory Usage**: Significantly less than surface-following marching cubes which keeps all $N_z + 1$ slices in memory. For a $110 \times 110 \times 110$ grid, the cache memory used is below 1MB [S1].
-   **Performance**: Faster than hash-map based approaches due to avoided lookup costs and better cache locality [S1].
-   **Scalability**: Memory usage is independent of the total volume size, depending only on the number of active slices [S1].
-   **Strengths**: Low memory footprint, good cache locality, straightforward parallelization across blocks [S1].
-   **Limitations**: Seeding failures can lead to complex traversal patterns and increased memory usage for abandoned frontiers [S1].

## Background
The Marching Slices algorithm was introduced to address the memory limitations of traditional Marching Cubes for large volumes. It was presented in the paper "Real-time particle isosurface extraction" [S1]. The method is particularly useful for real-time applications where memory bandwidth and capacity are critical constraints.

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
