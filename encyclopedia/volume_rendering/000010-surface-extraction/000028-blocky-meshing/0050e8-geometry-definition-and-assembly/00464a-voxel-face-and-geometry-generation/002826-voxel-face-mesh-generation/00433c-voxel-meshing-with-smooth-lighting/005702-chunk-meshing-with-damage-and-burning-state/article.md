# Chunk Meshing with Damage and Burning State

Also called: Mesher

## Overview
A technique for generating mesh geometry for voxel chunks in WebAssembly, incorporating damage and burning state to modify vertex attributes, using compact vertex formats [S1, unofficial]. It extends the base voxel meshing by dynamically adjusting visual properties based on block health and fire status, distinct from sibling methods that may use optimized indexed geometry without state-aware attribute modification [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Chunk coordinates ($cx, cy, cz$), sequence number ($seq$), chunk key, and a block table containing per-block state and color data.
- **Output:** A `MeshInput` structure containing solid geometry data (positions, normals, colors, emission, cut masks, tile indices, indices) and foliage geometry data (positions, UVs, colors, indices).

**Data Structures:**
1.  **Block Table Layout:** A `Float32Array` where each block occupies `BLOCK_STRIDE` (20) floats [S1, unofficial].
    -   Indices 0–2: Top color ($r, g, b$).
    -   Indices 3–5: Side color ($r, g, b$).
    -   Index 6: Side-top color ($r, g, b$) or $-1$.
    -   Indices 7–9: Bottom color ($r, g, b$).
    -   Index 10: Emission intensity.
    -   Index 11: Anchored flag ($0$ or $1$).
    -   Indices 12–14: Tile indices ($0, 1, 2$).
2.  **MeshData Struct:** Contains separate arrays for solid and foliage geometry [S1, unofficial]:
    -   Solid: `pos` (Float32Array), `nor` (Int8Array), `col` (Uint8Array), `emit` (Uint8Array), `cut` (Uint8Array), `tile` (Uint8Array), `ind` (Uint16Array).
    -   Foliage: `fpos` (Float32Array), `fuv` (Float32Array), `fcol` (Uint8Array), `find` (Uint16Array).

**Parameters:**
-   `CHUNK`: Chunk size, $32$ [S1, unofficial].
-   `BLOCK_STRIDE`: Stride for block table, $20$ [S1, unofficial].

**Steps:**
1.  **Mesh Job Creation:**
    -   Calculate local coordinates for damage and burning blocks.
    -   For damage: Compute health points ($hp$). Update state byte at index $li$:
        $$state[li] = (state[li] \ \& \ 0x80) \ | \ \text{round}\left(\min\left(1, \frac{dmg}{hp}\right) \times 127\right)$$
    -   For burning: Set bit 7 of the state byte:
        $$state[li] \ |= \ 0x80$$
    -   Return `MeshInput` with key, seq, cx, cy, cz, and state [S1, unofficial].

2.  **Solid Geometry Creation:**
    -   If `d.ind.length` is $0$, return `null`.
    -   Create `BufferGeometry` with attributes:
        -   `position`: `d.pos` (3 components).
        -   `normal`: `d.nor` (3 components, true).
        -   `color`: `d.col` (3 components, true).
        -   `emit`: `d.emit` (1 component, true).
        -   `cut`: `d.cut` (1 component).
        -   `tile`: `d.tile` (1 component).
    -   Set index to `d.ind`.
    -   Set bounding sphere to center $(CHUNK/2, CHUNK/2, CHUNK/2)$ with radius $CHUNK \times 0.87$ [S1, unofficial].

3.  **Foliage Geometry Creation:**
    -   If `d.find.length` is $0$, return `null`.
    -   Create normal array (`Int8Array`) with UP vector $(0, 127, 0)$ for each vertex.
    -   Create `BufferGeometry` with attributes:
        -   `position`: `d.fpos` (3 components).
        -   `normal`: `nor` (3 components, true).
        -   `uv`: `d.fuv` (2 components).
        -   `color`: `d.fcol` (3 components).
    -   Set index to `d.find`.
    -   Set bounding sphere to center $(CHUNK/2, CHUNK/2, CHUNK/2)$ with radius $CHUNK \times 0.87$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_mesh_job(cx, cy, cz, seq, key, blocks):
    """
    Calculate local coordinates for damage and burning blocks.
    blocks: list of block data, each with 'li' (local index), 'hp', 'dmg', 'burning'
    """
    state = [0] * len(blocks)
    
    for block in blocks:
        li = block['li']
        hp = block['hp']
        dmg = block['dmg']
        
        # Damage calculation
        # state[li] = (state[li] & 0x80) | round(min(1, dmg/hp) * 127)
        damage_ratio = min(1.0, dmg / hp) if hp > 0 else 0.0
        damage_val = round(damage_ratio * 127)
        state[li] = (state[li] & 0x80) | damage_val
        
        # Burning calculation
        if block['burning']:
            state[li] |= 0x80
            
    return {
        'key': key,
        'seq': seq,
        'cx': cx,
        'cy': cy,
        'cz': cz,
        'state': state
    }

def create_solid_geometry(d):
    """
    d: MeshData struct containing solid geometry arrays
    """
    if len(d['ind']) == 0:
        return None
        
    # Create BufferGeometry
    geometry = {
        'attributes': {
            'position': {'data': d['pos'], 'components': 3},
            'normal': {'data': d['nor'], 'components': 3, 'normalized': True},
            'color': {'data': d['col'], 'components': 3, 'normalized': True},
            'emit': {'data': d['emit'], 'components': 1, 'normalized': True},
            'cut': {'data': d['cut'], 'components': 1},
            'tile': {'data': d['tile'], 'components': 1}
        },
        'index': d['ind']
    }
    
    # Set bounding sphere
    chunk_size = 32
    geometry['bounding_sphere'] = {
        'center': (chunk_size / 2, chunk_size / 2, chunk_size / 2),
        'radius': chunk_size * 0.87
    }
    
    return geometry

def create_foliage_geometry(d):
    """
    d: MeshData struct containing foliage geometry arrays
    """
    if len(d['find']) == 0:
        return None
        
    # Create normal array with UP vector (0, 127, 0)
    num_vertices = len(d['fpos']) // 3
    nor = [0] * (num_vertices * 3)
    for i in range(num_vertices):
        nor[i * 3] = 0
        nor[i * 3 + 1] = 127
        nor[i * 3 + 2] = 0
        
    # Create BufferGeometry
    geometry = {
        'attributes': {
            'position': {'data': d['fpos'], 'components': 3},
            'normal': {'data': nor, 'components': 3, 'normalized': True},
            'uv': {'data': d['fuv'], 'components': 2},
            'color': {'data': d['fcol'], 'components': 3}
        },
        'index': d['find']
    }
    
    # Set bounding sphere
    chunk_size = 32
    geometry['bounding_sphere'] = {
        'center': (chunk_size / 2, chunk_size / 2, chunk_size / 2),
        'radius': chunk_size * 0.87
    }
    
    return geometry
```

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Performance:** Not given in the sources.
-   **Strengths:** Uses compact vertex formats (~15 bytes/vertex for solid geometry) to reduce memory bandwidth and storage [S1, unofficial].
-   **Limitations:** Not given in the sources.
-   **When to use:** When dynamic visual states (damage, burning) are required for voxel chunks in a WebAssembly environment [S1, unofficial].
-   **When not to use:** When static geometry without state-based attribute modification is sufficient [S1, unofficial].

## Background
Originated in a WebAssembly custom engine for rendering destructible voxels in the browser [S1, unofficial]. The method is part of a larger system that includes entity management and physics [S1, unofficial].

## Sources
[S1] 12.8B destructible voxels rendered in the browser with Rust WASM custom engine, https://github.com/cktang88/2.5d-experiment, fetched 2026-10-26 (unofficial)
