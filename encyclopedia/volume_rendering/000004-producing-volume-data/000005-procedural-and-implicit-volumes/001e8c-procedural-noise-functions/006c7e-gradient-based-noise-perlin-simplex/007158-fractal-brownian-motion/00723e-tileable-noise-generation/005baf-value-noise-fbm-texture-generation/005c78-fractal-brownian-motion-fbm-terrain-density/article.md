# Fractal Brownian Motion (FBM) Terrain Density

Also called: Octave Noise Terrain Generation

## Overview
Fractal Brownian Motion (FBM) Terrain Density generates a 3D scalar field representing terrain density by summing multiple octaves of simplex noise, modified by a base surface plane, terracing, and bedrock layers [S1, unofficial]. It differs from standard Value-Noise FBM Texture Generation by explicitly modeling volumetric terrain structure for solid geometry rather than 2D surface texture, using a specific noise transformation ($1 - |val|$ squared) to create sharp peaks suitable for mountainous topography [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A grid of vertex indices, chunk geometry parameters (`chunkSize`, `vertSpacing`, `center`), noise parameters (`scale`, `lacunarity`, `persistence`, `weight`, `weightMultiplier`), and structural parameters (`surfaceOffset`, `bedrockHeight`, `bedrockWeight`, `terracing`, `solidifyEdges`, `mapSize`).
*   **Outputs:** `vertexBuffer` (position of each vertex) and `densityBuffer` (density value at each vertex).

**Data Structures:**
The method uses a job struct `TerrainDensityJob` with the following fields:
*   `octaveOffsetBuffer`: `NativeArray<Vector3>`, ReadOnly. Stores the offset vector for each octave.
*   `vertexBuffer`: `NativeArray<Vector3>`. Stores the 3D position of each vertex.
*   `densityBuffer`: `NativeArray<float>`. Stores the computed density for each vertex.
*   `numberOfOctaves`: `int`. Number of noise octaves to sum.
*   `numVertsPerAxis`: `int`. Number of vertices along one axis of the chunk.
*   `chunkSize`: `float`. Size of the terrain chunk.
*   `vertSpacing`: `float`. Distance between adjacent vertices.
*   `lacunarity`: `float`. Frequency multiplier per octave.
*   `persistence`: `float`. Amplitude multiplier per octave.
*   `scale`: `float`. Initial frequency multiplier.
*   `weight`: `float`. Multiplier for the noise density component before adding to the surface plane.
*   `surfaceOffset`: `float`. Vertical offset for the base surface plane.
*   `weightMultiplier`: `float`. Controls how quickly octave weight decays based on the previous octave's value.
*   `bedrockHeight`: `float`. Y-coordinate threshold below which bedrock weight is added.
*   `bedrockWeight`: `float`. Value added to density if vertex is below bedrock height.
*   `center`: `Vector3`. Center of the terrain chunk.
*   `offset`: `Vector3`. Global offset for noise evaluation.
*   `mapSize`: `Vector3`. Dimensions of the map for edge solidification.
*   `terracing`: `Vector3`. `x` = terrace height step, `y` = terrace weight multiplier, `z` = enabled flag (1 if enabled).
*   `solidifyEdges`: `bool`. If true, vertices outside the map bounds are set to a solid density.

**Algorithm Steps:**

1.  **Initialize Vertex Position:**
    Calculate the offset from the center: `offsetPos = Vector3(1, 1, 1) * chunkSize / 2`.
    Calculate the vertex position from its index `id`:
    $$position = center - offsetPos + ToVector3(id) * vertSpacing$$
    Where `ToVector3(id)` converts the linear vertex index into 3D coordinates.

2.  **Initialize FBM Variables:**
    $$density = 0$$
    $$frequency = scale / 100$$
    $$amplitude = 1$$
    $$octaveWeight = 1$$

3.  **Loop Over Octaves:**
    For each octave $i$ from $0$ to $numberOfOctaves - 1$:
    a.  **Evaluate Noise:**
        $$val = SimplexNoise.Evaluate(position \times frequency + octaveOffsetBuffer[i] + offset)$$
    b.  **Transform Value:**
        The value is transformed to create peaks at noise zeros:
        $$val = 1 - |val|$$
        $$val = val \times val$$
    c.  **Apply Weight:**
        $$val = val \times octaveWeight$$
    d.  **Accumulate Density:**
        $$density = density + val \times amplitude$$
    e.  **Update Octave Weight:**
        The weight for the next octave depends on the current octave's transformed value:
        $$octaveWeight = \max(\min(val \times weightMultiplier, 1), 0)$$
    f.  **Update Amplitude:**
        $$amplitude = amplitude \times persistence$$
    g.  **Update Frequency:**
        $$frequency = frequency \times lacunarity$$

4.  **Combine with Surface Plane:**
    The density is adjusted relative to a base surface plane:
    $$density = -(position.y + surfaceOffset) + density \times weight$$

5.  **Apply Terracing (Optional):**
    If `terracing.z > 0`:
    $$density = density + (position.y \pmod{terracing.x}) \times terracing.y$$

6.  **Apply Bedrock Layer (Optional):**
    If `position.y < bedrockHeight`:
    $$density = density + bedrockWeight$$

7.  **Solidify Edges (Optional):**
    If `solidifyEdges` is true:
    Calculate the distance from the center in each axis:
    $$edgeOffset = Vector3(|position.x|, |position.y|, |position.z|) - mapSize / 2$$
    If any component of `edgeOffset` is $\ge 0$:
    $$density = -100$$

8.  **Store Results:**
    $$vertexBuffer[id] = position$$
    $$densityBuffer[id] = density$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_terrain_density(job, id):
    # 1. Initialize Vertex Position
    offset_pos = Vector3(1, 1, 1) * job.chunk_size / 2
    position = job.center - offset_pos + to_vector3(id) * job.vert_spacing

    # 2. Initialize FBM Variables
    density = 0.0
    frequency = job.scale / 100.0
    amplitude = 1.0
    octave_weight = 1.0

    # 3. Loop Over Octaves
    for i in range(job.number_of_octaves):
        # Evaluate Noise
        noise_input = position * frequency + job.octave_offset_buffer[i] + job.offset
        val = simplex_noise.evaluate(noise_input)
        
        # Transform Value
        val = 1.0 - abs(val)
        val = val * val
        
        # Apply Weight
        val = val * octave_weight
        
        # Accumulate Density
        density += val * amplitude
        
        # Update Octave Weight
        octave_weight = max(min(val * job.weight_multiplier, 1.0), 0.0)
        
        # Update Amplitude
        amplitude *= job.persistence
        
        # Update Frequency
        frequency *= job.lacunarity

    # 4. Combine with Surface Plane
    density = -(position.y + job.surface_offset) + density * job.weight

    # 5. Apply Terracing
    if job.terracing.z > 0:
        terrace_step = job.terracing.x
        terrace_weight = job.terracing.y
        density += (position.y % terrace_step) * terrace_weight

    # 6. Apply Bedrock Layer
    if position.y < job.bedrock_height:
        density += job.bedrock_weight

    # 7. Solidify Edges
    if job.solidify_edges:
        edge_offset_x = abs(position.x) - job.map_size.x / 2
        edge_offset_y = abs(position.y) - job.map_size.y / 2
        edge_offset_z = abs(position.z) - job.map_size.z / 2
        
        if edge_offset_x >= 0 or edge_offset_y >= 0 or edge_offset_z >= 0:
            density = -100.0

    # 8. Store Results
    job.vertex_buffer[id] = position
    job.density_buffer[id] = density
```

**Parameters:**
*   `scale`: Initial frequency multiplier. Typical values depend on world scale, but is divided by 100 in the job [S1, unofficial].
*   `lacunarity`: Frequency multiplier per octave. Typical value is 2.0 [G].
*   `persistence`: Amplitude multiplier per octave. Typical value is 0.5 [G].
*   `weight`: Multiplier for the noise density component. Controls the prominence of noise features relative to the surface plane [S1, unofficial].
*   `weightMultiplier`: Controls decay of octave weight. Higher values allow deeper octaves to contribute more [S1, unofficial].
*   `surfaceOffset`: Vertical offset for the base surface plane [S1, unofficial].
*   `bedrockHeight`: Y-coordinate threshold for bedrock [S1, unofficial].
*   `bedrockWeight`: Density value added below bedrock height [S1, unofficial].
*   `terracing`: Vector3 controlling terraced terrain features [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The transformation $1 - |val|$ followed by squaring creates sharp peaks. If `weightMultiplier` is too high, deep octaves may dominate, causing noise artifacts. If too low, the terrain may appear flat [S1, unofficial].
*   **Boundaries:** The `solidifyEdges` check uses absolute position values. If the terrain chunk is not centered at the origin or `mapSize` is incorrect, edges may be incorrectly solidified or left hollow [S1, unofficial].
*   **Degenerate Input:** If `numberOfOctaves` is 0, density is determined solely by the surface plane and bedrock/terracing layers [S1, unofficial].

## Subtopics

### [Multifractal Mountain Synthesis](17635-multifractal-mountain-synthesis.md)
Generates terrain height fields using ridged and hybrid multifractal components with domain warping. Chose when sharper peaks and more realistic geological features are needed compared to standard additive FBM.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot O)$, where $N$ is the number of vertices and $O$ is the number of octaves. Each vertex requires $O$ noise evaluations.
*   **Memory Complexity:** $O(N)$ for storing vertex positions and density values. The `octaveOffsetBuffer` requires $O(O)$ memory.
*   **Performance:** Performance depends heavily on the efficiency of the `SimplexNoise.Evaluate` function. Parallelization via job structs (e.g., Unity Job System) is implied by the data structure layout [S1, unofficial].
*   **Strengths:** Simple to implement, computationally efficient, allows for detailed control over terrain features via octaves and transformations.
*   **Limitations:** The specific noise transformation ($1 - |val|$ squared) is non-standard and may not produce natural-looking terrain without careful tuning of parameters. Lacks the geological realism of multifractal methods [S1, unofficial].
*   **When to Use:** When a simple, fast, and tunable procedural terrain density field is required, and the specific peak-forming noise transformation is desired.
*   **When Not to Use:** When realistic geological features or height-field-based terrain generation is required; use Multifractal Mountain Synthesis instead [S1, unofficial].

## Background
This method is described in the unofficial source `tcgj/terrain-generator` [S1, unofficial]. It represents a specific implementation of FBM for terrain generation, distinct from standard value-noise FBM texture generation by its focus on 3D volumetric density and specific noise transformations for peak formation.

## Sources
[S1] tcgj/terrain-generator, https://github.com/tcgj/terrain-generator, fetched 2026-10-27 (unofficial)
