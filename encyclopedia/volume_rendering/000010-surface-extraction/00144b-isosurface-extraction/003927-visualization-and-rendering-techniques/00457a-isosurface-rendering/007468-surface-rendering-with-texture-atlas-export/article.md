# Surface Rendering with Texture Atlas Export

Also called: OBJ/VRML Integration, surface texture export from volume rendering

## Overview
This method exports a textured 3D surface mesh directly from volume rendering by baking the volume rendering appearance (window/level, color lookup tables, and projection) onto the surface mesh, allowing export as OBJ or VRML with retained color maps [S1, unofficial]. It differs from standard isosurface rendering by decoupling the surface geometry generation (which uses a pre-existing mask-based surface) from the texturing process, which samples the underlying volumetric data to ensure the texture matches the visual appearance of the volume render exactly [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Surface mesh: Vertices ($V$), normals ($N$), and faces ($F$) provided by `Project().surface_dict` [S1, unofficial].
    *   Volume data: CT/MRI volume matrix ($M$) of type `int16` with dimensions $Z \times Y \times X$ provided by `Slice().matrix` [S1, unofficial].
    *   Voxel spacing: Spacing in millimeters ($S_x, S_y, S_z$) provided by `Slice().spacing` [S1, unofficial].
    *   Rendering parameters: Window Width ($WW$), Window Level ($WL$), and a 16-bit preset Color Lookup Table (CLUT) dictionary provided by `project.raycasting_preset` [S1, unofficial].
*   **Outputs:**
    *   A 2D texture atlas image (PNG) containing the baked colors.
    *   A mesh file (OBJ or VRML) with UV coordinates mapping each triangle to a region in the atlas.

**Data Structures**
*   **Volume Matrix:** A 3D array $M[z][y][x]$ storing Hounsfield Units (HU) or intensity values.
*   **Texture Atlas:** A 2D RGB image, default resolution $5000 \times 5000$ pixels [S1, unofficial].
*   **UV Coordinates:** For each triangle $f_i$ in the mesh, a set of three 2D UV pairs $(u_1, v_1), (u_2, v_2), (u_3, v_3)$ are assigned sequentially within the atlas [S1, unofficial].

**Algorithm Steps**
1.  **Spatial Alignment Correction:**
    Adjust the mesh coordinates to align with the volume grid. The mesh vertices are shifted by $mesh\_min - [10, 10, 10]$ to reverse Y-axis flips inherent in VTK image data imports, ensuring volumetric arrays are raycasted onto valid grid spaces and not empty areas [S1, unofficial].

2.  **Ray Casting and Sampling:**
    For each triangle $f_i$ in the surface mesh:
    a.  Shoot rays along the surface normals ($N$) into the dense CT/MRI volume [S1, unofficial].
    b.  Dynamically sample HU values from the volume matrix $M$ using sub-voxel trilinear interpolation to obtain precise intensity values at the ray intersection points [S1, unofficial].

3.  **Color Accumulation:**
    Apply the active Window Width ($WW$) and Window Level ($WL$) settings along with the 16-bit preset CLUT to the sampled HU values. Accumulate genuine RGB colors using Front-to-Back opacity accumulation to determine the final color for the triangle [S1, unofficial].

4.  **Atlas Packing:**
    Pack all distinct triangle color mappings into a single 2D atlas image. The default atlas resolution is $5000 \times 5000$ pixels [S1, unofficial]. Assign sub-pixel UV coordinates sequentially to the matching mesh faces, ensuring each triangle maps to a unique, non-overlapping region in the atlas [S1, unofficial].

5.  **Export Formatting:**
    Generate the output file format with specific syntax constraints:
    *   **OBJ:** Sanitize whitespace inside saved `.png` filenames by replacing spaces with underscores to prevent parsing failures in legacy MTL managers like MeshLab [S1, unofficial].
    *   **VRML:** Format V2.0 structural arrays strictly, ensuring commas are internal and all MF-Values are enclosed in `[ ]` brackets to bypass OutOfMemory crashes in 32-bit viewers like FreeWRL [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_surface_texture(mesh, volume_matrix, spacing, ww, wl, clut, atlas_size=5000):
    """
    Generates a texture atlas for a surface mesh based on volume rendering parameters.
    
    Args:
        mesh: Dict containing 'vertices', 'normals', 'faces'.
        volume_matrix: 3D array (Z, Y, X) of int16.
        spacing: Tuple (sx, sy, sz) in mm.
        ww: Window Width.
        wl: Window Level.
        clut: 16-bit preset CLUT dictionary.
        atlas_size: Resolution of the output atlas (default 5000).
        
    Returns:
        atlas_image: 2D RGB array.
        uv_map: Dictionary mapping face index to UV coordinates.
    """
    # Initialize atlas
    atlas = np.zeros((atlas_size, atlas_size, 3), dtype=np.uint8)
    uv_map = {}
    
    # Current position in atlas for sequential packing
    current_atlas_y = 0
    current_atlas_x = 0
    
    # Process each triangle
    for face_idx, face in enumerate(mesh['faces']):
        v0, v1, v2 = mesh['vertices'][face[0]], mesh['vertices'][face[1]], mesh['vertices'][face[2]]
        n0, n1, n2 = mesh['normals'][face[0]], mesh['normals'][face[1]], mesh['normals'][face[2]]
        
        # Calculate centroid and average normal for ray casting
        centroid = (v0 + v1 + v2) / 3.0
        normal = (n0 + n1 + n2) / 3.0
        
        # Apply spatial alignment correction
        # Note: mesh_min is derived from the original volume bounds
        corrected_centroid = centroid - np.array([10, 10, 10]) 
        
        # Ray cast into volume
        # Determine intersection point with volume grid
        # This step involves transforming mesh coordinates to volume index space
        # using spacing and volume dimensions.
        volume_coords = mesh_to_volume_index(corrected_centroid, spacing, volume_matrix.shape)
        
        # Sample HU value using trilinear interpolation
        hu_value = trilinear_interpolate(volume_matrix, volume_coords)
        
        # Map HU to RGB using WW/WL and CLUT
        # Front-to-back accumulation is simulated here by direct mapping 
        # as the surface is opaque in this context, or accumulated if transparency is enabled.
        # The source specifies "accumulate genuine RGB colors using Front-to-Back opacity accumulation"
        # but for a single surface point, this typically resolves to the color at the intersection.
        rgb_color = apply_clut_and_ww_wl(hu_value, ww, wl, clut)
        
        # Assign UV coordinates in the atlas
        # Simple sequential packing for demonstration; real implementation may use packing algorithms
        u0, v0 = current_atlas_x / atlas_size, current_atlas_y / atlas_size
        u1, v1 = (current_atlas_x + 1) / atlas_size, current_atlas_y / atlas_size
        u2, v2 = current_atlas_x / atlas_size, (current_atlas_y + 1) / atlas_size
        
        uv_map[face_idx] = [(u0, v0), (u1, v1), (u2, v2)]
        
        # Fill atlas region with color (simplified)
        # In reality, the region is reserved and filled based on precise UV mapping
        atlas[current_atlas_y, current_atlas_x] = rgb_color
        
        # Move to next position in atlas
        current_atlas_x += 1
        if current_atlas_x >= atlas_size:
            current_atlas_x = 0
            current_atlas_y += 1
            
    return atlas, uv_map

def mesh_to_volume_index(point, spacing, volume_shape):
    """
    Converts a world-space point to volume index space.
    """
    # Assuming origin at 0,0,0 for simplicity; real implementation uses volume origin
    x_idx = point[0] / spacing[0]
    y_idx = point[1] / spacing[1]
    z_idx = point[2] / spacing[2]
    return (z_idx, y_idx, x_idx)

def trilinear_interpolate(volume, coords):
    """
    Performs trilinear interpolation on the volume at the given coordinates.
    """
    # Clamp coordinates to volume bounds
    z, y, x = coords
    z = max(0, min(z, volume.shape[0] - 1))
    y = max(0, min(y, volume.shape[1] - 1))
    x = max(0, min(x, volume.shape[2] - 1))
    
    # Extract surrounding voxels and interpolate
    # Implementation details omitted for brevity; standard trilinear interpolation
    return volume[int(z), int(y), int(x)] # Placeholder for actual interpolation

def apply_clut_and_ww_wl(hu, ww, wl, clut):
    """
    Maps HU value to RGB using WW/WL and CLUT.
    """
    # Normalize HU to 0-1 range based on WW/WL
    normalized = (hu - (wl - ww/2)) / ww
    normalized = max(0, min(1, normalized))
    
    # Map to 16-bit index for CLUT
    clut_index = int(normalized * 65535)
    
    # Get RGB from CLUT
    return clut[clut_index]
```

**Parameters**
*   **Atlas Resolution:** $5000 \times 5000$ pixels (default) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Volumetric Spatial Alignment:** Mesh coordinates must be shifted by $mesh\_min - [10, 10, 10]$ to reverse Y-axis flips inherent in VTK image data imports. Failure to do so results in raycasting onto empty grid spaces [S1, unofficial].
*   **OBJ Format Parsing:** Whitespace inside saved `.png` filenames must be sanitized (replaced with underscores) to prevent parsing failures in legacy MTL managers like MeshLab [S1, unofficial].
*   **VRML Format Crashes:** VRML V2.0 structural arrays must strictly format commas as internal and enclose all MF-Values inside `[ ]` brackets. Failure to do so causes OutOfMemory crashes in native 32-bit viewers like FreeWRL [S1, unofficial].
*   **Large Meshes:** VRML export may not work reliably for very large meshes (e.g., millions of faces) due to format and viewer limitations. OBJ is recommended for large medical volumes as it streams geometry more efficiently [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles in the surface mesh, as each triangle requires ray casting and color computation [S1, unofficial].
*   **Memory Complexity:** Dominated by the texture atlas size ($5000 \times 5000 \times 3$ bytes $\approx 75$ MB) and the volume data [S1, unofficial].
*   **Performance:** The core texture generation is implemented in Rust for performance, converting legacy Python/Cython wrappers [S1, unofficial].
*   **Strengths:** Produces production-quality textured OBJ/VRML files with a single click; retains exact volume rendering appearance (WW/WL, CLUT) in the exported mesh; pure Rust execution ensures robustness [S1, unofficial].
*   **Limitations:** VRML export is unreliable for large meshes; requires pre-existing surface geometry (does not auto-segment); relies on specific viewer compliance for VRML syntax [S1, unofficial].
*   **When to use:** When exporting CT/MRI surfaces for use in external tools like Blender, MeshLab, Unity, or for 3D printing with colors [S1, unofficial].
*   **When not to use:** For very large meshes where VRML is required; when automatic segmentation is preferred over manual surface creation [S1, unofficial].

## Background
*   **Origin:** Based on the `medical_triangle_texture` algorithm by Thiago Franco de Moraes (2016) [S1, unofficial].
*   **History:** The method was implemented in InVesalius (invesalius3) to bridge the gap between volume rendering and textured mesh export, allowing users to export production-quality textured OBJ or VRML files directly from the volume rendering view [S1, unofficial].
*   **Usage:** Used in InVesalius for exporting CT/MRI surfaces as OBJ or VRML files for use in Blender, MeshLab, Unity, or 3D printing [S1, unofficial].

## Sources
[S1] Surface Texture Export from Volume Rendering (OBJ/VRML Integration), https://github.com/invesalius/invesalius3/pull/1291, fetched 2026-10-26 (unofficial)
