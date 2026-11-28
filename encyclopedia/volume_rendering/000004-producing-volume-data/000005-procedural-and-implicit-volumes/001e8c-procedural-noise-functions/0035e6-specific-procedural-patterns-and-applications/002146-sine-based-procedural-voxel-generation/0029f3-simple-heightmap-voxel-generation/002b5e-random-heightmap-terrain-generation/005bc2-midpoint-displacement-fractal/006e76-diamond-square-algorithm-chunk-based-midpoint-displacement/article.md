# Diamond-Square Algorithm (Chunk-based Midpoint Displacement)

Also called: DSNoise, Midpoint Displacement

## Overview
The Diamond-Square Algorithm (Chunk-based Midpoint Displacement) is a procedural world generation method that recursively subdivides a 2D grid into squares and diamonds to compute new cell values as the average of surrounding cells plus a scaled random perturbation [S1, unofficial]. It operates asynchronously on chunks to generate a continuous, infinite world model, distinguishing itself from standard implementations by its spatial partitioning and lazy evaluation strategy [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A chunk coordinate (implied by the asynchronous context), a global `WorldSeed`, and a `sampleLevel` integer scaling factor [S1, unofficial].
*   **Output:** A populated 2D array of `DSDataCase` instances representing the terrain heightmap for that chunk [S1, unofficial].

**Data Structures**
*   **`DSDataCase`**: Stores integer coordinates `(x, y)` and a float `Value`. The coordinates are instantiated as `new DSDataCase(x * this.SampleLevel, y * this.SampleLevel)` to map chunk-local grid indices to world coordinates [S1, unofficial].
*   **`casesArray`**: A 2D array `casesArray[y, x]` storing `DSDataCase` instances. It is initialized in `PrepareChunk` at index `[0,0]` and populated during generation loops [S1, unofficial].
*   **`CurrentStepLength`**: An integer representing the grid spacing between generated points at the current iteration. It halves each step [S1, unofficial].

**Parameters**
*   **`WorldSeed`**: A global integer seed combined with the parent layer ID hash to generate deterministic chunk seeds [S1, unofficial].
*   **`sampleLevel`**: Integer scaling factor applied to local coordinates [S1, unofficial].
*   **`CurrentStepLength`**: Grid spacing at the current iteration [S1, unofficial].

**Algorithm Steps**
1.  **Initialization (`PrepareChunk`)**: Generate the center case at local coordinates `[0,0]` with a value sampled uniformly from `[0.25, 0.75]` [S1, unofficial].
2.  **Iteration Control (`GenerateChunk`)**: Determine step parity using `bool isCurrentStepSquare = currentNbStep % 2 == 1`. Calculate `midStepLength = dataChunkLayer.CurrentStepLength / 2` [S1, unofficial].
3.  **Square-Step Traversal**: If `isCurrentStepSquare` is true, loop `i` from `0` to `secondStepPower - 1` and `j` from `0` to `CurrentStepPower - 1`. Compute `yCaseToGenerate = i * midStepLength` and `xCaseToGenerate = midStepLength * ((i + 1) % 2) + j * dataChunkLayer.CurrentStepLength`. Call `GenerateCase` for each coordinate [S1, unofficial].
4.  **Diamond-Step Traversal**: If `isCurrentStepSquare` is false, loop `i` and `j` from `0` to `CurrentStepPower - 1`. Compute `yCaseToGenerate = midStepLength + i * dataChunkLayer.CurrentStepLength` and `xCaseToGenerate = midStepLength + j * dataChunkLayer.CurrentStepLength`. Call `GenerateCase` for each coordinate [S1, unofficial].
5.  **Neighbor Retrieval (`GetCasesSummit`)**: Fetch the four surrounding cases based on step parity:
    *   **Square Step**: `(x, y-midStepLength)`, `(x+midStepLength, y)`, `(x-midStepLength, y)`, `(x, y+midStepLength)` [S1, unofficial].
    *   **Diamond Step**: `(x-midStepLength, y-midStepLength)`, `(x+midStepLength, y-midStepLength)`, `(x-midStepLength, y+midStepLength)`, `(x+midStepLength, y+midStepLength)` [S1, unofficial].
6.  **Value Computation (`GenerateCaseFrom`)**: Calculate the new case value using the formula below. Return the new `DSDataCase` instance [S1, unofficial].

**Formula**
The new value $V_{new}$ is calculated as:
$$V_{new} = \frac{V_{TL} + V_{TR} + V_{BL} + V_{BR}}{4} + R \cdot \left(\frac{2}{S + 3}\right)^2 \cdot 0.75$$
Where:
*   $V_{TL}, V_{TR}, V_{BL}, V_{BR}$ are the values of the top-left, top-right, bottom-left, and bottom-right neighbor cases [S1, unofficial].
*   $R \in [-1, 1]$ is a random perturbation [S1, unofficial].
*   $S$ is the current iteration step number [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk(chunk_seed, sample_level, world_seed):
    # Initialization
    # Note: Exact determination of 'secondStepPower' and 'CurrentStepPower' 
    # from chunk size is not fully specified in sources [G]. 
    # Assuming a standard chunk size of N x N points, powers are derived from log2(N).
    # For this implementation, we assume a fixed max iteration depth or dynamic sizing.
    
    cases_array = {} # Dictionary mapping (x, y) to DSDataCase
    current_step_length = 1 # Initial step length for the center point context
    
    # 1. Initialization: Center case at [0,0]
    center_value = random.uniform(0.25, 0.75)
    cases_array[(0, 0)] = DSDataCase(x=0, y=0, value=center_value)
    
    # Determine iteration steps. 
    # In a chunk-based system, steps continue until the grid covers the chunk.
    # The source implies a loop structure based on step parity.
    # We simulate the recursive subdivision logic within the chunk bounds.
    
    step = 0
    while current_step_length < chunk_size:
        is_current_step_square = (step % 2 == 1)
        mid_step_length = current_step_length / 2
        
        if is_current_step_square:
            # Square-step traversal
            # Loop bounds derived from source: i from 0 to secondStepPower - 1, j from 0 to CurrentStepPower - 1
            # Mapping this to a grid: we iterate over the centers of the squares formed by the previous diamond steps.
            # The source formula for xCaseToGenerate suggests a staggered grid.
            for i in range(0, chunk_size // current_step_length):
                for j in range(0, chunk_size // current_step_length):
                    y_case = i * mid_step_length
                    x_case = mid_step_length * ((i + 1) % 2) + j * current_step_length
                    
                    # Check bounds and generate
                    if is_within_chunk(x_case, y_case, chunk_size):
                        generate_case(cases_array, x_case, y_case, mid_step_length, step, world_seed)
        else:
            # Diamond-step traversal
            # Loop i and j from 0 to CurrentStepPower - 1
            for i in range(0, chunk_size // current_step_length):
                for j in range(0, chunk_size // current_step_length):
                    y_case = mid_step_length + i * current_step_length
                    x_case = mid_step_length + j * current_step_length
                    
                    # Check bounds and generate
                    if is_within_chunk(x_case, y_case, chunk_size):
                        generate_case(cases_array, x_case, y_case, mid_step_length, step, world_seed)
        
        current_step_length *= 2
        step += 1

    return cases_array

def generate_case(cases_array, x, y, mid_step_length, step, world_seed):
    # 5. Neighbor Retrieval
    neighbors = get_neighbors(cases_array, x, y, mid_step_length, step)
    
    if len(neighbors) < 4:
        # Boundary handling: If out of bounds or missing neighbors, skip or handle gracefully
        # Source says: "skip generation for that point" if lookup fails
        return None
    
    v_tl, v_tr, v_bl, v_br = neighbors
    
    # 6. Value Computation
    # Formula: V_new = (V_TL + V_TR + V_BL + V_BR) / 4 + R * (2 / (S + 3))^2 * 0.75
    mean_val = (v_tl + v_tr + v_bl + v_br) / 4.0
    r = random.uniform(-1, 1)
    s = step
    perturbation = r * ((2.0 / (s + 3)) ** 2) * 0.75
    
    new_value = mean_val + perturbation
    
    # Instantiate DSDataCase with sample_level scaling
    scaled_x = x * sample_level
    scaled_y = y * sample_level
    
    cases_array[(x, y)] = DSDataCase(x=scaled_x, y=scaled_y, value=new_value)
    return new_value

def get_neighbors(cases_array, x, y, mid_step_length, step):
    is_current_step_square = (step % 2 == 1)
    
    if is_current_step_square:
        # Square Step Neighbors
        coords = [
            (x, y - mid_step_length),
            (x + mid_step_length, y),
            (x - mid_step_length, y),
            (x, y + mid_step_length)
        ]
    else:
        # Diamond Step Neighbors
        coords = [
            (x - mid_step_length, y - mid_step_length),
            (x + mid_step_length, y - mid_step_length),
            (x - mid_step_length, y + mid_step_length),
            (x + mid_step_length, y + mid_step_length)
        ]
    
    values = []
    for nx, ny in coords:
        # Boundary handling: GetCaseAtLocal checks if x < 0 or x >= realNbCaseSide
        # If out of bounds, it wraps coordinates using realNbCaseSide and attempts to fetch from adjacent chunks
        # If adjacent chunk is null or lookup fails, return null -> skip generation
        if is_within_chunk(nx, ny, chunk_size):
            if (nx, ny) in cases_array:
                values.append(cases_array[(nx, ny)].value)
            else:
                # Neighbor not yet generated in this chunk (should not happen in correct order)
                # or boundary condition. 
                # In a strict chunk implementation, missing neighbors might require fetching from ExtendedChunksMonitor
                # Source: "attempts to fetch from adjacent chunks via ExtendedChunksMonitor.GetChunkContainerAt"
                # For this self-contained pseudocode, we assume neighbors are available or return None to skip
                return None 
        else:
            # Out of local chunk bounds. 
            # Source implies wrapping or fetching from adjacent chunks.
            # If we cannot resolve the neighbor, we skip.
            return None
            
    if len(values) == 4:
        return values
    else:
        return None

def is_within_chunk(x, y, chunk_size):
    # Simplified boundary check. 
    # Source: "GetCaseAtLocal checks if x < 0 or x >= realNbCaseSide"
    return 0 <= x < chunk_size and 0 <= y < chunk_size
```

**Edge Cases and Pitfalls**
*   **Boundary Handling**: `GetCaseAtLocal` checks if `x < 0` or `x >= realNbCaseSide` (and same for y). If out of bounds, it wraps coordinates using `realNbCaseSide` and attempts to fetch from adjacent chunks via `ExtendedChunksMonitor.GetChunkContainerAt`. If the adjacent chunk is null or the final lookup fails, it returns null, causing `GetCasesSummit` to return false and skip generation for that point [S1, unofficial].
*   **Precision**: The random perturbation scales with $(S+3)^{-2}$. For large $S$, the perturbation becomes very small, potentially leading to flat terrain if floating-point precision is insufficient [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N^2)$ for a grid of size $N \times N$, as each cell is computed once. In a chunk-based system, complexity is proportional to the chunk area.
*   **Memory Complexity**: $O(N^2)$ to store the `casesArray`.
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Asynchronous chunk generation allows for infinite world generation without precomputing the entire map. Deterministic seeding ensures consistency.
*   **Limitations**: Boundary handling between chunks requires complex neighbor fetching logic. The specific scaling factor $(2/(S+3))^2$ is non-standard compared to the typical $2^{-H \cdot S}$ fractal noise, potentially affecting the fractal dimension [S1, unofficial].

## Background
The Diamond-Square Algorithm is a classic procedural generation technique for heightmaps. The specific implementation described here is from an unofficial project "Deneyr/WorldGeneration" used in a Pokemon game core to generate chunks of a procedural world [S1, unofficial]. It does not manage game logic or rendering, its only purpose is to generate world chunks asynchronously in a given area [S1, unofficial].

## Sources
[S1] Deneyr/WorldGeneration, https://github.com/Deneyr/WorldGeneration, fetched 2026-10-27 (unofficial)
