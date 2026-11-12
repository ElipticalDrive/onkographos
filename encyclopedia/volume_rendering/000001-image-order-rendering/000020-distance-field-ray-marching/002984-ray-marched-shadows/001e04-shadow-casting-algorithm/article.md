# Shadow Casting Algorithm

Also called: cast shadows algorithm

## Overview
The Shadow Casting Algorithm projects light through a volumetric dataset to compute occlusion, enabling the simultaneous rendering of embedded geometric polygons and volumetric data [S1, unofficial]. It treats illumination as a projection through space, allowing textures and shadows to be handled as extensions of the same resampling pipeline [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volumetric data, embedded geometric polygons, and a light source definition [S1, unofficial].
*   **Output:** An array of total voxel colors representing the final illumination after shadowing [S1, unofficial].

**Data Structures:**
*   **Voxel Array:** A 3D grid storing initial light strengths or opacities [S1, unofficial].
*   **Polygon Data:** Geometric definitions of embedded surfaces, often scan-converted into the volume space [S1, unofficial].

**Steps:**
1.  **Initialization:** Load the 2D profile geometry or polygon data into the array of initial light strengths. Assign all voxels in the volumetric data a slightly non-zero opacity if simulating effects like dust-filled rooms [S1, unofficial].
2.  **Resampling (Shadow Calculation):** Perform shadow calculation by projecting light through the dataset. The standard implementation involves three successive resampling steps to map the light field onto the volume [S1, unofficial].
3.  **Optimization:** Reorganize the order of operations to eliminate one resampling step, yielding crisper shadows [S1, unofficial].
4.  **Attenuation:** Apply tri-linear interpolation during 3D scan-conversion to resample 3D data [S1, unofficial].
5.  **Final Computation:** Compute the array of total voxel colors [S1, unofficial].

**Pitfalls:**
*   **Excessive Blurring:** Frequent tri-linear interpolation for resampling 3D data causes excessive blurring during 3D scan-conversion, resulting in softer polygonal edges than desired [S1, unofficial]. A better filter would reduce this blurring [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tri_linear_interpolate(grid, x, y, z):
    """Interpolate value at fractional coordinates (x, y, z) in a 3D grid."""
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    ix, iy, iz = max(0, min(ix, len(grid)-2)), max(0, min(iy, len(grid[0])-2)), max(0, min(iz, len(grid[0][0])-2))
    v000 = grid[ix][iy][iz]
    v100 = grid[ix+1][iy][iz]
    v010 = grid[ix][iy+1][iz]
    v110 = grid[ix+1][iy+1][iz]
    v001 = grid[ix][iy][iz+1]
    v101 = grid[ix+1][iy][iz+1]
    v011 = grid[ix][iy+1][iz+1]
    v111 = grid[ix+1][iy+1][iz+1]
    lerp = lambda a, b, t: a + t * (b - a)
    c00 = lerp(v000, v100, fx)
    c10 = lerp(v010, v110, fx)
    c01 = lerp(v001, v101, fx)
    c11 = lerp(v011, v111, fx)
    c0 = lerp(c00, c10, fy)
    c1 = lerp(c01, c11, fy)
    return lerp(c0, c1, fz)

def shadow_casting_algorithm(voxel_grid, polygons, light_source):
    """Standard implementation with three successive resampling steps."""
    dims = (len(voxel_grid), len(voxel_grid[0]), len(voxel_grid[0][0]))
    light_array = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]

    # Initialization: load polygon geometry into initial light strengths
    # [G] Standard polygon scan-conversion fills intersected voxels with intensity
    for poly in polygons:
        # poly.scan_convert(dims) would return voxel coordinates here
        pass

    # Assign slightly non-zero opacity to voxels for dust/beam effects
    opacity_grid = [[[1e-4] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]

    # Step 1: Resample based on volume density
    resampled_1 = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(dims[2]):
                resampled_1[i][j][k] = tri_linear_interpolate(light_array, i + 0.5, j + 0.5, k + 0.5) * opacity_grid[i][j][k]

    # Step 2: Resample based on polygon occlusion
    resampled_2 = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(dims[2]):
                resampled_2[i][j][k] = tri_linear_interpolate(resampled_1, i + 0.5, j + 0.5, k + 0.5)

    # Step 3: Final resampling to map to output space
    total_voxel_colors = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(dims[2]):
                total_voxel_colors[i][j][k] = tri_linear_interpolate(resampled_2, i + 0.5, j + 0.5, k + 0.5)

    return total_voxel_colors

def optimized_shadow_casting(voxel_grid, polygons, light_source):
    """Reorganized order to eliminate one resampling step."""
    dims = (len(voxel_grid), len(voxel_grid[0]), len(voxel_grid[0][0]))
    light_array = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]

    for poly in polygons:
        # poly.scan_convert(dims) would return voxel coordinates here
        pass

    opacity_grid = [[[1e-4] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]

    # Optimized sequence combines or reorders steps to skip one resampling
    total_voxel_colors = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for i in range(dims[0]):
        for j in range(dims[1]):
            for k in range(dims[2]):
                val = tri_linear_interpolate(light_array, i + 0.5, j + 0.5, k + 0.5)
                total_voxel_colors[i][j][k] = val * opacity_grid[i][j][k]

    return total_voxel_colors
```

## Subtopics

### [Texture Projection](2002-texture-projection.md)
Texture projection maps 2D textures onto geometric and volumetric data by projecting textures through space onto surfaces. It solves the difficulty of wrapping textures around volumetrically defined objects, where the defining surfaces are not explicitly known, by projecting through space rather than relying on surface parameterization. It differs from direct surface wrapping by treating the texture as a light-like projection, allowing it to be handled as a straightforward extension of the shadow casting algorithm.

### Extruded 2D Profile Rendering via Shadow Casting
A strategy to render geometric objects defined by extrusion of 2D profiles using the shadow casting algorithm to simulate volumetric light shafts [S1, unofficial]. The profile is loaded into the array of initial light strengths, and all voxels in the volumetric data are assigned a slightly non-zero opacity to obtain the effect of sculpted shafts of light passing through a dust-filled room [S1, unofficial]. If the beam is assumed to pass through the volumetric data without attenuation, it can be rendered without a separate shadow pass and without the accompanying 3D representation [S1, unofficial].

### Brute Force Ray Marcher Baseline
A comparison algorithm consisting of a simple ray marcher that uses shadow maps for visibility queries [S2]. The shadow map used for visibility tests is laid out in 8 × 8 blocks such that the texels in each block are scanned in Z-order curve for cache coherence optimization [S2].

## Operating characteristics
*   **Time Complexity:** Adding shadows roughly doubles the time required to compute the array of total voxel colors [S1, unofficial]. However, it does not affect the time required to generate each frame in a rotation sequence when lighting and data are fixed, the observer moves, and specular highlights are absent [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** The addition of textures increases image generation time by roughly 50%, depending on the number and size of textured polygons [S1, unofficial].
*   **Strengths:** Produces images free from aliasing artifacts and handles mixtures of geometric and volumetric data efficiently [S1, unofficial].
*   **Limitations:** The standard implementation suffers from blurring due to tri-linear interpolation. Computational expense can be high for complex polygonal data, potentially requiring hierarchical data structures or unified data structures for both geometric and volumetric data to speed up processing [S1, unofficial].

## Background
The Shadow Casting Algorithm was developed in the context of rendering mixtures of geometric and volumetric data, particularly for applications such as radiation treatment planning [S1, unofficial]. It addresses the challenge of visualizing how light (or radiation beams) interacts with both explicit geometric structures and implicit volumetric data simultaneously [S1, unofficial]. The work acknowledges contributions from the Computer Science and Radiation Oncology departments, utilizing CT scans and electron density maps for validation [S1, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S2] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
