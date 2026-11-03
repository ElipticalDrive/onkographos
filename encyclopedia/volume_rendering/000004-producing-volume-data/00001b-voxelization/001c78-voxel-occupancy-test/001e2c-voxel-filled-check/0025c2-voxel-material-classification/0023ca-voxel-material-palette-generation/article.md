# Voxel Material Palette Generation


## Overview
This method generates a compact lookup table (palette) that maps discrete voxel values (1–255) to physically-based rendering (PBR) material definitions for each of the six voxel faces [S1, unofficial]. It solves the problem of efficiently storing and accessing per-face material properties by packing material layer indices and flags into a single 2D texture atlas and a corresponding 3D palette array [S1, unofficial]. It differs from per-voxel material instances by batching materials into atlases, thereby reducing memory overhead and draw calls [S1, unofficial].

## How it works
**Inputs:**
- `materials`: A dictionary mapping string voxel values ("1" to "255") to a list of material names for the 6 faces [S1, unofficial].
- `material_definitions`: A source (e.g., JSON) providing PBR texture paths and boolean flags (`is_opaque`, `has_color`, etc.) for each material name [S1, unofficial].
- `atlas_width`, `atlas_height`: Dimensions of the output texture atlas in pixels [S1, unofficial].
- `res`: Resolution of each material tile in the atlas. Typical value: 512 [S1, unofficial].

**Outputs:**
- `palette`: A 3D NumPy array of shape `(256, 6, 3)` with type `np.uint8` [S1, unofficial].
- `color_atlas`, `emission_atlas`, `normal_atlas`, `rsma_atlas`: 2D texture arrays containing the packed material channels [S1, unofficial].

**Data Structures:**
1. **Palette Texture**: Shape `(256, 6, 3)` [S1, unofficial]. Each entry `[voxel, face]` contains:
   - Index 0: `layer & 255` (lower 8 bits of the material index) [S1, unofficial].
   - Index 1: `layer >> 8` (upper 8 bits of the material index) [S1, unofficial].
   - Index 2: `flag` (bitmask of material properties) [S1, unofficial].
2. **Material Flags Bitmask**:
   - Bit 0: `IS_OPAQUE`
   - Bit 1: `HAS_COLOR`
   - Bit 2: `HAS_EMISSION`
   - Bit 3: `HAS_NORMAL`
   - Bit 4: `HAS_RSMA` [S1, unofficial].
3. **Material Channels**:
   - `color`: RGBA texture [S1, unofficial].
   - `emission`: RGB texture [S1, unofficial].
   - `normal`: RGB texture (derived from RG source) [S1, unofficial].
   - `rsma`: RGBA texture (R=roughness, G=specular, B=metallic, A=ambient occlusion) [S1, unofficial].

**Algorithm Steps:**

1. **Load and Sort Materials**:
   - Parse material definitions to extract unique material names [S1, unofficial].
   - Sort unique names alphabetically to create a deterministic index `i` for each name [S1, unofficial].
   - Calculate atlas grid dimensions:
     $$ cols = \lfloor atlas\_width / res \rfloor $$
     $$ rows = \lfloor atlas\_height / res \rfloor $$ [S1, unofficial].

2. **Build Atlas and Lookup Tables**:
   - Initialize empty atlas textures for `color`, `emission`, `normal`, and `rsma` with dimensions `(rows * res, cols * res)` [S1, unofficial].
   - For each unique material name `name` at index `i`:
     - Calculate grid position:
       $$ col = i \pmod{cols} $$
       $$ row = \lfloor i / cols \rfloor $$ [S1, unofficial].
     - Calculate pixel coordinates:
       $$ x = col \times res $$
       $$ y = row \times res $$ [S1, unofficial].
     - Extract sub-region `[y:y+res, x:x+res]` from each atlas [S1, unofficial].
     - Load the material's PBR channels. If a channel is missing, fill it with `self.red_x` (default) [S1, unofficial].
     - Store the loaded textures in lists `self.color_textures`, etc. [S1, unofficial].
     - Record the layer index: `layers[name] = i` [S1, unofficial].
     - Compute the flag bitmask:
       $$ flag = (IS\_OPAQUE \text{ if } is\_opaque \text{ else } 0) \ | \ (HAS\_COLOR \text{ if } has\_color \text{ else } 0) \ | \ (HAS\_EMISSION \text{ if } has\_emission \text{ else } 0) \ | \ (HAS\_NORMAL \text{ if } has\_normal \text{ else } 0) \ | \ (HAS\_RSMA \text{ if } has\_rsma \text{ else } 0) $$ [S1, unofficial].
     - Record flags: `flags[name] = flag` [S1, unofficial].

3. **Fill Palette**:
   - Initialize `palette` array of shape `(256, 6, 3)` with zeros [S1, unofficial].
   - For each voxel value `v` (1 to 255):
     - Get the list of material names for the 6 faces [S1, unofficial].
     - If the list length is 1, replicate it to 6 faces: `names *= 6` [S1, unofficial].
     - If the list length is not 6, raise an error [S1, unofficial].
     - For each face `f` (0 to 5):
       - Get `name = names[f]` [S1, unofficial].
       - Lookup `layer = layers[name]` and `flag = flags[name]` [S1, unofficial].
       - Set `palette[v, f] = [layer & 255, layer >> 8, flag]` [S1, unofficial].

4. **Verify Palette**:
   - Check for mixed opaque/transparent sides on a single voxel [S1, unofficial].
   - Count opaque flags per voxel: `opaque_counts = sum(flag & 1 for face in palette[v])` [S1, unofficial].
   - If `0 < opaque_counts < 6` for any voxel, raise an error. Mixed opacity on a single voxel is invalid [S1, unofficial].

