# Isomesh

Also called: isomesh=level, solid 3D isosurfaces

## Overview
Isomesh is a surface extraction technique that triangulates the level set of a 3D scalar field using the Marching Cubes algorithm and renders the result as a shaded, depth-tested mesh [S1, unofficial]. It solves the visual artifacts of volume rendering approaches—specifically the striping inherent in ray-marched iso-shells and the lack of depth testing in MIP/absorption methods that forces the body to occlude the field [S1, unofficial]. Unlike its siblings, it produces smooth, correctly occluded surfaces by leveraging GPU-accelerated rasterization of a pre-computed mesh rather than per-pixel ray marching [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field observable $\sigma$ defined on a grid, and a scalar level value $L$ (the isovalue).
*   **Output:** A `GeometryBasics.Mesh` containing the triangulated isosurface. If the field does not cross $L$ within the domain, the mesh is empty [S1, unofficial].

**Data Structures:**
*   **Field:** A 3D array of floating-point values. The implementation asserts the dimension $d=3$; it is not applicable to 2D fields [S1, unofficial].
*   **Mesh:** A `GeometryBasics.Mesh` object containing vertices and triangle indices. The normals are oriented towards increasing field values [S1, unofficial].

**Parameters:**
*   `isomesh`: The level value $L$ for extraction.
*   `backlight`: Lighting parameter for the mesh. Default is $1$. Setting `backlight=1` lights both sides of the mesh, removing dependence on normal sign. Setting `backlight=0` causes surfaces with normals pointing away from the light to render dark [S1, unofficial].
*   `transparency`: Boolean flag. When `true` combined with a translucent color, it allows the body to remain visible through the mesh [S1, unofficial].
*   `enable_depth`: Implicitly `true` for isomesh. Passing `false` raises an error ("Invalid attribute enable_depth for plot type Mesh") because mesh plots are always depth-tested [S1, unofficial].

**Algorithm Steps:**
1.  **Validation:** Assert that the input field dimension $d=3$. If not, abort [S1, unofficial].
2.  **Extraction:** Execute the Marching Cubes algorithm on the CPU over the plotted region of the field $\sigma$ to extract the isosurface at level $L$. This generates a `GeometryBasics.Mesh` [S1, unofficial].
    *   *Note:* The Marching Cubes implementation runs single-threaded on the CPU [S1, unofficial].
3.  **Plotting:** Lift the field observable to the isosurface mesh and invoke the mesh plotting method (`plot_σ_obs!`). Pass rendering kwargs (e.g., `color`, `transparency`, `backlight`) to `Makie.mesh!` [S1, unofficial].
4.  **Rendering:** Render the mesh with depth testing enabled. The normals point into the region of interest for fields like vorticity or $-\lambda_2$, and out of it for low pressure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plot_isomesh(field_observable, level, backlight=1, transparency=False):
    """
    Extracts and renders an isosurface mesh for a 3D field.
    
    Args:
        field_observable: 3D scalar field array.
        level: Scalar isovalue.
        backlight: Lighting mode (1 for both sides, 0 for front-side only).
        transparency: Boolean for alpha blending.
    """
    # Assert 3D dimension
    if field_observable.ndim != 3:
        raise ValueError("Isomesh requires a 3D field.")

    # Extract mesh using Marching Cubes (single-threaded CPU)
    # Marching Cubes orients normals towards increasing values
    mesh = marching_cubes(field_observable, level)

    # If the field does not cross the level, mesh is empty
    if mesh.is_empty:
        return

    # Render mesh with depth testing (implicit)
    # Pass kwargs to mesh renderer
    render_mesh(
        mesh,
        backlight=backlight,
        transparency=transparency,
        enable_depth=True  # Implicitly true; cannot be set to False
    )
```

**Edge Cases and Pitfalls:**
*   **Performance:** The Marching Cubes step runs single-threaded on the CPU. Performance can be improved by cropping the domain to reduce the region processed [S1, unofficial].
*   **Normal Orientation:** Normals point towards increasing values. For fields where the region of interest is below the level (e.g., low pressure), normals point outward. Use `backlight=1` to ensure visibility regardless of orientation [S1, unofficial].
*   **Depth Testing:** Unlike volume methods (`:mip`, `:absorption`) which default to `enable_depth=false` (drawing the body over the field), isomesh uses depth testing so the body is occluded by the field where appropriate [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The Marching Cubes extraction is $O(N)$ with respect to the number of voxels in the plotted region, but runs single-threaded on the CPU [S1, unofficial].
*   **Memory Complexity:** Requires storage for the field array and the resulting mesh (vertices and indices).
*   **Measured Performance:** On an RTX 4060 8 GB with 8 threads, on a $512 \times 192 \times 192$ domain:
    *   Isomesh update time: 1.48s
    *   Render time: 0.016s
    *   Total per frame: 1.50s [S1, unofficial]
    *   The Marching Cubes step adds 0.65s per frame for approximately 52k triangles, which is about 0.5s more than volume methods, while its draw time is the cheapest among all methods [S1, unofficial].
*   **Strengths:** Produces smooth surfaces with correct depth testing, avoiding the striping of ray-marched iso volumes and the occlusion issues of MIP/absorption [S1, unofficial].
*   **Limitations:** Slower extraction time due to single-threaded CPU execution. Not applicable to 2D fields [S1, unofficial].

## Background
Isomesh was introduced as part of the `viz!` function in the WaterLily.jl ecosystem to provide a solid 3D isosurface visualization option [S1, unofficial]. It was developed to address the visual limitations of existing volume rendering methods in the library, specifically the striped appearance of `:iso` ray marching and the lack of depth testing in `:mip` and `:absorption` [S1, unofficial]. The implementation relies on the `Meshing` package for the Marching Cubes algorithm and `GeometryBasics` for the mesh structure [S1, unofficial].

## Sources
[S1] viz!: add isomesh keyword for solid 3D isosurfaces, https://github.com/WaterLily-jl/WaterLily.jl/pull/334, fetched 2026-10-28 (unofficial)
