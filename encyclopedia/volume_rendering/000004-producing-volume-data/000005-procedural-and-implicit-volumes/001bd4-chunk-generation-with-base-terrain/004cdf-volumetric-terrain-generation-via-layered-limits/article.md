# Volumetric Terrain Generation via Layered Limits

Also called: Layered Limits Terrain Generation, Terrain to Volume Conversion

## Overview
Volumetric Terrain Generation via Layered Limits is a procedural method that converts a terrain composed of stacked layers, veins, and caves into a volume representation by defining mixed areas between layers to remove sharp frontiers [S1]. It represents data as columns subdivided into chunks, where each column consists of stacked material limits that define volume properties at specific heights [S1]. This method sets itself apart from sibling methods like [Procedural Terrain Heightmap Generation](1850-procedural-terrain-heightmap-generation.md) by storing full volumetric material composition and flow vectors rather than just surface height, and from [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md) by using a limit-based interpolation system that dynamically generates voxels based on proximity to layer boundaries rather than fixed grid assignments [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Configuration File:** Contains three sections:
    1.  **Material Data:** For each material: Name, Id (unique integer), Color (RGB), Textures (path), Special Parameters (Void, Base, Alone, Vein, Cave), Cave/vein thickness (absolute value conversion).
    2.  **Affinity Data:** A matrix or list of pairs defining compatibility between materials. Values are on a 0-1 scale for compatible materials, -1 for incompatible, and 0.0 for materials not prone to mix but blendable by other reasons [S1].
    3.  **Global Parameters:** Terrain depth, Layer thickness (absolute value conversion), Cave and vein area (bounding square size limits), Scale (frequency for Perlin noise) [S1].
*   **Chunk Dimensions:** Defined by $N \times L$ columns, where each column represents $M$ voxels [S1].

**Outputs:**
*   **Volumetric Data:** A dynamic voxel grid where each voxel contains:
    *   **Material Composition:** A list of materials with associated concentrations (summing to 1).
    *   **Material Flow:** A 3D vector representing internal material flow [S1].
*   **Limit Structure:** A set of limits stored per column, defining the height and properties of material transitions [S1].

### Data Structures

**Limit:**
A structure representing a transition boundary between materials at a specific height within a column.
*   `height`: Float. The vertical position of the limit.
*   `materials`: List of Material IDs.
*   `concentrations`: List of Floats. Corresponding concentrations for each material in the list. Sum must be 1.
*   `flow`: Vector3. The material flow vector tangent to the layer surface.
*   `properties`: Dictionary/Struct containing derived properties: `thickness`, `similarity`, `blend_value`, `roughness` [S1].

**Column:**
A vertical stack of limits within a chunk.
*   `limits`: List of Limit objects.
    *   Minimum 2 limits (top and bottom).
    *   Maximum $M$ limits (one per voxel height step, though typically fewer) [S1].

**Chunk:**
*   `grid_dimensions`: $N \times L$ (number of columns).
*   `columns`: Array of Column objects [S1].

### Parameters and Constants
*   `VeinArea`: Global parameter defining the maximum side length of the vein bounding square [S1].
*   `CaveArea`: Global parameter defining the maximum side length of the cave bounding square [S1] (used instead of `VeinArea`) [S1].
*   `SmoothConstant`: Taken from material roughness parameter. Controls roughness loss in fractal generation [S1].
*   `d0`: Constant for max displacement at depth 0, taken from material roughness parameter [S1].
*   `Recursive Depth Limit`: 5 (for time performance) [S1].
*   `Main Material Concentration Range`: [0.5, 0.9] [S1].

### Algorithm Steps

#### 1. Column Subdivision
Subdivide the chunk into an $N \times L$ regular grid of columns. Each column represents $M$ voxels vertically [S1].

#### 2. Layer Generation
For each column, generate stacked layers from bottom to top.

**2.1 Material Selection**
1.  Identify materials from the last generated layer.
2.  Filter out materials with affinity -1 to any material in the last layer [S1].
3.  Calculate Global Affinity $A_i$ for each possible material $i$:
    $$ A_i = \sum_{j=1}^{N} (a_{ij} \cdot c_j) $$
    Where:
    *   $N$ is the number of materials in the last layer.
    *   $a_{ij}$ is the affinity between possible material $i$ and material $j$ of the last layer.
    *   $c_j$ is the concentration of material $j$ in the last layer [S1].
4.  Calculate Probability $P_i$ for each material $i$:
    $$ P_i = \frac{A_i}{\sum_{j=1}^{N} A_j} $$
    Select the next layer's materials based on these probabilities [S1].

**2.2 Concentration Assignment**
1.  Ensure the sum of concentrations in the layer is 1.
2.  If the layer has $\ge 2$ materials, assign the main material a random concentration in $[0.5, 0.9]$.
3.  Assign random concentrations to remaining materials, normalizing so the total sum is 1 [S1].

**2.3 Property Calculation**
Calculate layer properties ($V$) using the weighted sum of material properties ($v_i$) and concentrations ($c_i$):
$$ V = \sum_{i=1}^{N} (v_i \cdot c_i) $$
Where $V$ is the property value (e.g., thickness, similarity, roughness), $N$ is the number of materials, $v_i$ is the property value of material $i$, and $c_i$ is its concentration [S1].

**2.4 Height Calculation**
1.  Calculate modified height $h$:
    $$ h = h_{-1} \cdot s + n \cdot (1 - s) $$
    Where:
    *   $h_{-1}$ is the height of the last layer at the same point.
    *   $s$ is the similarity calculated in step 2.3.
    *   $n$ is the Perlin noise value (scaled by the global `Scale` parameter) [S1].
2.  Calculate final height: Add the layer's thickness and the average height of the previous layer to $h$ [S1].

**2.5 Flow Generation**
Generate underground material flow tangent to the layer surface. Rotate the flow vector on the Y-axis based on horizontal 2D Perlin noise [S1].

**2.6 Limit Creation**
Create a Limit object for the column with the calculated height, materials, concentrations, flow, and properties. Repeat until the terrain depth is reached or the maximum number of limits ($M$) is exceeded [S1].

#### 3. Vein Generation
Veins are generated independently of layers but within the column structure.

**3.1 Initialization**
1.  Select a random location within the terrain plot [S1].
2.  Determine starting bounding square side length: Random value between `VeinArea/2` and `VeinArea` [S1].

**3.2 Material Selection**
1.  Select materials with the `Vein` special parameter [S1].
2.  Select only materials from the last layer located under the vein's region [S1].
3.  Select the main material among those with the `Vein` parameter [S1].

**3.3 Property Calculation**
Calculate relative area, roughness, and relative thickness using the same process as Layer Generation (Steps 2.2-2.3) [S1].

**3.4 Shape Generation (Fractal)**
1.  Calculate final bounding square: Multiply relative area by the base bounding square [S1].
2.  Generate relative shape using midpoint displacement:
    *   Locate a square in 2D space.
    *   Displace vertices ($P_1, P_2, P_3, P_4$) along diagonals.
    *   Take a random point $P_0$ from a segment.
    *   Displace $P_0$ along the line formed by $P_0$ and the square center [S1].
    *   Recursively repeat for all segments.
    *   Max displacement at depth $i$:
        $$ d_i = \frac{d_{i-1} / \text{SmoothConstant}}{2} $$
        Where $d_0$ is a constant, and $d_{i-1}$ is the max displacement at depth $i-1$. Minimum displacement is $-d_i$. Recursive depth limited to 5 [S1].

**3.5 Height Calculation**
1.  Calculate normalized minimum distance $d_{xy}$ from each point $[x,y]$ to the fractal border (range 0-1) [S1].
2.  Calculate height $h_{xy}$:
    $$ h_{xy} = \left| 1 - (d_{xy} - 1)^2 \right| $$
    This simplifies a unit radius circumference centered at $[1,0]$ [S1].
3.  Final vein height: Multiply $h_{xy}$ by relative thickness and global thickness, then add the height of the last layer [S1].

**3.6 Flow Generation**
Generate flow using the same process as Layer Generation (Step 2.5) [S1].

**3.7 Limit Creation**
Create Limit objects for the vein within the affected columns [S1].

#### 4. Cave Generation
Follows Vein Generation steps with exceptions:
1.  Use `CaveArea` instead of `VeinArea` for bounding square size [S1].
2.  Select only one material from the set with the `Cave` special parameter [S1].
3.  Extract material properties to modify the bounding square [S1].
4.  Generate shape and height using the same fractal and distance methods as veins [S1].

#### 5. Volume Conversion (Interpolation)
Convert the limit-based representation to voxels.

**5.1 Dynamic Voxel Generation**
For each voxel at chunk, column, and height $H$:
1.  Check if $H$ matches the height of any limit in the column.
2.  If yes, use the limit's data directly [S1].
3.  If no, interpolate between the surrounding limits (e.g., the limit immediately below and the one immediately above) [S1].

**5.2 Blending**
1.  Multiply the thickness of each layer by its blend value.
2.  The result is the height where the transition with the last layer ends.
3.  This process removes sharp frontiers, mixing materials and flow for a realistic appearance [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class Material:
    def __init__(self, name, id, color, textures, special_params, affinity_matrix):
        self.name = name
        self.id = id
        self.color = color
        self.textures = textures
        self.special_params = special_params  # Set of {'Void', 'Base', 'Alone', 'Vein', 'Cave'}
        self.affinity_matrix = affinity_matrix # Dict: (other_id) -> affinity_value

class Limit:
    def __init__(self, height, materials, concentrations, flow, properties):
        self.height = height
        self.materials = materials
        self.concentrations = concentrations
        self.flow = flow
        self.properties = properties # Dict: {'thickness', 'similarity', 'blend_value', 'roughness'}

class Column:
    def __init__(self):
        self.limits = []

def generate_terrain(chunk_n, chunk_l, chunk_m, config):
    """
    chunk_n, chunk_l: Grid dimensions
    chunk_m: Voxels per column
    config: Dict with 'materials', 'affinities', 'global_params'
    """
    columns = [[Column() for _ in range(chunk_l)] for _ in range(chunk_n)]
    
    # Initialize bottom limits for all columns
    base_materials = [m for m in config['materials'] if 'Base' in m.special_params]
    for col in columns:
        for c in col:
            # Simplified: Assume single base material or mixed base
            # In reality, select based on affinity/properties
            c.limits.append(Limit(0, base_materials, [1.0], Vector3(0,0,0), {'thickness': 0, 'similarity': 0, 'roughness': 0}))

    # Generate Layers
    for col_idx in range(chunk_n):
        for row_idx in range(chunk_l):
            col = columns[col_idx][row_idx]
            current_height = 0
            last_layer_materials = col.limits[-1].materials
            last_layer_concentrations = col.limits[-1].concentrations
            
            while current_height < config['global_params']['terrain_depth'] and len(col.limits) < chunk_m:
                # 1. Material Selection
                possible_materials = []
                for mat in config['materials']:
                    if mat.id == 'Base': continue # Base only at bottom
                    # Check affinity
                    compatible = True
                    for i, last_mat_id in enumerate(last_layer_materials):
                        if config['affinities'].get((mat.id, last_mat_id), -1) == -1:
                            compatible = False
                            break
                    if compatible:
                        possible_materials.append(mat)
                
                if not possible_materials:
                    break # No valid materials

                # 2. Global Affinity
                affinities = []
                for mat in possible_materials:
                    ai = 0
                    for i, last_mat_id in enumerate(last_layer_materials):
                        a_ij = config['affinities'].get((mat.id, last_mat_id), 0)
                        c_j = last_layer_concentrations[i]
                        ai += a_ij * c_j
                    affinities.append(ai)
                
                sum_affinities = sum(affinities)
                if sum_affinities == 0:
                    break
                
                probabilities = [a / sum_affinities for a in affinities]
                
                # Select materials (simplified: pick top 1-2 based on prob)
                selected_indices = select_materials(probabilities, possible_materials)
                selected_materials = [possible_materials[i] for i in selected_indices]
                
                # 3. Concentrations
                concentrations = assign_concentrations(selected_materials)
                
                # 4. Properties
                properties = calculate_properties(selected_materials, concentrations, config)
                
                # 5. Height
                # Perlin noise n (simplified)
                n = perlin_noise(col_idx, row_idx, config['global_params']['scale'])
                similarity = properties['similarity']
                h_minus_1 = col.limits[-1].height if col.limits else 0
                
                h_modified = h_minus_1 * similarity + n * (1 - similarity)
                
                # Final height relative to previous
                avg_prev_height = sum(l.height for l in col.limits) / len(col.limits) if col.limits else 0
                final_height = h_modified + properties['thickness'] + avg_prev_height
                
                # 6. Flow
                flow = generate_flow(col_idx, row_idx, final_height, config['global_params']['scale'])
                
                # 7. Create Limit
                limit = Limit(final_height, selected_materials, concentrations, flow, properties)
                col.limits.append(limit)
                
                current_height = final_height

    # Generate Veins and Caves (Simplified: Apply to random columns)
    for col_idx in range(chunk_n):
        for row_idx in range(chunk_l):
            col = columns[col_idx][row_idx]
            if random.random() < 0.1: # Chance to generate vein/cave
                generate_vein_or_cave(col, col_idx, row_idx, config, is_vein=True)
            if random.random() < 0.05: # Chance to generate cave
                generate_vein_or_cave(col, col_idx, row_idx, config, is_vein=False)

    return columns

def assign_concentrations(materials):
    if len(materials) == 1:
        return [1.0]
    
    # Main material concentration [0.5, 0.9]
    main_conc = random.uniform(0.5, 0.9)
    remaining = 1.0 - main_conc
    
    concentrations = [main_conc]
    for _ in range(len(materials) - 2):
        conc = random.uniform(0, remaining)
        concentrations.append(conc)
        remaining -= conc
    
    # Last material gets the rest
    concentrations.append(remaining)
    
    # Normalize to ensure sum is exactly 1 (handle float errors)
    total = sum(concentrations)
    concentrations = [c / total for c in concentrations]
    return concentrations

def calculate_properties(materials, concentrations, config):
    # V = sum(v_i * c_i)
    # Simplified: Assume material has 'roughness', 'similarity', 'thickness' attributes
    # In real implementation, these are looked up from config/materials
    
    roughness = sum(m.roughness * c for m, c in zip(materials, concentrations))
    similarity = sum(m.similarity * c for m, c in zip(materials, concentrations))
    thickness = sum(m.thickness * c for m, c in zip(materials, concentrations))
    
    # Blend value is derived from thickness and roughness? 
    # Source says: "multiplies the thickness by the blend value... result is height where transition ends"
    # It implies blend_value is a property. Let's assume it's related to roughness or similarity.
    # Source doesn't explicitly define blend_value formula, just its usage.
    # We'll use similarity as a proxy for blending smoothness if not provided.
    blend_value = similarity 
    
    return {
        'roughness': roughness,
        'similarity': similarity,
        'thickness': thickness,
        'blend_value': blend_value
    }

def generate_flow(x, y, z, scale):
    # Horizontal 2D Perlin noise
    noise_x = perlin_noise(x, y, scale)
    noise_y = perlin_noise(x + 100, y, scale)
    
    # Rotate on Y axis
    # Flow tangent to surface. Simplified: Flow vector in XZ plane based on noise gradient
    flow = Vector3(noise_x, 0, noise_y).normalize()
    return flow

def generate_vein_or_cave(col, x, y, config, is_vein):
    area_param = config['global_params']['VeinArea'] if is_vein else config['global_params']['CaveArea']
    
    # 1. Bounding Square
    side_length = random.uniform(area_param / 2, area_param)
    
    # 2. Materials
    if is_vein:
        mats = [m for m in config['materials'] if 'Vein' in m.special_params]
    else:
        mats = [m for m in config['materials'] if 'Cave' in m.special_params]
    
    if not mats:
        return
        
    # Select main material
    main_mat = mats[0] # Simplified selection
    
    # 3. Properties
    # Same as layer generation but for single material or vein-specific logic
    # Source says: "same process as layer characteristics"
    # For vein/cave, it's often single material, so concentration is 1.0
    properties = {
        'roughness': main_mat.roughness,
        'similarity': 1.0, # Veins/Caves are distinct
        'thickness': main_mat.thickness,
        'relative_area': 1.0, # Simplified
        'relative_thickness': 1.0
    }
    
    # 4. Shape (Fractal)
    border_points = generate_fractal_shape(side_length, main_mat.roughness)
    
    # 5. Height
    # Calculate heights based on distance to border
    # This is complex to implement fully in pseudocode without grid context
    # We assume this generates a height map for the vein/cave region
    
    # 6. Flow
    flow = generate_flow(x, y, col.limits[-1].height, config['global_params']['scale'])
    
    # 7. Add Limit
    # The vein/cave creates a new limit at its top/bottom
    # Simplified: Add a limit at the calculated height
    limit = Limit(0, [main_mat], [1.0], flow, properties)
    col.limits.append(limit)

def generate_fractal_shape(side_length, roughness):
    # Midpoint displacement
    # d0 = roughness (from material)
    # SmoothConstant = roughness (from material)
    # Recursive depth 5
    
    d0 = roughness
    smooth_constant = roughness
    
    points = [Vector2(0,0), Vector2(side_length,0), Vector2(side_length,side_length), Vector2(0,side_length)]
    
    def displace(p1, p2, depth):
        if depth > 5:
            return
        
        mid = (p1 + p2) / 2
        max_disp = d0
        for _ in range(depth):
            max_disp = (max_disp / smooth_constant) / 2
            
        disp = random.uniform(-max_disp, max_disp)
        # Displace along diagonal/normal
        # Simplified: Just add random offset for pseudocode
        new_point = mid + Vector2(disp, disp)
        
        displace(p1, new_point, depth + 1)
        displace(new_point, p2, depth + 1)
        
    displace(points[0], points[1], 0)
    displace(points[1], points[2], 0)
    displace(points[2], points[3], 0)
    displace(points[3], points[0], 0)
    
    return points

def perlin_noise(x, y, scale):
    # Placeholder for actual Perlin noise implementation
    return random.uniform(-1, 1)

def select_materials(probabilities, materials):
    # Simplified: Pick the material with highest probability
    max_prob = max(probabilities)
    idx = probabilities.index(max_prob)
    return [idx]
```

## Operating characteristics

**Time Complexity:**
The generation time is dominated by the number of columns ($N \times L$) and the number of limits per column. The fractal generation for veins/caves is recursive with a fixed depth limit (5), making it $O(1)$ per vein/cave relative to recursion depth, but dependent on the area size. The overall complexity is linear with respect to the number of columns and average limits per column.

**Memory Complexity:**
The method significantly reduces memory usage compared to traditional voxel grids. The average number of limits generated is 84,376.42, compared to 125,000 voxels in a traditional volumetric terrain of the same dimensions [S1]. This represents a reduction of approximately 33% in data points, though each limit stores more data (materials, concentrations, flow) than a single voxel type ID.

**Measured Performance:**
Performance was measured using eight configurations with varying numbers of regular, cave, and vein materials. All materials had maximum affinity. The method scales well with the number of materials, as the affinity calculation is linear with respect to the number of materials in the last layer [S1].

**Strengths:**
*   Reduces memory requirements by storing limits instead of all voxels [S1].
*   Provides realistic appearance by mixing materials and flow between layers [S1].
*   High control for designers via configuration files [S1].
*   Supports dynamic voxel generation, allowing for infinite terrain extension [S1].

**Limitations:**
*   Cannot manage certain complex terrain types (e.g., subfigure 6(c) in source) [S1].
*   Fractal generation is limited to depth 5 for performance, which may restrict detail in veins/caves [S1].
*   Interpolation between limits may introduce artifacts if limits are too sparse [S1].

**When to Use:**
*   When memory efficiency is critical.
*   When realistic material mixing and flow are desired.
*   When dynamic, infinite terrain extension is needed.

**When Not to Use:**
*   When exact voxel-level control is required without interpolation.
*   When the terrain structure is too complex for the limit-based representation (e.g., highly fractured or non-layered structures) [S1].

## Background

**Origin:**
The method was proposed by Aitor Santamaría-Ibirika et al. in the paper "Procedural approach to volumetric terrain generation" [S1].

**History and Development:**
The paper presents a procedural approach to generate volumetric terrains with mixed materials and a natural, realistic appearance. It introduces a specific volume representation method that reduces memory requirements by using stacked material limits. The method includes metrics and requisites to evaluate terrain generators. The fractal algorithm for vein generation is based on the midpoint displacement method, extensively used for terrain generation [S1].

**Usage:**
The method is used for generating volumetric terrains in applications requiring realistic material distribution and flow, such as games or simulations. It allows for real-time extension of terrain layers [S1].

## Sources

[S1] Procedural approach to volumetric terrain generation, https://zenodo.org/records/18429902, fetched 2026-10-27
