# Cellular Automata Simulation
Also called: Elementary Cellular Automata, Rule 110, Rule 90, Voxel-based Cellular Automata

## Overview
Cellular automata (CA) simulation generates volume data by evolving discrete grid states through local interaction rules, producing emergent patterns such as fluid flow or geological structures [S1, S2, unofficial]. Unlike physics-based Navier-Stokes solvers, this method relies on deterministic or pseudo-random state transitions rather than differential equations, making it computationally lightweight and suitable for real-time procedural generation [S1, S2, unofficial]. It is distinct from reaction-diffusion systems by its discrete, grid-bound nature and lack of continuous field interpolation [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid representing the current state (e.g., air, water, lava, sand, stone) and a simulation time step $t$.
- Output: A 3D voxel grid representing the state at time step $t+1$.

**Data Structures:**
- `Grid`: A 3D array of voxel types. To handle concurrency on the GPU, two buffers are used: `current_grid` and `next_grid` [S1, unofficial].
- `Direction`: A 2D vector or integer encoding horizontal movement direction (e.g., -1 for left, 1 for right) [S1, unofficial].

**Steps:**
1. **Initialize Buffers:** Copy `current_grid` to `next_grid` or allocate new buffers to prevent race conditions during parallel updates [S1, unofficial].
2. **Process Water:**
   - For each water voxel at position $(x, y, z)$:
     - **Gravity:** Check the voxel directly underneath $(x, y-1, z)$. If it is air, move the water voxel there [S1, unofficial].
     - **Sideways Movement:** If gravity cannot move the voxel, compute a pseudo-random direction using a hash of the position and frame number: $dir = \text{hash}(x, y, z, t) \pmod 2$. If $dir=0$, move left; if $dir=1$, move right [S1, unofficial].
     - **Persistence:** Store the first successful horizontal direction in a persistent state variable for the voxel. Continue moving in this direction until a solid wall is hit. Upon hitting a wall, select a new random direction. Apply a small probability $p_{change}$ to switch direction sideways to simulate natural turbulence [S1, unofficial].
     - **Boundary Check:** Only move if the target voxel is not solid [S1, unofficial].
3. **Process Lava:**
   - Apply the same movement logic as water [S1, unofficial].
   - **Second Pass (Reaction):** For each lava voxel, check its von Neumann neighborhood (up, down, left, right) for water voxels. If any neighbor is water, convert the lava voxel to stone [S1, unofficial].
4. **Process Sand:**
   - For each sand voxel at $(x, y, z)$:
     - **Fall Down:** Check $(x, y-1, z)$. If air, move down [S1, unofficial].
     - **Diagonal Fall:** If obstructed below, check diagonal neighbors $(x-1, y-1, z)$ and $(x+1, y-1, z)$. If either is air, move diagonally [S1, unofficial].
     - **Displacement:** If sand falls onto a liquid (water/lava), push the liquid voxel up to $(x, y+1, z)$ if it is air [S1, unofficial].
5. **Swap Buffers:** Assign `next_grid` to `current_grid` for the next iteration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_cellular_automata(current_grid, next_grid, t):
    # Initialize next_grid as a copy or empty buffer depending on implementation
    # Note: In a real GPU implementation, this is done via kernel launches
    
    for each voxel at (x, y, z) in current_grid:
        type = current_grid[x, y, z]
        
        if type == WATER:
            # Gravity
            if current_grid[x, y-1, z] == AIR:
                next_grid[x, y-1, z] = WATER
                next_grid[x, y, z] = AIR
                continue
            
            # Sideways Movement with Persistence
            # Assume persistent_direction is stored in a separate buffer or state
            dir = persistent_direction[x, y, z]
            
            # If no direction stored or hit wall, pick new one
            if dir == 0: # 0 represents 'none' or 'hit wall'
                hash_val = hash(x, y, z, t)
                if hash_val % 2 == 0:
                    dir = -1
                else:
                    dir = 1
                persistent_direction[x, y, z] = dir
            
            # Try to move in dir
            target_x = x + dir
            if current_grid[target_x, y, z] == AIR:
                next_grid[target_x, y, z] = WATER
                next_grid[x, y, z] = AIR
                # Small probability to change direction for natural look
                if random() < p_change:
                    persistent_direction[x, y, z] = -dir if dir == 1 else 1
            else:
                # Hit wall, reset direction for next frame
                persistent_direction[x, y, z] = 0
                
        elif type == LAVA:
            # Same movement as water (simplified for brevity, assume similar logic)
            # ... (water movement code) ...
            pass
            
            # Reaction Pass: Check von Neumann neighborhood for water
            neighbors = [
                (x+1, y, z), (x-1, y, z),
                (x, y+1, z), (x, y-1, z)
            ]
            for nx, ny, nz in neighbors:
                if current_grid[nx, ny, nz] == WATER:
                    next_grid[x, y, z] = STONE
                    break
                    
        elif type == SAND:
            # Fall down
            if current_grid[x, y-1, z] == AIR:
                next_grid[x, y-1, z] = SAND
                next_grid[x, y, z] = AIR
            elif current_grid[x, y-1, z] in [WATER, LAVA]:
                # Push liquid up
                if current_grid[x, y+1, z] == AIR:
                    next_grid[x, y+1, z] = current_grid[x, y-1, z] # Move liquid up
                    next_grid[x, y, z] = SAND
                    next_grid[x, y-1, z] = AIR
            else:
                # Check diagonals
                if current_grid[x-1, y-1, z] == AIR:
                    next_grid[x-1, y-1, z] = SAND
                    next_grid[x, y, z] = AIR
                elif current_grid[x+1, y-1, z] == AIR:
                    next_grid[x+1, y-1, z] = SAND
                    next_grid[x, y, z] = AIR

    # Swap buffers
    current_grid = next_grid
```

**Parameters:**
- $p_{change}$: Probability of changing horizontal direction for water. Value not given in sources [G: typical value 0.01-0.1].
- Hash function: Any deterministic hash mapping position and time to a binary direction [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Concurrency:** Writing to neighboring voxels simultaneously on the GPU causes race conditions. Using double-buffered textures (current/next) is required for correctness [S1, unofficial].
- **Solid Obstacles:** Movement rules must check for non-air, non-liquid solids to prevent voxels from passing through walls [S1, unofficial].
- **Sand Stacking:** Sand logic must handle diagonal falling to create natural pyramid shapes rather than flat piles [S1, unofficial].

## Subtopics

### Conway's Game of Life
An archetypal 2D cellular automaton where each cell’s next state depends on its eight neighbors according to four rules (birth, survival, death) [S2, unofficial]. It is set apart by its binary state (on/off) and discrete topological neighborhood [S2, unofficial]. Choose it for generating complex, self-organizing patterns like gliders and oscillators [S2, unofficial]. It runs efficiently on the GPU via double-buffered textures in GLSL [S2, unofficial]. For music synchronization, treat stable patterns and gliders as triggers; pre-compute a timeline of cell births aligned with BPM markers to drive particle emission or color shifts [S2, unofficial].

### Elementary Cellular Automata (Rule 110, Rule 90)
One-dimensional cellular automata that generate complex patterns from a single seed row [S2, unofficial]. Rule 110 and Rule 90 are notable for their computational universality and fractal-like structures [S2, unofficial]. Choose them for generating 1D history volumes that can be extruded into 3D [S2, unofficial]. They are computed using bitwise operations from a single seed row, keeping executable size minimal for 64k intros [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per time step, where $N$ is the number of voxels, as each voxel is updated based on a constant number of neighbors [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing two grid buffers (current and next) [S1, unofficial].
- **Performance:** Elementary CAs can run at high frame rates on CPUs (e.g., Processing 4.3 updates a 512x512 grid at 200 fps) and even higher on GPUs (4K resolution in fragment shaders) [S2, unofficial].
- **Strengths:** Extremely low code size (suitable for 64k intros), no need for complex physics solvers, emergent complexity from simple rules [S1, S2, unofficial].
- **Limitations:** Lacks physical accuracy compared to Navier-Stokes; patterns are discrete and grid-aligned; requires careful handling of concurrency on parallel hardware [S1, S2, unofficial].
- **When to use:** Procedural art, demoscene productions, simple fluid/geology simulations where physical accuracy is not critical [S1, S2, unofficial].
- **When not to use:** Realistic fluid dynamics, scientific simulation requiring continuous fields [S2, unofficial].

## Background
Cellular automata were popularized by John Conway’s Game of Life in 1970 [S2, unofficial]. Elementary cellular automata, particularly Rule 110 and Rule 90, have been used in creative coding and the demoscene for generating procedural graphics [S2, unofficial]. The technique is favored in size-constrained environments like 64k intros due to its minimal computational requirements and bitwise optimization potential [S2, unofficial]. It is often compared to L-systems and reaction-diffusion systems in procedural art contexts [S2, unofficial].

## Sources
[S1] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S2] Procedural Graphics: Noise, Fractals and Mathematical Art, https://www.mattcurrent.org/procedural-graphics-guide/, fetched 2026-10-27 (unofficial)
