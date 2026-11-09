# Heterogeneous Volumes (HV) / Sparse Volume Textures (SVT) approach
Also called: Chunked SVT rendering, HV/SVT approach, Heterogeneous Volume, Heterogeneous Volume Actor (HVA)

## Overview
The Heterogeneous Volumes (HV) approach renders large volumetric datasets by chunking them into multiple Sparse Volume Textures (SVT), each fitting within engine and API limits, and treating the collection as a single logical volume [S2, unofficial]. It solves the problem of DirectX 11's contiguous memory fragmentation limiting single texture allocation to approximately 1 gigavoxel, while leveraging DirectX 12's reserved resources to support larger theoretical allocations up to $2048^3 = 8$ gigavoxels [S2, unofficial]. Unlike a single contiguous texture which avoids lighting discontinuities but exceeds allocation limits, HV enables the display of entire massive datasets at reasonable frame rates by accepting chunked rendering [S2, unofficial]. It supports full lighting, emission, and color, making it suitable for scientific data visualization, unlike debug viewers or fog-specific renderers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A large sparse volumetric dataset (e.g., density, color, opacity fields) [S2, unofficial].
- Input: Graphics API context (DirectX 11 or 12) [S2, unofficial].
- Output: Rendered image of the composite volume [S1, unofficial].

**Data Structures:**
The method relies on the Sparse Volume Texture (SVT) structure defined in [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md). The HV approach organizes the dataset into a grid of SVT chunks [S2, unofficial].
- **Chunk Size:** Each chunk must be small enough to fit within the volume texture limits [S2, unofficial]. In DirectX 11, this is constrained by memory fragmentation to approximately 1 gigavoxel [S2, unofficial]. In DirectX 12, using reserved resources, the backing memory can be allocated as a collection of tiles, allowing a virtual address space mapped to these tiles, theoretically supporting up to $2048^3 = 8$ gigavoxels [S2, unofficial].
- **Compression:** Tile data is compressed [S2, unofficial]. The total compressed tile data (including MIP data and padding for correct interpolation) must not exceed 4 gigavoxels for successful rendering of the largest tested datasets [S2, unofficial].

**Numbered Steps:**
1. **Chunking:** Divide the large volumetric dataset into chunks [S2, unofficial]. Each chunk is assigned to a separate volume texture resource [S2, unofficial]. The dimensions of each chunk are chosen to ensure they fit within the maximum allocatable size for the target API (e.g., < 1 gigavoxel for DX11, up to 8 gigavoxels for DX12 reserved resources) [S2, unofficial].
2. **Texture Allocation:** For DirectX 11, allocate contiguous memory for each chunk, as VRAM fragmentation may cause allocation failures for large chunks [S2, unofficial]. For DirectX 12, use reserved resources to allocate backing memory as a collection of tiles and map the virtual address space of the large logical texture to these allocated tiles [S2, unofficial].
3. **Data Population:** Populate each SVT chunk with the corresponding subset of the dataset [S2, unofficial]. Ensure compressed tile data (including MIP levels and padding) is correctly formatted [S2, unofficial].
4. **Engine Integration:** Integrate the SVT chunks into the rendering engine (e.g., Unreal Engine 5) [S2, unofficial]. This may require modifying engine source code to handle large indices [S2, unofficial].
5. **Rendering:** Render the volume by sampling from the SVT chunks [S1, unofficial]. The engine treats the collection of chunks as a Heterogeneous Volume [S2, unofficial].

