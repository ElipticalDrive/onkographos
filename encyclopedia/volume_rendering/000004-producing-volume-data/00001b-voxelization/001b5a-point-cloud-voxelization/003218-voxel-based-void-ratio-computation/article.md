# Voxel-based Void Ratio Computation

Also called: Particle Voxelization, Void Ratio Field Computation, Void Ratio Calculation

## Overview
This method computes a void ratio field from a legacy ASCII VTK particle point cloud by binning particle centers into cubic voxels, calculating the solid volume via sphere-box intersection integration, and assigning the resulting void ratio to all particles in the voxel [S1, unofficial]. It focuses on precise geometric volume accounting rather than visual approximation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A legacy VTK `POLYDATA` file containing particle positions and a scalar array for particle radius [S1, unofficial].
- **Output (Default):** A VTK `POLYDATA` file with the computed void ratio stored as `POINT_DATA` on the original particles [S1, unofficial].
- **Output (Optional):** A VTK `UNSTRUCTURED_GRID` file with one `VTK_VOXEL` cell per occupied voxel, containing fields `void_ratio`, `void_ratio_smoothed`, `particle_count`, `solid_volume`, and `measurement_volume` as `CELL_DATA` [S1, unofficial].

**Data Structures:**
- **Voxel Grid:** Implicitly defined by `mins` (minimum bounds), `maxs` (maximum bounds), `voxel_size`, and `dims` (number of voxels per axis) [S1, unofficial].
- **Voxel Data Map:** A dictionary keyed by voxel indices `(ix, iy, iz)` storing `solid_volume`, `particle_count`, `measurement_volume`, and `box_min`/`box_max` [S1, unofficial].

**Steps:**
1. **Read Data:** Load particle positions and radius scalar array from the VTK file [S1, unofficial]. Ensure all scalar/vector/tensor/normals are read via `ReadAll*On` methods to guarantee the radius array is loaded, as it may appear after other arrays [S1, unofficial].
2. **Calculate Voxel Size:** Determine `voxel_size` based on the target average number of particles per voxel (`-n` parameter) and the total number of particles [S1, unofficial]. Calculate `dims` as $\lceil (maxs - mins) / voxel\_size \rceil$ [S1, unofficial].
3. **Bin Particles:** For each particle, compute its voxel index: $indices = \lfloor (points - mins) / size \rfloor$ [S1, unofficial]. Assign the particle to the corresponding voxel [S1, unofficial].
4. **Compute Solid Volume:** For each occupied voxel, calculate the solid volume by summing the volumes of particle spheres clipped to the voxel boundaries using Gauss-Legendre quadrature [S1, unofficial].
5. **Compute Measurement Volume:** For each voxel, calculate the measurement volume [S1, unofficial]. For interior voxels, this is the nominal cubic volume [S1, unofficial]. For boundary voxels, adjust the bounds based on the local extent of particles to avoid overestimating empty space [S1, unofficial].
6. **Compute Void Ratio:** For each voxel, compute the void ratio $e$ using the formula:
   $$ e = \frac{V_{meas} - V_{solid}}{V_{solid}} $$
   where $V_{meas}$ is the measurement volume and $V_{solid}$ is the accumulated solid volume from clipped sphere intersections [S1, unofficial].