5. **Write Outputs**:
   - Save `palette` as a PNG texture [S1, unofficial].
   - Save `color_atlas`, `emission_atlas`, `normal_atlas`, and `rsma_atlas` as PNG textures [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_voxel_palette(materials, material_defs, atlas_width, atlas_height, res=512):
    # 1. Load and Sort Materials
    all_names = set()
    for face_names in materials.values():
        all_names.update(face_names)
    names = sorted(list(all_names))
    
    cols = atlas_width // res
    rows = atlas_height // res
    
    # Initialize atlases
    color_atlas = np.zeros((rows * res, cols * res, 4), dtype=np.uint8)
    emission_atlas = np.zeros((rows * res, cols * res, 3), dtype=np.uint8)
    normal_atlas = np.zeros((rows * res, cols * res, 3), dtype=np.uint8)
    rsma_atlas = np.zeros((rows * res, cols * res, 4), dtype=np.uint8)
    
    layers = {}
    flags = {}
    
    # 2. Build Atlas and Lookup Tables
    for i, name in enumerate(names):
        col = i % cols
        row = i // cols
        x = col * res
        y = row * res
        
        # Load material properties
        mat = material_defs[name]
        is_opaque = mat.get('is_opaque', True)
        has_color = mat.get('has_color', True)
        has_emission = mat.get('has_emission', False)
        has_normal = mat.get('has_normal', False)
        has_rsma = mat.get('has_rsma', False)
        
        # Load textures: if channel missing, fill with self.red_x (default)
        color = mat.get('color', np.full((res, res, 4), 255, dtype=np.uint8))
        emission = mat.get('emission', np.full((res, res, 3), 0, dtype=np.uint8))
        normal = mat.get('normal', np.full((res, res, 3), 128, dtype=np.uint8))
        rsma = mat.get('rsma', np.full((res, res, 4), 0, dtype=np.uint8))
        
        # Store in atlas
        color_atlas[y:y+res, x:x+res] = color
        emission_atlas[y:y+res, x:x+res] = emission
        normal_atlas[y:y+res, x:x+res] = normal
        rsma_atlas[y:y+res, x:x+res] = rsma
        
        # Compute flags
        flag = 0
        if is_opaque: flag |= 1 # IS_OPAQUE
        if has_color: flag |= 2 # HAS_COLOR
        if has_emission: flag |= 4 # HAS_EMISSION
        if has_normal: flag |= 8 # HAS_NORMAL
        if has_rsma: flag |= 16 # HAS_RSMA
        
        layers[name] = i
        flags[name] = flag
        
    # 3. Fill Palette
    palette = np.zeros((256, 6, 3), dtype=np.uint8)
    
    for voxel_str, face_names in materials.items():
        voxel = int(voxel_str)
        if voxel < 1 or voxel > 255:
            raise ValueError(f"Invalid voxel value: {voxel}")
            
        if len(face_names) == 1:
            face_names = face_names * 6
        if len(face_names) != 6:
            raise ValueError(f"Invalid material names at voxel {voxel}: {face_names}")
            
        for f, name in enumerate(face_names):
            layer = layers[name]
            flag = flags[name]
            palette[voxel, f] = [layer & 255, layer >> 8, flag]
            
    # 4. Verify Palette
    opaque_counts = np.sum(palette[:, :, 2] & 1, axis=1, dtype=np.uint8)
    mixed = np.logical_and(opaque_counts > 0, opaque_counts < 6)
    if np.any(mixed):
        raise ValueError("ERROR: Palette contains voxels with mixed opaque/transparent sides.")
        
    return palette, color_atlas, emission_atlas, normal_atlas, rsma_atlas
```

**Edge Cases and Pitfalls:**
- **Mixed Opacity**: A voxel cannot have both opaque and transparent faces. The system validates this and exits with an error if detected [S1, unofficial].
- **Missing Channels**: If a material lacks a specific channel, it must be filled with `self.red_x` (default) to prevent atlas corruption [S1, unofficial].
- **Voxel Value Range**: Only voxel values 1–255 are supported. The system explicitly validates this range and raises an error for values outside it [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot R^2 + V \cdot F)$, where $N$ is the number of unique materials, $R$ is the resolution of each tile, $V$ is the number of voxel types (255), and $F$ is the number of faces (6) [S1, unofficial]. The dominant cost is loading and packing textures for $N$ materials [S1, unofficial].
- **Memory Complexity**: $O(N \cdot R^2 + V \cdot F)$ [S1, unofficial]. The atlas size is $N \cdot R^2$ pixels per channel [S1, unofficial]. The palette size is fixed at $256 \times 6 \times 3$ bytes [S1, unofficial].
- **Performance**: Reduces memory overhead and draw calls by batching materials into atlases instead of using per-voxel material instances [S1, unofficial].
- **Strengths**: Efficient storage for voxel worlds with many distinct materials; fast lookup during rendering [S1, unofficial].
- **Limitations**: Fixed voxel value range (1–255); mixed opacity per voxel is not supported; atlas size grows linearly with the number of unique materials [S1, unofficial].

## Background
This method is part of the voxel material classification and storage system in the Godot Voxel plugin [S1, unofficial]. It enables efficient PBR rendering for voxel-based games by mapping discrete voxel IDs to a shared set of material atlases [S1, unofficial].

## Sources
[S1] viktor-ferenczi/godot-voxel, https://github.com/viktor-ferenczi/godot-voxel, fetched 2026-10-24 [S1, unofficial]
