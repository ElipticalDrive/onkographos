# Online Terrain Generation

Also called: Online approach, Terrain extension

## Overview
An algorithm to extend an already existing terrain by generating new chunks adjacent to the original plot, preserving the density of veins and caves [S1]. Unlike offline generation where layer composition and properties are calculated during generation, online generation calculates the composition and properties of layers beforehand [S1]. This allows the method to seamlessly integrate new data into a persistent world state without regenerating the entire volume [S1].

## How it works
**Inputs:**
*   Existing terrain data (original plot), specifically the count of veins and caves.
*   Target dimensions for the new terrain plot (e.g., chunk size).
*   Input parameters for layer density (used to initialize the first plot's vein/cave density) [S1].

**Outputs:**
*   A new terrain plot consisting of stacked layers, veins, and caves, compatible with the existing terrain [S1].

**Data Structures:**
*   **Terrain Plot:** A 3D volume composed of a vertical stack of elements.
*   **Element Types:**
    *   **Layers:** Defined by material composition and properties (calculated beforehand in online mode) [S1].
    *   **Veins:** Procedural structures generated via specific algorithms [S1].
    *   **Caves:** Procedural voids generated via specific algorithms [S1].
*   **Stack:** An ordered list of elements from bottom to top [S1].

**Steps:**
1.  **Determine Element Counts:** Calculate the number of veins and caves for the new terrain plot. This value is obtained randomly between $0.5$ and $1.5$ times the number of veins and caves in the original plot [S1]. This scaling preserves the density of elements across the whole terrain [S1].
2.  **Determine Positions:** Determine the position of veins and caves between the layers using a completely random process [S1]. The result is a stack of layers, veins, and caves [S1].
3.  **Generate Layers:** Generate elements of the stack beginning from the bottom [S1]. Extend layers in the horizontal plane (along X and Z coordinates) obtaining new heights using the Perlin noise algorithm [S1]. Note that the composition and properties of the layer are already calculated in this step [S1].
4.  **Generate Veins and Caves:** Generate veins and caves following the same steps as the offline generation process (refer to [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md) for details) [S1].
5.  **Convert to Volume (Optional/Contextual):** If converting the terrain to a volume representation, multiply the thickness by the blend value of each layer to determine the transition height with the last layer, removing sharp frontiers [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_online_terrain(existing_terrain, target_chunk_size):
    # Step 1: Determine number of veins and caves
    # Random factor between 0.5 and 1.5
    # Preserve density relative to the original plot
    original_vein_count = existing_terrain.count_veins()
    original_cave_count = existing_terrain.count_caves()
    
    random_factor_veins = random.uniform(0.5, 1.5)
    random_factor_caves = random.uniform(0.5, 1.5)
    
    new_vein_count = int(original_vein_count * random_factor_veins)
    new_cave_count = int(original_cave_count * random_factor_caves)
    
    # Step 2: Determine positions (completely random process)
    # Result is a stack of layers, veins, and caves
    stack = create_random_stack(new_vein_count, new_cave_count, target_chunk_size)
    
    # Step 3: Generate elements from bottom up
    for element in stack:
        if element.type == "layer":
            # Composition and properties are already calculated
            # Extend in horizontal plane (X, Z) using Perlin noise
            element.height = generate_layer_height_perlin_noise(element, target_chunk_size)
        elif element.type == "vein":
            # Follow same steps as offline generation
            generate_vein_structure(element)
        elif element.type == "cave":
            # Follow same steps as offline generation
            generate_cave_structure(element)
            
    return stack

def generate_layer_height_perlin_noise(layer, chunk_size):
    # Use Perlin noise algorithm to obtain new heights
    # in the horizontal plane (X and Z coordinates)
    heights = []
    for x in range(chunk_size.x):
        row = []
        for z in range(chunk_size.z):
            # Perlin noise call [G]
            h = perlin_noise(x, z, layer.seed)
            row.append(h)
        heights.append(row)
    return heights
```

**Parameters:**
*   **Random Factor Range:** $[0.5, 1.5]$ for vein and cave counts [S1].
*   **Noise Algorithm:** Perlin noise for layer height generation [S1].

**Edge Cases and Pitfalls:**
*   **Density Preservation:** The random scaling factor must be applied to the *original* plot's counts to ensure density is preserved along the whole terrain, not just the current chunk [S1].
*   **Composition Calculation:** In online mode, layer composition and properties must be calculated *beforehand*, unlike offline generation where they are calculated during generation [S1]. Failure to pre-calculate these will break the online extension logic.

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in terms of Big-O. Measured performance shows online generation times are comparable to offline generation.
*   **Measured Performance:** On an Intel Core i7 950 CPU (3.07GHz), generating 100 terrains of 5x5x5 chunks of 10x10x10 voxels:
    *   Offline average time: 319.14 ms to 354.30 ms (std dev 41.94 ms to 75.01 ms) [S1].
    *   Online average time: 309.24 ms to 342.84 ms (std dev 32.20 ms to 60.35 ms) [S1].
    *   Baseline Perlin Noise generation: 61.25 ms (std dev 14.46 ms) [S1].
*   **Strengths:** Can generate terrains with complex material configurations online; preserves vein and cave density; similar performance to offline generation [S1].
*   **Limitations:** Generating terrains with caves increases computing overhead; vein generation is the most time-consuming configurable feature [S1].
*   **When to use:** When extending an existing terrain volume without regenerating the entire dataset.
*   **When not to use:** When generating a terrain from scratch (use offline generation) or when high performance is required and vein/cave complexity is high (consider simpler noise-based methods).

## Background
The method was proposed by Aitor Santamaría-Ibirika et al. in the paper "Procedural approach to volumetric terrain generation" [S1]. It is part of a broader procedural approach to volumetric terrain generation that includes offline generation, online extension, and conversion to volume representations [S1]. The method was tested on an Intel Core i7 950 CPU [S1].

## Sources
[S1] Procedural approach to volumetric terrain generation, https://zenodo.org/records/18429902, fetched 2026-10-27
