# Procedural Density Calculation

Also called: calculateCloudDensity, calculateCloudDensityMinimal

## Overview
Procedural Density Calculation generates volumetric density fields on-the-fly using mathematical functions rather than pre-computed voxel grids, enabling infinite resolution and memory efficiency. It distinguishes itself from pre-baked approaches by computing density values directly during rendering or generation passes, often integrating noise and shape modulation to preserve detail without storage overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin, ray direction, bounding volume parameters, and procedural parameters (e.g., sphere ranges, noise seeds, eccentricity, curvature) [S1, S2, unofficial].
- **Output:** Density value $\rho$ and optionally albedo $\alpha$ at a given 3D position $\mathbf{p}$ [S1, unofficial].

**Data Structures:**
- **3D Volume Texture:** A 3D texture array storing density and albedo. The layout is defined as:
  - **R Channel:** Noise value [S1, unofficial].
  - **G Channel:** Local Ambient Occlusion (AO) [S1, unofficial].
  - **B Channel:** Noise-driven height-based baseline albedo [S1, unofficial].
  - **A Channel:** Procedural cloud physical density [S1, unofficial].
- **Cell ID Hash:** Used to decode eccentricity and curvature parameters for shape modulation [S1, unofficial].

**Steps:**
1. **Shape Modulation:** For a given position $\mathbf{p}$, compute a base shape function (e.g., ellipsoid profile) driven by eccentricity, curvature exponent, and center distance [S1, unofficial].
2. **Noise Integration:** Apply on-the-fly procedural 3D noise to modulate the shape. This preserves noise details that might be lost in lower-resolution representations [S1, unofficial].
3. **Density Calculation:** Combine the modulated shape and noise to produce the physical density $\rho$. In the specific implementation of `calculateCloudDensity`, this value is stored in the Alpha channel of the 3D volume texture [S1, unofficial].
4. **Albedo Calculation:** Compute the height-based baseline albedo based on noise and store it in the Blue channel [S1, unofficial].
5. **Rendering Consistency:** During raymarching, retrieve density and albedo directly from the 3D volume texture to ensure 100% rendering consistency between generation and rendering passes [S1, unofficial].
6. **Raymarching Stability:** Use stable, uniform linear marching between bounding bounds instead of unstable backing-up or bifurcated stepping loops to eliminate spots, missing clouds, noise, and banding [S1, unofficial]. Correct empty space skipping using robust positive-step bifurcated step size logic if necessary to eliminate coordinate jumping and infinite loops [S1, unofficial].

**Formulas:**
- **Ellipsoid Profile:** Not explicitly defined in sources, but described as driven by eccentricity $e$, curvature exponent $c$, and center distance $d$ [S1, unofficial].
- **Density Storage:** $\rho(\mathbf{p})$ is stored in $V_{3D}(\mathbf{p}).A$ [S1, unofficial].
- **Albedo Storage:** $\alpha_{base}(\mathbf{p})$ is stored in $V_{3D}(\mathbf{p}).B$ [S1, unofficial].

**Parameters:**
- **Eccentricity:** Decoded from cell ID hash [S1, unofficial].
- **Curvature Exponent:** Decoded from cell ID hash [S1, unofficial].
- **Sphere Size/Position:** Random values within a user-defined range (for CSG variants) [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Low-resolution raymarching can suffer from coordinate jumping and infinite loops if step size logic is not robust [S1, unofficial].
- **Artifacts:** Unstable stepping loops cause spots, banding, and missing clouds [S1, unofficial].
- **Consistency:** Discrepancies between generation and rendering passes can occur if density is not retrieved directly from the same texture used for generation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_cloud_density(p, cell_id_hash, noise_seed):
    """
    Calculate procedural cloud density at position p.
    """
    # Decode shape parameters from hash [S1, unofficial]
    eccentricity = decode_eccentricity(cell_id_hash)
    curvature = decode_curvature(cell_id_hash)
    
    # Compute base shape (e.g., ellipsoid) [S1, unofficial]
    # Note: Exact formula for ellipsoid profile not provided in sources [G]
    base_shape = compute_ellipsoid_profile(p, eccentricity, curvature)
    
    # Apply procedural 3D noise [S1, unofficial]
    noise_val = sample_3d_noise(p, noise_seed)
    
    # Modulate shape with noise [S1, unofficial]
    density = base_shape * (1.0 + noise_val)
    
    # Clamp density to valid range [G]
    density = max(0.0, min(1.0, density))
    
    return density

def calculate_albedo(p, noise_seed):
    """
    Calculate height-based baseline albedo [S1, unofficial].
    """
    noise_val = sample_3d_noise(p, noise_seed)
    # Height-based baseline albedo logic [S1, unofficial]
    # Note: Exact formula for height-based albedo not provided in sources [G]
    albedo = compute_height_based_albedo(p.y, noise_val)
    
    return albedo

def raymarch_clouds(ray_origin, ray_direction, volume_texture):
    """
    Stable linear raymarching through cloud volume [S1, unofficial].
    """
    t = 0.0
    density_accum = 0.0
    color_accum = vec3(0.0)
    
    # Uniform linear marching between bounding bounds [S1, unofficial]
    while t < max_distance:
        p = ray_origin + ray_direction * t
        
        # Retrieve density and albedo directly from 3D volume texture [S1, unofficial]
        density = volume_texture[p].A
        albedo = volume_texture[p].B
        
        # Accumulate density and color [G]
        # Note: Standard volumetric integration logic [G]
        density_accum += density * step_size
        color_accum += albedo * density * step_size
        
        t += step_size
        
    return color_accum, density_accum
```

## Subtopics

### [Height Fog](14108-height-fog.md)
Simulates a dense flat atmospheric volume by blending voxel colors with a height fog color based on submersion factor and visibility curve. It is chosen when a uniform, height-dependent atmospheric effect is needed rather than complex cloud structures. See [Height Fog](14108-height-fog.md).

### Procedural Volume Data Creation
Applies constructive solid geometry (CSG) operations to volume data, procedurally adding and subtracting thousands of spheres from an empty voxel volume using Boolean operations to create complex landscapes. Sphere size and position are random values within a user-defined range. It is used for generating test terrains or complex geometric volumes. See [Procedural Volume Data Creation](#procedural-volume-data-creation).

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples along the ray, as density is computed on-the-fly [S1, unofficial].
- **Memory Complexity:** $O(1)$ for pure procedural calculation without texture storage, or $O(V)$ if a 3D volume texture is used for caching/retrieval consistency [S1, unofficial].
- **Performance:** On-the-fly calculation preserves noise details but requires significant computational power per sample [S1, unofficial].
- **Strengths:** Infinite resolution, no pre-computation time, memory efficient for pure procedural variants [S1, unofficial].
- **Limitations:** Computationally expensive per sample; potential for artifacts if raymarching steps are not stable [S1, unofficial].
- **When to Use:** Real-time rendering where memory is constrained, or when infinite detail is required [S1, unofficial].
- **When Not to Use:** When pre-computed data is available and performance is critical, or when complex static geometry is needed [S2, unofficial].

## Background
The method originates from implementations in cloud rendering optimization, specifically within the `boidsish` project, which focuses on eroded Voronoi weather maps and stable linear raymarching [S1, unofficial]. It also draws from procedural terrain generation techniques using CSG operations on voxel volumes [S2, unofficial].

## Sources
[S1] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
[S2] Visualizing Large Procedural Volumetric Terrains Using Nested Clip-Boxes, https://waseda.repo.nii.ac.jp/records/19543, fetched 2026-10-26 (unofficial)
