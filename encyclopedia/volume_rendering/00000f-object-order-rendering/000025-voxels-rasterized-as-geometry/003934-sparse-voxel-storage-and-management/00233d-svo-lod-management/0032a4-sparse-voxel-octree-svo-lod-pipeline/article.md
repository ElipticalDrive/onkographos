# Sparse Voxel Octree (SVO) LOD Pipeline

Also called: SVO geometry cache, unified level-of-detail pipeline

## Overview
This method is a unified production pipeline that converts complex, animated assets (characters, props, vegetation) into a consistent Sparse Voxel Octree (SVO) representation to streamline lighting and rendering workflows [S1]. It addresses the challenge of efficiently populating, animating, and ray-tracing dynamic scenes with thousands of intricate assets by replacing direct geometry processing with a pre-voxelized, LOD-managed cache [S1]. Unlike standard real-time SVO ray-tracing which often requires dynamic voxelization per frame, this pipeline separates asset preparation from rendering, using stored surface data and procedural blend shapes to animate static SVOs, thereby reducing memory footprint and voxelization time [S1].

## How it works
The pipeline operates in two distinct phases: Asset Preparation (offline) and Render-Time Execution (online) [S1].

### Inputs and Outputs
**Inputs:**
*   Source geometry assets with nested hierarchies, rig joints, and procedural components (fur, materials) [S1].
*   Artist-defined LOD choices or screen-size thresholds [S1].
*   User-defined asset groupings for parallel processing [S1].

**Outputs:**
*   A unified render object containing the SVO data, ready for the modular generator [S1].
*   LOD representations including: light proxy, simplified binary geometry, partial voxel (fine detail), and fully voxelized SVO [S1].

### Data Structures
The core data structure is the Sparse Voxel Octree (SVO) [S1].
*   **Node Content:** Each voxel node stores surface data including positions, normals, UVs, and local offsets [S1].
*   **Material Channels:** Material procedures are baked into channels for color, specular, roughness, and other PBR parameters [S1].
*   **Control Signals:** Channels store control signals and alternate material descriptions to support asset variations [S1].
*   **Local Vectors:** To resolve world-space dependency issues (e.g., side-specific coloring), local-space vectors are stored within the SVO to be transposed by the generator at render time [S1].

### Steps

#### Phase 1: Asset Preparation
1.  **Generate LODs:** For each asset, generate consistent LOD representations: light proxy, simplified binary, partial voxel, and full SVO [S1].
2.  **Construct SVO:** Sample the geometry into the stored octree. Bake material procedures into the defined channels [S1].
3.  **Optimize for Animation:**
    *   Regenerate SVOs at high resolution with optimal posing [S1].
    *   Splay overlapping elements (e.g., feathers) apart during static generation to prevent clipping artifacts when animated [S1].
    *   Store local-space vectors for features requiring world-space orientation (e.g., wing sides) [S1].
4.  **Consolidate Deformations:** Replace complex rig emulations with a set of blend shapes. These are procedurally selected and triggered to avoid emulating thousands of joints [S1].
5.  **Group Assets:** Combine dense asset groups into fewer, larger voxel spaces by creating a single generator definition containing multiple SVO instances. This minimizes tracking between bodies and improves ray-tracing efficiency [S1].

