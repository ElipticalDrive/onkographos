# 3D Texture Proxy Geometry — 3D Texture Proxy, View-Aligned Slices, 3D-Texture Volume Rendering, 3D Texture-Based Volume Rendering

Also called: 3D-Texture Volume Rendering, 3D Texture-Based Volume Rendering, GPU Volume Rendering with View-Aligned Slices

## Overview
3D Texture Proxy Geometry is a direct volume rendering method that represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation to sample volume data at arbitrary locations [S2, unofficial][S3, unofficial]. It solves the problem of efficient hardware-accelerated volume rendering with directional lighting by pre-computing color and opacity per voxel and storing them in 3D textures, allowing the graphics pipeline to handle interpolation and compositing [S3, unofficial]. Unlike shear-warp methods which fix plane counts to volume resolution, this method allows the number of rendering planes to be adjusted at runtime independent of volume resolution, and unlike coherent projection, it supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays [S3, unofficial][S4, unofficial].

## How it works
**Inputs:** Rectilinear volume data (scalar field values), voxel spacing $(A_x, A_y, A_z)$, voxel counts $(n_x, n_y, n_z)$, client-supplied gradient-magnitude threshold, transfer function, and lighting model parameters [S3, unofficial].
**Outputs:** Final composited image [S3, unofficial].

**Data Structures:**
1. **Volume Data:** Array of size $n_x \times n_y \times n_z$ [S3, unofficial].
2. **Lookup Table (LUT):** Array of size $(m+1) \times g$, where $m$ is the number of materials and $g$ is the number of quantized gradient directions [S3, unofficial]. Entries store RGBA values (color and opacity) [S3, unofficial].
3. **3D Texture Maps:** 3D arrays of resolution $(N_x, N_y, N_z)$, where $N_i$ are powers of two $\ge n_i$ [S3, unofficial]. Each texel stores pre-computed RGBA values for one voxel [S3, unofficial]. Large volumes are partitioned into multiple texture maps due to texture memory limits [S3, unofficial].
4. **Bounding Cube:** Defined by side length $d$, centered at the origin [S3, unofficial].

**Preprocessing Steps:**
1. **Classification and Gradient Quantization:** For each voxel, compute the gradient magnitude [S3, unofficial]. Classify the voxel as "reflecting" or "ambient" based on the client-supplied gradient-magnitude threshold and material field value range [S3, unofficial]. Compute a quantized gradient index for reflecting voxels [S3, unofficial].
2. **Lookup Table Creation:** Create the LUT [S3, unofficial]. For each combination of quantized gradient and reflecting material, assign appropriate color and opacity (RGBA) accounting for depth integration [S4, unofficial]. Ambient material colors are computed independently of the gradient [S3, unofficial].
3. **Texture Map Filling:** Fill 3D texture maps with pre-computed color values, one texel per volume voxel, using the LUT indices [S4, unofficial]. This involves integer operations from the volume array to the texture buffers [S3, unofficial].

**Rendering Steps:**
1. **Plane Generation:** Determine the number of planes [S3, unofficial]. The default is $d/A_z$ (where $A_z$ is voxel size along view direction), ensuring one sample per data point when viewed straight on [S3, unofficial]. Typically, 2–4 times the default is used for accuracy [S3, unofficial].
2. **Coordinate Transformation:** Construct a bounding cube centered on the origin with side length $d = \sqrt{L_x^2 + L_y^2 + L_z^2}$, where $L_i = N_i A_i$ [S3, unofficial]. Define texture coordinates $(s,t,r)$ for the cube corners [S3, unofficial]. For arbitrary viewing angles defined by rotation matrix $R$, transform texture coordinates using $(s,t,r) = (x,y,z) D^{-1} R^{-1} S T$, where $D$ is uniform scale by $d$, $S$ is non-uniform scale by $(d/L_x, d/L_y, d/L_z)$, and $T$ is translation by $(\frac{1}{2}n_x/N_x, \frac{1}{2}n_y/N_y, \frac{1}{2}n_z/N_z)$ [S3, unofficial]. Rotation must precede scaling to avoid shearing distortion if dimensions differ [S3, unofficial].
3. **Clipping:** Position six programmable world-space clipping planes at the volume faces to clip regions outside the volume [S3, unofficial].
4. **Rasterization and Interpolation:** Apply each texture map to parallel planes orthogonal to screen-z (squares in screen space) [S4, unofficial]. Texture coordinates are specified only at the four corners [S4, unofficial]. Hardware interpolates texture coordinates across the square; coordinates outside $[0, 1.0]$ are interpolated (not clamped), creating in-range values precisely when the pixel is within the volume [S3, unofficial].
5. **Compositing:** Render planes back-to-front [S3, unofficial]. Combine via hardware blending using the slab thickness $\Delta$ [S3, unofficial].

