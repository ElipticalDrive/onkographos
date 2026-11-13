# Material ID Assignment & Caching

Also called: Material ID Disambiguation, Terrain Material ID Assignment & Caching, TerrainMatManager

## Overview
This method maps block-level material identifiers to unique, globally distinct terrain material IDs to facilitate mesh merging and GPU resource management [S1, unofficial]. It creates material objects on demand and caches the results to avoid redundant GPU allocations, ensuring that visually identical materials share a single GPU resource [S1, unofficial]. It differs from sibling methods by focusing on the logical disambiguation and caching of material references rather than the geometric assembly or sectioning of the mesh itself [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A block material ID (integer), and optionally a custom render material object or a texture URL string.
*   **Output:** A unique terrain material ID (integer) and the corresponding material object reference.

**Data Structures**
The method maintains the following state maps and counters:
1.  `_idCounter`: An integer starting at 1000, used to generate unique terrain IDs [S1, unofficial].
2.  `_blockMatIDtoTerrainID`: A map from block material IDs to terrain material IDs [S1, unofficial].
3.  `_terrainIDtoMatObject`: A map from terrain material IDs to material objects [S1, unofficial].
4.  `_texURLtoTerrainID`: A map from texture URLs to terrain material IDs [S1, unofficial].
5.  `_renderMatToTerrainID`: A map from material objects to terrain material IDs [S1, unofficial].

**Algorithm Steps**
1.  **Lookup Cache:** Check `_blockMatIDtoTerrainID` for the input block material ID. If found, return the cached terrain ID and material object.
2.  **Assign Terrain ID:** If not found, call `decideTerrainMatID` to determine the unique terrain ID based on the material's properties [S1, unofficial].
3.  **Create Material:** Check `_terrainIDtoMatObject` for the assigned terrain ID. If not found, call `createTerrainMat` to generate the material object, add it to the global `allMaterials` list, and cache the mapping [S1, unofficial].
4.  **Return:** Return the terrain ID and material object.

**Decision Logic (`decideTerrainMatID`)**
The terrain ID is determined by the following priority rules [S1, unofficial]:
1.  **Custom Render Material:** If a custom render material object is provided, assign a unique ID per material object.
2.  **Texture URL:** If a texture URL is provided, assign a unique ID per URL.
3.  **Alpha Transparency:** If the material has an alpha value strictly between 0 and 1, discretize it into 100 buckets. The ID is calculated as:
    $$ \text{terrainID} = 10 + \lfloor \alpha \times 100 \rfloor $$
    This maps alpha values to the range [10, 109] [S1, unofficial].
4.  **Default:** Otherwise, return ID `1` [S1, unofficial].

**Material Creation (`createTerrainMat`)**
1.  If a custom render material exists, return it [S1, unofficial].
2.  If no texture is specified:
    *   If alpha is required (non-zero/non-one), create a flat material with alpha support [S1, unofficial].
    *   Otherwise, return a default material [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
_idCounter = 1000
_blockMatIDtoTerrainID = {}
_terrainIDtoMatObject = {}
_texURLtoTerrainID = {}
_renderMatToTerrainID = {}
allMaterials = []

def get_terrain_material(block_mat_id, custom_mat=None, texture_url=None, alpha=1.0):
    # Step 1: Lookup cache
    if block_mat_id in _blockMatIDtoTerrainID:
        terrain_id = _blockMatIDtoTerrainID[block_mat_id]
        return terrain_id, _terrainIDtoMatObject[terrain_id]

    # Step 2: Assign Terrain ID
    terrain_id = decide_terrain_mat_id(custom_mat, texture_url, alpha)

    # Step 3: Create Material if not cached
    if terrain_id not in _terrainIDtoMatObject:
        mat_obj = create_terrain_mat(custom_mat, texture_url, alpha)
        _terrainIDtoMatObject[terrain_id] = mat_obj
        allMaterials.append(mat_obj)
        _renderMatToTerrainID[mat_obj] = terrain_id
        if texture_url:
            _texURLtoTerrainID[texture_url] = terrain_id

    # Cache block mapping
    _blockMatIDtoTerrainID[block_mat_id] = terrain_id

    return terrain_id, _terrainIDtoMatObject[terrain_id]

def decide_terrain_mat_id(custom_mat, texture_url, alpha):
    # Priority 1: Custom Render Material
    if custom_mat is not None:
        if custom_mat in _renderMatToTerrainID:
            return _renderMatToTerrainID[custom_mat]
        # Assign unique ID per material object
        # Note: Logic for generating unique ID per object instance not fully specified in source
        # [G] Use a hash or counter for object identity if not present
        terrain_id = _idCounter
        _idCounter += 1
        return terrain_id

    # Priority 2: Texture URL
    if texture_url is not None:
        if texture_url in _texURLtoTerrainID:
            return _texURLtoTerrainID[texture_url]
        terrain_id = _idCounter
        _idCounter += 1
        return terrain_id

    # Priority 3: Alpha Discretization
    if 0 < alpha < 1:
        # Discretize into 100 buckets: 10 to 109
        bucket = int(alpha * 100)
        terrain_id = 10 + bucket
        return terrain_id

    # Priority 4: Default
    return 1

def create_terrain_mat(custom_mat, texture_url, alpha):
    if custom_mat is not None:
        return custom_mat
    
    # Create new material
    # [G] Assuming a generic material creation function
    mat_obj = create_new_material()
    
    if texture_url is not None:
        mat_obj.texture = texture_url
    
    if alpha != 1.0:
        mat_obj.alpha = alpha
        mat_obj.has_alpha = True
    else:
        mat_obj.has_alpha = False
        
    return mat_obj
```

**Edge Cases and Pitfalls**
*   **Alpha Precision:** Alpha values are discretized into 100 buckets (IDs 10–109) to group visually similar transparency levels. This avoids creating unique IDs for negligible alpha differences [S1, unofficial].
*   **Identity of Custom Materials:** The source implies unique IDs per material object for custom renders, but does not specify how object identity is determined (e.g., memory address, hash) [S1, unofficial].

## Subtopics

### Material-Based Mesh Sectioning
Splits the decoded mesh into separate sections based on material IDs. Choose this when you need to separate geometry by material for efficient rendering or culling. [Material-Based Mesh Sectioning](8778-material-based-mesh-sectioning.md)

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ on average using hash maps. Material creation is $O(1)$ but involves GPU allocation.
*   **Memory Complexity:** Proportional to the number of unique materials in the scene.
*   **Strengths:** Reduces GPU state changes by merging identical materials; avoids redundant allocations.
*   **Limitations:** Relies on a global cache which may grow large in scenes with many unique materials.
*   **When to Use:** When rendering large terrains or blocks with repeated material patterns.
*   **When Not to Use:** When materials are unique per triangle or change frequently per frame.

## Background
This method originates from the NOA (Noa) engine, an open-source voxel game engine [S1, unofficial]. It is used in terrain rendering systems where block-based meshes are assembled into larger terrain chunks, requiring efficient material management to maintain performance [S1, unofficial].

## Sources
[S1] fenomas/noa, https://github.com/fenomas/noa, fetched 2026-10-26 (unofficial)