#### Phase 2: Render-Time Execution
1.  **Load Asset Description:** The modular generator loads the object description, which includes the SVO and blend shape data [S1].
2.  **Select LOD:** Choose the appropriate LOD level based on screen size or artist choice [S1].
3.  **Animate via Blend Shapes:** Procedurally select and trigger blend shapes to deform the static SVO. This avoids evaluating the original complex rig [S1].
4.  **Voxelize:** Sample the deformed SVO at the chosen octree level into a frustum-aligned voxel grid [S1].
5.  **Purge Geometry:** Once the posed SVO is sampled into the voxel grid, the original geometry and rig data are purged from memory [S1].
6.  **Render:** Pass the voxel grid to the renderer. If assets are grouped, render the combined voxel space [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prepare_asset(asset):
    # Generate LOD representations: light proxy, simplified binary, partial voxel, full SVO [S1]
    lod_proxies = []  # [G] Standard LOD generation
    lod_binary = []   # [G] Standard binary export
    lod_partial = []  # [G] Standard partial voxelization
    # Construct SVO: sample geometry into stored octree, containing positions, normals, UVs, local offsets [S1]
    svo = {"nodes": [], "channels": {"color": [], "specular": [], "roughness": [], "control_signals": [], "alternate_materials": []}}
    # Store local-space vectors for world-space dependencies (e.g., wing sides) [S1]
    svo["local_vectors"] = []  # [G] Compute local-space vectors from asset geometry
    # Optimize for animation: regenerate SVOs with high resolution and optimal posing, splaying overlapping elements [S1]
    # [G] Regenerate with high res and optimal posing
    # Consolidate deformations into blend shapes procedurally selected and triggered [S1]
    blend_shapes = []  # [G] Generate blend shapes from rig to avoid emulating thousands of joints
    return {"lod_proxies": lod_proxies, "lod_binary": lod_binary, "lod_partial": lod_partial, "svo": svo, "blend_shapes": blend_shapes}

def render_asset(asset_data, camera, lod_level, group_id=None):
    svo = asset_data["svo"]
    blend_shapes = asset_data["blend_shapes"]
    # Select LOD based on screen size or artist choice [S1]
    # [G] Traverse octree to chosen depth
    current_svo = svo
    # Animate using blend shapes to deform the static SVO [S1]
    # [G] Apply blend shape weights to SVO samples
    posed_svo = current_svo
    # Transpose local-space vectors to world-space for materials requiring world orientation [S1]
    posed_svo["world_vectors"] = [transform(v, camera.transform) for v in posed_svo["local_vectors"]]
    # Voxelize: sample SVO at chosen octree level into frustum-aligned voxel grid [S1]
    # [G] Rasterize octree samples into grid
    voxel_grid = posed_svo
    # Purge original geometry and rig data from memory [S1]
    del asset_data["original_geometry"]
    # Handle grouping: combine dense asset groups into fewer, larger voxel spaces [S1]
    if group_id is not None:
        # [G] Combine grids for ray-tracing efficiency
        voxel_grid = merge_voxel_grids(voxel_grid, get_group_voxels(group_id))
    return voxel_grid
```

### Parameters and Typical Values
*   **LOD Levels:** Not explicitly defined in bit-depths, but categorized as: Light Proxy, Simplified Binary, Partial Voxel, Full SVO [S1].
*   **Resolution:** High resolution is recommended for assets with overlapping details (e.g., feathers) to maintain fidelity after animation [S1].
*   **Grouping:** Dense asset groups are combined automatically to minimize tracking [S1].

### Edge Cases and Pitfalls
*   **Overlapping Elements:** Static SVOs with overlapping geometry (e.g., feathers) may not separate cleanly when animated. **Mitigation:** Regenerate SVOs with high resolution and optimal posing, splaying elements apart as much as possible [S1].
*   **World-Space Material Dependencies:** Materials requiring world-space orientation (e.g., side-specific wing coloring) fail if only local data is stored. **Mitigation:** Store local-space vectors in the SVO and transpose them at render time [S1].
*   **Rig Emulation Cost:** Attempting to emulate complex rigs (thousands of joints) at render time is inefficient. **Mitigation:** Use blend shapes instead [S1].

## Operating characteristics
*   **Load Time Reduction:**
    *   40 assets: 74% reduction (339s to 89s) [S1].
    *   400 assets: 89% reduction (3451s to 374s) [S1].
*   **Memory Reduction:**
    *   40 assets: 56% reduction (4253 MB to 1189 MB) [S1].
    *   400 assets: 82% reduction (15262 MB to 2816 MB) [S1].
*   **Voxelization Time:** Grouped SVOs reduced voxelization times by up to 99% (416 min to 10 min) compared to full geometry assets [S1].
*   **Render Time:** Grouped SVOs reduced render times by 95% (264 min to 12 min) compared to full geometry assets [S1].
*   **Strengths:** Consistent input to the renderer, reduced memory footprint, faster voxelization, efficient handling of thousands of assets [S1].
*   **Limitations:** Requires offline preparation; static SVOs may suffer from overlap artifacts if not carefully posed; blend shapes may not capture all rig nuances [S1].

## Background
This pipeline was developed for the production of *Rio 2* at Blue Sky Studios [S1]. It was created to handle the complexity of characters, props, and vegetation in elaborate scenes, such as a choreographed flying dance and a macaw soccer match [S1]. The pipeline consolidates tools from Set Dressing, Animation, Crowds, and Fur departments into a shared library, enabling flexible shot design and efficient ray-tracing [S1]. It is analogous to SVO data structures used in real-time ray-tracing applications but adapted for a high-end offline rendering workflow [S1].

## Sources
[S1] Using sparse voxel octrees in a level-of-detail pipeline for Rio 2, https://doi.org/10.1145/2614106.2614112, fetched 2026-11-04