7. **Smooth Field (Optional):** Compute a smoothed void ratio field using face-reconstructed Gauss quadrature [S1, unofficial]. This involves averaging values from neighboring voxels and applying two-point Gauss quadrature inside each voxel [S1, unofficial].
8. **Assign Values:** Assign the computed void ratio (and optionally the smoothed value) to all particles belonging to each voxel [S1, unofficial].
9. **Write Output:** Write the results to a VTK `POLYDATA` file [S1, unofficial]. If `--voxel-output` is specified, also write a `UNSTRUCTURED_GRID` file with voxel cells [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def sphere_box_intersection_volume_numpy(center, radius, box_min, box_max, nodes, weights):
    sphere_min = center - radius
    sphere_max = center + radius
    clipped_min = np.maximum(box_min, sphere_min)
    clipped_max = np.minimum(box_max, sphere_max)
    if np.any(clipped_max <= clipped_min):
        return 0.0
    if np.all(box_min <= sphere_min) and np.all(box_max >= sphere_max):
        return (4.0 / 3.0) * math.pi * radius ** 3
    x0, y0, z0 = clipped_min - center
    x1, y1, z1 = clipped_max - center
    x_mid = 0.5 * (x0 + x1)
    y_mid = 0.5 * (y0 + y1)
    x_half = 0.5 * (x1 - x0)
    y_half = 0.5 * (y1 - y0)
    x = x_mid + x_half * nodes
    y = y_mid + y_half * nodes
    xx = x[:, None]
    yy = y[None, :]
    remaining = radius * radius - xx * xx - yy * yy
    valid = remaining > 0.0
    z_radius = np.zeros_like(remaining)
    z_radius[valid] = np.sqrt(remaining[valid])
    z_low = np.maximum(z0, -z_radius)
    z_high = np.minimum(z1, z_radius)
    height = np.maximum(z_high - z_low, 0.0)
    return float(x_half * y_half * np.sum((weights[:, None] * weights[None, :]) * height))

def evaluate_face_reconstruction_at_gauss_plain(e0, face_values):
    ex_minus, ex_plus, ey_minus, ey_plus, ez_minus, ez_plus = face_values
    ax = 0.5 * (ex_plus - ex_minus)
    bx = 0.5 * (ex_plus + ex_minus) - e0
    ay = 0.5 * (ey_plus - ey_minus)
    by = 0.5 * (ey_plus + ey_minus) - e0
    az = 0.5 * (ez_plus - ez_minus)
    bz = 0.5 * (ez_plus + ez_minus) - e0
    gp = 1.0 / math.sqrt(3.0)
    total = 0.0
    count = 0
    for xi in (-gp, gp):
        for eta in (-gp, gp):
            for zeta in (-gp, gp):
                total += e0 + ax * xi + bx * xi * xi + ay * eta + by * eta * eta + az * zeta + bz * zeta * zeta
                count += 1
    return total / float(count)

def compute_void_ratio(points, radii, mins, maxs, particles_per_voxel, sphere_clip_order):
    total_particles = len(points)
    volume_per_particle = np.prod(maxs - mins) / total_particles
    voxel_volume = volume_per_particle * particles_per_voxel
    voxel_size = voxel_volume ** (1.0 / 3.0)
    dims = np.ceil((maxs - mins) / voxel_size).astype(int)
    
    indices = np.floor((points - mins) / voxel_size).astype(int)
    indices = np.clip(indices, 0, dims - 1)
    
    solid_by_voxel = {}
    count_by_voxel = {}
    local_min_by_voxel = {}
    local_max_by_voxel = {}
    particle_indices_by_voxel = {}
    
    for i in range(total_particles):
        key = tuple(indices[i])
        if key not in solid_by_voxel:
            solid_by_voxel[key] = 0.0
            count_by_voxel[key] = 0
            local_min_by_voxel[key] = points[i].copy()
            local_max_by_voxel[key] = points[i].copy()
            particle_indices_by_voxel[key] = []
        count_by_voxel[key] += 1
        local_min_by_voxel[key] = np.minimum(local_min_by_voxel[key], points[i])
        local_max_by_voxel[key] = np.maximum(local_max_by_voxel[key], points[i])
        particle_indices_by_voxel[key].append(i)
        
    nodes, weights = np.polynomial.legendre.leggauss(sphere_clip_order)
    for key in particle_indices_by_voxel:
        solid_volume = 0.0
        for idx in particle_indices_by_voxel[key]:
            center = points[idx]
            radius = radii[idx]
            box_min = mins + key * voxel_size
            box_max = box_min + voxel_size
            solid_volume += sphere_box_intersection_volume_numpy(center, radius, box_min, box_max, nodes, weights)
        solid_by_voxel[key] = solid_volume
        
    void_by_voxel = {}
    smoothed_void_by_voxel = {}
    measurement_volume_by_key = {}
    box_bounds_by_key = {}
    adjusted_boundary_voxels = 0
    
    for key in solid_by_voxel:
        nominal_low = mins + key * voxel_size
        nominal_high = nominal_low + voxel_size
        lengths = []
        for axis in range(3):
            adjusted_low = nominal_low[axis]
            adjusted_high = nominal_high[axis]
            if key[axis] == 0:
                adjusted_low = max(adjusted_low, local_min_by_voxel[key][axis])
            if key[axis] == dims[axis] - 1:
                adjusted_high = min(adjusted_high, local_max_by_voxel[key][axis])
            lengths.append(max(adjusted_high - adjusted_low, 0.0))
            
        measurement_volume = lengths[0] * lengths[1] * lengths[2]
        if measurement_volume < voxel_size ** 3:
            adjusted_boundary_voxels += 1
            
        measurement_volume_by_key[key] = measurement_volume
        box_bounds_by_key[key] = (
            [mins[axis] + key[axis] * voxel_size if key[axis] != 0 else max(mins[axis] + key[axis] * voxel_size, local_min_by_voxel[key][axis]) for axis in range(3)],
            [mins[axis] + (key[axis] + 1) * voxel_size if key[axis] != dims[axis] - 1 else min(mins[axis] + (key[axis] + 1) * voxel_size, local_max_by_voxel[key][axis]) for axis in range(3)]
        )
        
        v_meas = measurement_volume
        v_solid = solid_by_voxel[key]
        void_by_voxel[key] = (v_meas - v_solid) / v_solid if v_solid != 0 else 0.0
        
    voxel_keys = list(solid_by_voxel.keys())
    for key in voxel_keys:
        e0 = void_by_voxel[key]
        face_values = []
        for axis, step in ((0, -1), (0, 1), (1, -1), (1, 1), (2, -1), (2, 1)):
            neighbor = list(key)
            neighbor[axis] += step
            neighbor = tuple(neighbor)
            if neighbor in void_by_voxel:
                face_values.append(0.5 * (e0 + void_by_voxel[neighbor]))
            else:
                face_values.append(e0)
        smoothed_void_by_voxel[key] = evaluate_face_reconstruction_at_gauss_plain(e0, face_values)
        
    particle_void_ratios = {}
    for i in range(total_particles):
        key = tuple(indices[i])
        particle_void_ratios[i] = void_by_voxel.get(key, 0.0)
        
    return {
        "void_ratio": void_by_voxel,
        "void_ratio_smoothed": smoothed_void_by_voxel,
        "particle_void_ratios": particle_void_ratios,
        "measurement_volumes": [measurement_volume_by_key[k] for k in solid_by_voxel],
        "box_bounds": box_bounds_by_key,
        "solid_volumes": solid_by_voxel,
        "particle_counts": count_by_voxel
    }
```

**Parameters:**
- `-n, --particles-per-voxel`: Target average number of particles per occupied voxel. Larger values create larger voxels [S1, unofficial].
- `--sphere-clip-order`: Gauss-Legendre quadrature order for estimating sphere/voxel intersection volumes. Default is 4. Higher values are more accurate but slower [S1, unofficial].
- `--smoothed-void-field`: Name of the face-reconstructed Gauss field written to the output. Default: `void_ratio_smoothed` [S1, unofficial].
- `--voxel-output`: Optional path for an additional VTK file containing actual voxel cells [S1, unofficial].
- `--allow-other-vtk`: Allows the script to run with VTK versions other than 7.1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **VTK Version:** VTK 7.1 is required because newer versions mishandle the file in the workflow. Use `--allow-other-vtk` to override if necessary [S1, unofficial].
- **Data Loading:** VTK legacy readers do not always load every point-data array unless explicitly asked via `ReadAll*On` methods, which is critical because radius appears after several vector arrays [S1, unofficial].
- **Degenerate Extents:** If point cloud extents are degenerate (`max_extent <= 0.0`), the voxel size calculation raises a `RuntimeError` [S1, unofficial].
- **Negative Void Ratios:** Negative void ratios can occur if solid volume exceeds measurement volume, tracked via `negative_void_ratio_voxels` counter [S1, unofficial].

## Subtopics

### [Compute Void Ratio (NumPy)](7451-compute-void-ratio-numpy.md)
Computes the void ratio (volume of empty space divided by volume of solid) for each voxel in a point cloud by binning particle centers into cubic voxels, calculating the solid volume via sphere-box intersection integration, and adjusting for boundary effects. It differs from the plain Python variant by using vectorized NumPy operations for performance, and from higher-level rendering methods by focusing on precise geometric volume accounting rather than visual approximation.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Measured Performance:** Not given in the sources [S1, unofficial].
- **Scaling:** Not given in the sources [S1, unofficial].
- **Strengths:** Not given in the sources [S1, unofficial].
- **Limitations:** Not given in the sources [S1, unofficial].
- **When to Use:** Not given in the sources [S1, unofficial].
- **When Not to Use:** Not given in the sources [S1, unofficial].

## Background
This method originates from the `SammySuliman/point-cloud-voxelization` repository, an unofficial implementation for computing void ratio fields from VTK point clouds [S1, unofficial].

## Sources
[S1] SammySuliman/point-cloud-voxelization, https://github.com/SammySuliman/point-cloud-voxelization, fetched 2026-10-27 (unofficial)