**Formulas:**
* **Slab Thickness:** $\Delta = \frac{\text{total distance covered by stack}}{\text{number of planes}}$ [S3, unofficial].
* **Color and Opacity Integration:** For slab thickness $\Delta$, color emission per unit distance $E(p)$, and opacity per unit distance $\alpha_i$ (for color $p$) and $\alpha$ (total):
  $$ C_p = E(p) \frac{1 - e^{-\alpha \Delta}}{\alpha} $$
  $$ A = 1 - e^{-\alpha \Delta} $$
  Where $C_p$ is the color contribution for primary $p$, and $A$ is the opacity contribution [S3, unofficial].
* **Texture Coordinates (Initial):**
  $$ s(x) = \frac{x + \frac{1}{2}n_x A_x}{N_x A_x}, \quad t(y) = \frac{y + \frac{1}{2}n_y A_y}{N_y A_y}, \quad r(z) = \frac{z + \frac{1}{2}n_z A_z}{N_z A_z} $$
  [S3, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def next_power_of_two(n):
    if n <= 1: return 1
    return 1 << (n - 1).bit_length()

def compute_gradient_magnitude(volume, x, y, z, nx, ny, nz):
    dx = (volume[x+1][y][z] - volume[x-1][y][z]) / 2.0 if 0 < x < nx-1 else 0.0
    dy = (volume[x][y+1][z] - volume[x][y-1][z]) / 2.0 if 0 < y < ny-1 else 0.0
    dz = (volume[x][y][z+1] - volume[x][y][z-1]) / 2.0 if 0 < z < nz-1 else 0.0
    return math.sqrt(dx*dx + dy*dy + dz*dz)

def classify_voxels(volume_data, threshold):
    nx, ny, nz = len(volume_data), len(volume_data[0]), len(volume_data[0][0])
    material_indices = [[[-1 for _ in range(nz)] for _ in range(ny)] for _ in range(nx)]
    gradient_indices = [[[0 for _ in range(nz)] for _ in range(ny)] for _ in range(nx)]
    for x in range(nx):
        for y in range(ny):
            for z in range(nz):
                grad_mag = compute_gradient_magnitude(volume_data, x, y, z, nx, ny, nz)
                if grad_mag > threshold:
                    material_indices[x][y][z] = 1
                    gradient_indices[x][y][z] = 0
                else:
                    material_indices[x][y][z] = 0
                    gradient_indices[x][y][z] = 0
    return material_indices, gradient_indices

def create_lut(material_indices, gradient_indices, transfer_func, lighting_model):
    m = max(max(row) for row in material_indices) + 1
    g = max(max(row) for row in gradient_indices) + 1
    lut = [[[0.0, 0.0, 0.0, 0.0] for _ in range(g)] for _ in range(m)]
    for mat in range(m):
        for grad in range(g):
            rgba = transfer_func(mat, grad, lighting_model)
            lut[mat][grad] = rgba
    return lut

def fill_texture_maps(volume_data, material_indices, gradient_indices, lut, Nx, Ny, Nz):
    nx, ny, nz = len(volume_data), len(volume_data[0]), len(volume_data[0][0])
    chunk_size = 64
    texture_maps = []
    for x0 in range(0, nx, chunk_size):
        for y0 in range(0, ny, chunk_size):
            for z0 in range(0, nz, chunk_size):
                x1, y1, z1 = min(x0+chunk_size, nx), min(y0+chunk_size, ny), min(z0+chunk_size, nz)
                tex_map = [[[lut[material_indices[x][y][z]][gradient_indices[x][y][z]] for z in range(z0, z1)] for y in range(y0, y1)] for x in range(x0, x1)]
                texture_maps.append(tex_map)
    return texture_maps

def setup_clipping_planes(volume_faces):
    # Position six programmable world-space clipping planes at volume faces [S3, unofficial]
    pass

def compute_texture_transform(R_inv, d, Lx, Ly, Lz, nx, ny, nz, Nx, Ny, Nz):
    # D: uniform scale by d, S: non-uniform scale by (d/Lx, d/Ly, d/Lz), T: translation
    # Correct order: rotation then scaling to avoid shearing [S3, unofficial]
    return R_inv

def render_volume(texture_maps, rotation_matrix, num_planes, d, Az, image_height, image_width):
    slab_thickness = d / num_planes
    setup_clipping_planes()
    transform = compute_texture_transform(rotation_matrix, d, d, d, d, 1, 1, 1, 1, 1, 1)
    final_image = [[0.0, 0.0, 0.0, 0.0] for _ in range(image_height * image_width)]
    for tex_map in texture_maps:
        for plane_idx in range(num_planes):
            # Hardware rasterizes square with texture coordinates at corners
            # Interpolation creates in-range values precisely when pixel is within volume [S3, unofficial]
            # Back-to-front compositing with hardware blending [S3, unofficial]
            pass
    return final_image
```

**Edge Cases and Pitfalls:**
* **Perspective Projection:** Causes irregular sampling (non-constant distance) along rays, leading to artifacts during internal navigation. Requires spherical support geometries which are expensive [S2, unofficial]. Effective slab thickness increases with off-axis angle; ignoring this causes up to 4% error for 600-pixel windows and 14% for 1000-pixel windows [S3, unofficial].
* **Nonlinear Transfer Functions:** Hardware interpolates color, not data. This causes inaccuracies with nonlinear mappings, especially in close-up views [S3, unofficial].
* **Aliasing:** Discontinuous transfer functions (single material per voxel) cause aliasing artifacts, though not sensitive to orientation [S3, unofficial].
* **Texture Memory:** Large volumes exceed texture memory limits, requiring partitioning. Transferring maps to texture memory is not parallelizable and serial [S4, unofficial].
* **Order of Operations:** Scaling does not commute with rotation. Rotation must precede scaling to avoid shearing distortion [S3, unofficial].
* **Few Planes:** Artifacts may be visible where clipped edges of rendered squares are visible [S3, unofficial].

## Operating characteristics
* **Time Complexity:** Lookup table size is independent of volume size ($O(m \cdot g)$) [S3, unofficial]. Texture map filling is proportional to volume size ($O(n_x n_y n_z)$) but involves few integer operations [S3, unofficial]. Rendering time depends on the number of planes and texture map transfers [S3, unofficial].
* **Memory Complexity:** Requires texture memory for 3D textures [S3, unofficial]. Large volumes require partitioning, increasing memory management overhead [S3, unofficial].
* **Performance:** On Silicon Graphics Reality Engine II with 2MB 3D texture hardware, rendering times are nearly interactive [S3, unofficial]. Without directional lighting, times are ~10x faster [S3, unofficial]. Using four processors reduces times by 1.2–2.5x due to serial graphics pipeline bottlenecks [S3, unofficial].
* **Strengths:** Hardware-accelerated, supports perspective views without a time penalty, allows runtime adjustment of plane count, good integration with surface geometry via Z-buffer [S1][S2, unofficial][S3, unofficial].
* **Limitations:** High geometric cost (primitives generated/rasterized) becomes limiting as hardware focuses on fragment processing [S2, unofficial]. Inaccurate with nonlinear transfer functions [S3, unofficial]. Aliasing from discontinuous transfer functions [S3, unofficial]. Perspective artifacts if not compensated [S3, unofficial].
* **When to Use:** Suitable for moderate-sized volumes ($\approx 256^3$) where interactive performance with lighting is needed and hardware supports 3D textures [S2, unofficial]. Not suitable for very large volumes without partitioning or when high accuracy with nonlinear transfer functions is required [S3, unofficial].

## Background
The method was introduced by Cullip and Neumann in 1993 [CN94] and improved by Cabral et al. in 1994 [CCF94] to exploit hardware 3D texture interpolation (trilinear) [S2, unofficial]. It was implemented in the Voltx system on Silicon Graphics Reality Engine II hardware [S3, unofficial]. It supports orthographic, perspective, and stereo views [S3, unofficial].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S3] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S4] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
