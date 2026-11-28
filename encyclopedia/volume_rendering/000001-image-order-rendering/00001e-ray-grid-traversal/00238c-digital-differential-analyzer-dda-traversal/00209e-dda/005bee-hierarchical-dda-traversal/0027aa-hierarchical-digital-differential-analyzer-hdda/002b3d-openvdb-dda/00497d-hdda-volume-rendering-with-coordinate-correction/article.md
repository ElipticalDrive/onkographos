# HDDA Volume Rendering with Coordinate Correction

Also called: HDDA with ray-based coordinate lookup

## Overview
This method corrects square and planar artifacts in volume rendering on OpenVDB grids by fixing the voxel coordinate lookup during ray traversal [S1, unofficial]. It replaces the DDA's internal voxel state with a coordinate derived from the ray's position slightly ahead of the current traversal time [S1, unofficial]. It sets itself apart from the standard OpenVDB DDA implementation by altering the coordinate lookup logic within the rendering loop to ensure geometric consistency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin (`start`), ray end (`end`), OpenVDB accessor (`acc`), opacity scalar (`opacity`) [S1, unofficial].
- **Output:** Transmittance value (`float`) [S1, unofficial].

**Data Structures:**
- `nanovdb::Ray<float>`: Stores ray origin, normalized direction, current time `t`, and total length [S1, unofficial].
- `nanovdb::Coord`: Integer coordinate (`ijk`) representing the voxel index [S1, unofficial].
- `nanovdb::HDDA`: Hierarchical DDA state object managing traversal steps and time [S1, unofficial].
- `acc`: Accessor object providing `getValue(ijk)` and `getDim(ijk, ray)` [S1, unofficial].

**Parameters:**
- **Coordinate Offset:** `1.0001`. This small epsilon is added to the DDA time to shift the sampling point slightly ahead of the current voxel boundary [S1, unofficial].

**Algorithm Steps:**
1. Initialize `transmittance` to `1.0` [S1, unofficial].
2. Compute ray direction `dir = end - start` and length `len = dir.length()` [S1, unofficial].
3. Construct `nanovdb::Ray<float> ray` with `start`, `dir/len`, `t=0.0`, and `length=len` [S1, unofficial].
4. Compute initial voxel coordinate `ijk` using `nanovdb::RoundDown<nanovdb::Coord>(ray.start())` [S1, unofficial].
5. Initialize `hdda` object with `ray` and dimensions from `acc.getDim(ijk, ray)` [S1, unofficial].
6. Initialize `t = 0.0` [S1, unofficial].
7. Fetch initial density: `density = acc.getValue(ijk) * opacity` [S1, unofficial].
8. Loop while `hdda.step()` returns true:
    a. Calculate segment length: `dt = hdda.time() - t` [S1, unofficial].
    b. Update transmittance: `transmittance *= exp(-density * dt)` [S1, unofficial].
    c. Update time: `t = hdda.time()` [S1, unofficial].
    d. **Coordinate Correction:** Calculate new `ijk` using the corrected formula:
       $$ijk = \text{nanovdb::RoundDown}<\text{nanovdb::Coord}>(\text{ray}(hdda.time() + 1.0001))$$
       This replaces the buggy `ijk = hdda.voxel()` [S1, unofficial].
    e. Fetch new density: `density = acc.getValue(ijk) * opacity` [S1, unofficial].
    f. Update DDA step size if necessary: `hdda.update(ray, acc.getDim(ijk, ray))` [S1, unofficial].
9. Return `transmittance` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, origin, direction, t, length):
        self.origin = origin
        self.direction = direction
        self.t = t
        self.length = length
    def position_at(self, time):
        return self.origin + self.direction * time

class HDDA:
    def __init__(self, ray, dims):
        self.ray = ray
        self.dims = dims
        self.current_time = 0.0
    def step(self):
        # Advances traversal state; returns True while inside grid
        self.current_time += 1.0
        return self.current_time <= self.ray.length
    def time(self):
        return self.current_time
    def update(self, ray, dims):
        # Adjusts internal step size based on new dimensions
        self.ray = ray
        self.dims = dims

def transmittance_hdda_with_coord_correction(start, end, acc, opacity):
    transmittance = 1.0
    dir_vec = end - start
    length = dir_vec.length()
    
    ray = Ray(origin=start, direction=dir_vec / length, t=0.0, length=length)
    ijk = int(floor(ray.origin))
    
    hdda = HDDA(ray=ray, dims=acc.get_dim(ijk, ray))
    
    t = 0.0
    density = acc.get_value(ijk) * opacity
    
    while hdda.step():
        dt = hdda.time() - t
        transmittance *= exp(-density * dt)
        t = hdda.time()
        
        corrected_time = hdda.time() + 1.0001
        ijk = int(floor(ray.position_at(corrected_time)))
        
        density = acc.get_value(ijk) * opacity
        hdda.update(ray=ray, dims=acc.get_dim(ijk, ray))
        
    return transmittance
```

**Edge Cases and Pitfalls:**
- **Precision:** The offset `1.0001` is critical [S1, unofficial]. Using `hdda.voxel()` directly causes artifacts, but the exact geometric reason is not fully documented; it is empirically observed to eliminate square/planar artifacts [S1, unofficial].
- **Boundary Conditions:** The initial `ijk` calculation uses `RoundDown` on the ray start, ensuring the first voxel is correctly identified at the entry point [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Strengths:** Eliminates visible square/planar artifacts present in the standard OpenVDB OptiX example implementation [S1, unofficial].
- **Limitations:** The theoretical justification for the specific offset value `1.0001` is not provided in the source; it is a heuristic fix [S1, unofficial]. Performance impact relative to the buggy version is not given in the sources [S1, unofficial].

## Background
This method originated from a GitHub issue reported by Kaavay Gupta in May 2024 regarding the OpenVDB library's OptiX integration [S1, unofficial]. The issue highlighted square/planar artifacts in volume rendering when using the provided `transmittanceHDDA` example code [S1, unofficial]. The correction was proposed as a workaround to fix the coordinate sampling logic within the raymarching loop [S1, unofficial]. It is not part of the official OpenVDB library but serves as a community-identified fix for the OptiX backend [S1, unofficial].

## Sources
[S1] [BUG] Square/Planar Artifacts While Volume Rendering NVDB Files Using Optix, https://github.com/AcademySoftwareFoundation/openvdb/issues/1812, fetched 2026-10-25 (unofficial)