**Parameters:**
- **Maximum Volume Texture Size (DX11):** ~1 gigavoxel (limited by fragmentation) [S2, unofficial].
- **Maximum Volume Texture Size (DX12):** $2048^3 = 8$ gigavoxels (theoretical/API limit) [S2, unofficial].
- **Maximum Compressed Tile Data Size:** 4 gigavoxels (including MIP data and padding) [S2, unofficial].
- **Maximum Successfully Rendered Sparse Dataset Resolution:** $32k \times 32k \times 16k$ voxels [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Lighting Artifacts:** Incorrect lighting occurs at chunk borders because each chunk is lit separately without neighbor information [S2, unofficial]. This makes borders incorrectly bright [S2, unofficial].
- **Integer Overflow:** Signed `int32` overflows in the Engine's SVT-related source code cause issues [S2, unofficial]. These must be handled by converting them to unsigned `uint32` or `int64` [S2, unofficial].
- **VRAM Limitations:** Current implementation is limited by VRAM [S2, unofficial]. Out-of-core rendering is a future goal to overcome this [S2, unofficial].
- **64-bit Arithmetic:** Performance issues are associated with 64-bit arithmetic in GPU shaders, which is a target for future optimization [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def determine_chunk_size(resolution, max_voxels):
    """Calculate grid dimensions for a chunk that fits within max_voxels [G]."""
    dim = int(max_voxels ** (1/3))
    return (dim, dim, dim)

def chunk_dataset(dataset, chunk_size):
    """Yield subsets of the dataset aligned to chunk_size [G]."""
    for x in range(0, dataset.width, chunk_size[0]):
        for y in range(0, dataset.height, chunk_size[1]):
            for z in range(0, dataset.depth, chunk_size[2]):
                yield dataset[x:x+chunk_size[0], y:y+chunk_size[1], z:z+chunk_size[2]]

def create_svt(chunk_data):
    """Allocate a Sparse Volume Texture resource for the chunk data [G]."""
    return {"data": chunk_data, "format": "compressed_tile"}

def get_compressed_tile_size(svt):
    """Return the size of the compressed tile data including MIP levels and padding [G]."""
    return len(svt["data"])

def fix_engine_int32_overflows(chunks):
    """Convert signed int32 indices to unsigned uint32 or int64 to prevent overflows [S2, unofficial]."""
    for chunk in chunks:
        chunk.indices = [int(idx) & 0xFFFFFFFF if idx < 0 else idx for idx in chunk.indices]

def render_svt_chunk(chunk, camera, lights):
    """Sample and render a single SVT chunk with per-chunk lighting [G]."""
    pass

def composite_rendered_chunks():
    """Combine rendered chunks into the final image [G]."""
    pass

def setup_heterogeneous_volume(dataset, api_version="DX12"):
    chunks = []
    max_chunk_voxels = 1e9 if api_version == "DX11" else (2048**3 if api_version == "DX12" else 0)
    if max_chunk_voxels == 0: raise ValueError("Unsupported API")
    chunk_size = determine_chunk_size(dataset.resolution, max_chunk_voxels)
    for chunk_data in chunk_dataset(dataset, chunk_size):
        svt = create_svt(chunk_data)
        if get_compressed_tile_size(svt) > 4e9:
            raise ValueError("Compressed tile data exceeds 4 gigavoxels")
        chunks.append(svt)
    fix_engine_int32_overflows(chunks)
    return chunks

def render_heterogeneous_volume(chunks, camera, lights):
    for chunk in chunks:
        render_svt_chunk(chunk, camera, lights)
    composite_rendered_chunks()
```

## Operating characteristics
- **Time Complexity:** Not given in the sources. [S2, unofficial]
- **Memory Complexity:** Depends on the number of chunks and the compressed tile data size [S2, unofficial]. The total compressed tile data (including MIP data and padding) must not exceed 4 gigavoxels for the largest tested datasets [S2, unofficial]. DirectX 12 allows virtual address space mapping to tiled backing memory, mitigating fragmentation issues present in DirectX 11 [S2, unofficial].
- **Measured Performance:** Achieved 20 fps on an NVIDIA RTX 3500 Ada Generation Laptop GPU for the Kolumbo dataset (4211 × 1501 × 935 ≈ 6 billion total voxels, ~2.6 billion non-empty) [S2, unofficial].
- **Scaling:** Scaling is limited by VRAM [S2, unofficial]. Out-of-core rendering is planned to overcome VRAM limitations and graphics API constraints [S2, unofficial]. Performance issues with 64-bit arithmetic in GPU shaders may impact scaling [S2, unofficial].
- **Strengths:** Enables rendering of very large sparse datasets (up to $32k \times 32k \times 16k$ voxels) that would otherwise exceed engine or API limits [S2, unofficial]. DirectX 12 reserved resources allow larger theoretical allocations despite VRAM fragmentation [S2, unofficial]. Supports full lighting, emission, and color for scientific data visualization [S1, unofficial].
- **Limitations:** Introduces lighting artifacts at chunk borders due to lack of neighbor information for lighting calculations [S2, unofficial]. Limited by VRAM in current implementation [S2, unofficial]. Requires engine modifications to fix integer overflows [S2, unofficial].
- **When to use:** When rendering large sparse volumetric datasets in Unreal Engine 5 that exceed single texture allocation limits, and when lighting artifacts at chunk borders are acceptable or can be mitigated [S2, unofficial].
- **When not to use:** When high-fidelity lighting continuity across the entire volume is required, or when VRAM is severely constrained and out-of-core rendering is not available [S2, unofficial].

## Background
The HV/SVT approach was implemented in a custom build of Unreal Engine 5.4 for the ARENA2 visualization dome at the GEOMAR Helmholtz Centre for Ocean Research Kiel [S2, unofficial]. It was developed to address the challenges of rendering large volumetric datasets in game engines, specifically overcoming DirectX 11's contiguous memory allocation limits and leveraging DirectX 12's reserved resources [S2, unofficial]. The approach was identified as the most promising among several SVT-based methods for rendering large datasets, unlike debug viewers or fog-specific renderers [S1, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
[S2] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
