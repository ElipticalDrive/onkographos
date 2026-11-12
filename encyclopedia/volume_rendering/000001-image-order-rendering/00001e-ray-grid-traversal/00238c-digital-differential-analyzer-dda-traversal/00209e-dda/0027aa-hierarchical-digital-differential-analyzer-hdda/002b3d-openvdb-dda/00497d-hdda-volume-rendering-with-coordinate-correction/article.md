# HDDA Volume Rendering with Coordinate Correction

Also called: HDDA with ray-based coordinate lookup

## Overview
This method corrects a specific artifact (square/planar discontinuities) in volume rendering using the Hierarchical Digital Differential Analyzer (HDDA) on OpenVDB grids. It solves the problem of incorrect voxel coordinate sampling during ray traversal by replacing the direct use of the DDA's internal voxel state with a coordinate derived from the ray's position slightly ahead of the current traversal time. It sets itself apart from the standard OpenVDB DDA implementation by altering the coordinate lookup logic within the rendering loop to ensure geometric consistency.

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin (`start`), ray end (`end`), OpenVDB accessor (`acc`), opacity scalar (`opacity`).
- **Output:** Transmittance value (`float`).

**Data Structures:**
- `nanovdb::Ray<float>`: Stores ray origin, normalized direction, current time `t`, and total length.
- `nanovdb::Coord`: Integer coordinate (ijk) representing the voxel index.
- `nanovdb::HDDA`: Hierarchical DDA state object managing traversal steps and time.
- `acc`: Accessor object providing `getValue(ijk)` and `getDim(ijk, ray)`.

**Parameters:**
- **Coordinate Offset:** `1.0001f`. This small epsilon is added to the DDA time to shift the sampling point slightly ahead of the current voxel boundary [S1, unofficial].

**Algorithm Steps:**
1. Initialize `transmittance` to `1.0f`.
2. Compute ray direction `dir = end - start` and length `len = dir.length()`.
3. Construct `nanovdb::Ray<float> ray` with `start`, `dir/len`, `t=0.0f`, and `length=len`.
4. Compute initial voxel coordinate `ijk` using `nanovdb::RoundDown<nanovdb::Coord>(ray.start())`. This represents the first hit of the bounding box [S1, unofficial].
5. Initialize `hdda` object with `ray` and dimensions from `acc.getDim(ijk, ray)`.
6. Initialize `t = 0.0f`.
7. Fetch initial density: `density = acc.getValue(ijk) * opacity`.
8. Loop while `hdda.step()` returns true:
    a. Calculate segment length: `dt = hdda.time() - t`.
    b. Update transmittance: `transmittance *= expf(-density * dt)`.
    c. Update time: `t = hdda.time()`.
    d. **Coordinate Correction:** Calculate new `ijk` using the corrected formula:
       $$ijk = \text{nanovdb::RoundDown}<\text{nanovdb::Coord}>(\text{ray}(hdda.time() + 1.0001f))$$
       This replaces the buggy `ijk = hdda.voxel()` [S1, unofficial].
    e. Fetch new density: `density = acc.getValue(ijk) * opacity`.
    f. Update DDA step size if necessary: `hdda.update(ray, acc.getDim(ijk, ray))`.
9. Return `transmittance`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transmittance_hdda_with_coord_correction(start: Vec3f, end: Vec3f, acc: Accessor, opacity: float) -> float:
    transmittance = 1.0f
    dir = end - start
    len = dir.length()
    
    # Construct ray object
    ray = NanovdbRay(start, dir / len, 0.0f, len)
    
    # Initial voxel coordinate
    ijk = NanovdbRoundDownCoord(ray.start())
    
    # Initialize HDDA
    hdda = NanovdbHDDA(ray, acc.get_dim(ijk, ray))
    
    t = 0.0f
    density = acc.get_value(ijk) * opacity
    
    while hdda.step():
        dt = hdda.time() - t
        transmittance *= expf(-density * dt)
        t = hdda.time()
        
        # Corrected coordinate lookup
        # Offset 1.0001f ensures position is slightly ahead of current voxel boundary
        corrected_time = hdda.time() + 1.0001f
        ijk = NanovdbRoundDownCoord(ray(corrupted_time))
        
        density = acc.get_value(ijk) * opacity
        hdda.update(ray, acc.get_dim(ijk, ray))
        
    return transmittance
```

**Edge Cases and Pitfalls:**
- **Precision:** The offset `1.0001f` is critical. Using `hdda.voxel()` directly causes artifacts, but the exact geometric reason is not fully documented; it is empirically observed to eliminate square/planar artifacts [S1, unofficial].
- **Boundary Conditions:** The initial `ijk` calculation uses `RoundDown` on the ray start, ensuring the first voxel is correctly identified at the entry point.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels/tiles traversed by the ray. The coordinate correction adds a constant-time vector evaluation and rounding operation per step.
- **Memory Complexity:** Constant auxiliary memory per ray (HDDA state, ray object, current density).
- **Strengths:** Eliminates visible square/planar artifacts present in the standard OpenVDB OptiX example implementation.
- **Limitations:** The theoretical justification for the specific offset value `1.0001f` is not provided in the source; it is a heuristic fix. Performance impact relative to the buggy version is negligible as it replaces one coordinate fetch with a ray evaluation and rounding.

## Background
This method originated from a GitHub issue reported by Kaavay Gupta in May 2024 regarding the OpenVDB library's OptiX integration [S1, unofficial]. The issue highlighted square/planar artifacts in volume rendering when using the provided `transmittanceHDDA` example code. The correction was proposed as a workaround to fix the coordinate sampling logic within the raymarching loop. It is not part of the official OpenVDB library but serves as a community-identified fix for the OptiX backend.

## Sources
[S1] [BUG] Square/Planar Artifacts While Volume Rendering NVDB Files Using Optix, https://github.com/AcademySoftwareFoundation/openvdb/issues/1812, fetched 2026-10-25 (unofficial)
