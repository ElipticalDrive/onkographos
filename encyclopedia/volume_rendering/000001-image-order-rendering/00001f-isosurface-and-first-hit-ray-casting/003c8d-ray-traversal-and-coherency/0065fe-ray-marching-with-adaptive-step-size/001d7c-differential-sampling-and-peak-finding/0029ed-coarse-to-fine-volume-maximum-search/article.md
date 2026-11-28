# Coarse-to-Fine Volume Maximum Search


## Overview
A two-stage search algorithm to find the approximate location of the minimum value in a volume along a ray [S1, unofficial]. The first stage performs a coarse linear scan, and the second stage refines the position using a divide-by-two approach [S1, unofficial]. Note: despite the title, the source implementation searches for the minimum value.

## How it works
**Inputs:** `start_coord` (3D coordinate), `step_coord` (step vector), `nsteps` (integer step count), `sizef` (volume dimensions), and `sample_vol(coord, sizef)` (sampling function returning `vec4<f32>`) [S1, unofficial].

**Outputs:** `the_coord` (3D coordinate of the found extremum), `the_ref` (scalar extremum value extracted from the `r` component), and `the_value` (full `vec4<f32>` at `the_coord`) [S1, unofficial].

**Data Structures:** `the_ref` (`f32`, initialized to `999999.0`), `the_coord` (`vec3<f32>`, initialized to `start_coord`), `the_value` (`vec4<f32>`, initialized to zero), and `substep_coord` (`vec3<f32>`, initialized to `step_coord`) [S1, unofficial].

**Steps:**

1. **Coarse Linear Scan:** Initialize `the_ref = 999999.0`, `the_coord = start_coord`, and `the_value = vec4<f32>(0.0)` [S1, unofficial]. Loop `iter` from `0.0` to `nstepsf` (exclusive), where `nstepsf = f32(nsteps)` [S1, unofficial]. In each iteration, calculate `coord = start_coord + iter * step_coord` [S1, unofficial]. Sample `value = sample_vol(coord, sizef)` [S1, unofficial]. Extract `reff = value.r` [S1, unofficial]. If `reff < the_ref`, update `the_ref = reff`, `the_coord = coord`, and `the_value = value` [S1, unofficial].

2. **Fine Refinement:** Initialize `substep_coord = step_coord` [S1, unofficial]. Loop `iter2` from `0` to `3` (4 iterations) [S1, unofficial]. In each iteration, halve the step: `substep_coord = substep_coord * 0.5` [S1, unofficial]. Calculate neighbors: `coord1 = the_coord - substep_coord` and `coord2 = the_coord + substep_coord` [S1, unofficial]. Sample neighbors: `value1 = sample_vol(coord1, sizef)` and `value2 = sample_vol(coord2, sizef)` [S1, unofficial]. Extract values: `ref1 = value1.r` and `ref2 = value2.r` [S1, unofficial]. If `ref1 <= the_ref` (deliberate less-equal), update `the_ref = ref1`, `the_coord = coord1`, and `the_value = value1` [S1, unofficial]. Else if `ref2 < the_ref`, update `the_ref = ref2`, `the_coord = coord2`, and `the_value = value2` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_search(start_coord, step_coord, nsteps, sizef, sample_vol):
    # Stage 1: Coarse Linear Scan
    the_ref = 999999.0
    the_coord = start_coord
    the_value = vec4(0.0, 0.0, 0.0, 0.0)
    nstepsf = float(nsteps)
    for iter in range(nsteps):
        coord = start_coord + iter * step_coord
        value = sample_vol(coord, sizef)
        reff = value.r
        if reff < the_ref:
            the_ref = reff
            the_coord = coord
            the_value = value
    # Stage 2: Fine Refinement
    substep_coord = step_coord
    for iter2 in range(4):
        substep_coord = substep_coord * 0.5
        coord1 = the_coord - substep_coord
        coord2 = the_coord + substep_coord
        value1 = sample_vol(coord1, sizef)
        value2 = sample_vol(coord2, sizef)
        ref1 = value1.r
        ref2 = value2.r
        if ref1 <= the_ref:  # Deliberate less-equal
            the_ref = ref1
            the_coord = coord1
            the_value = value1
        elif ref2 < the_ref:
            the_ref = ref2
            the_coord = coord2
            the_value = value2
    return the_coord, the_ref, the_value
```

**Parameters:** `nstepsf` is the number of steps in the coarse linear scan loop [S1, unofficial]. Typical values: Not given in the sources. Refinement iterations: Fixed at 4 [S1, unofficial].

**Edge Cases and Pitfalls:** The refinement loop uses a deliberate less-equal (`<=`) comparison for the first neighbor (`coord1`) to handle ties or specific boundary conditions, while using strict less-than (`<`) for the second neighbor (`coord2`) [S1, unofficial]. Boundary checks for `coord1` and `coord2` are not explicitly detailed in the source snippet; implementation must ensure `sample_vol` handles out-of-bounds coordinates gracefully or clamps them [G].

## Operating characteristics
Not given in the sources.

## Background
Originating from implementations in the pygfx library (WGPU renderer), this method is used in image-order volume rendering to efficiently locate peaks along rays for isosurface extraction or maximum intensity projection [S1, unofficial]. It builds on [Differential Sampling and Peak Finding](1533-differential-sampling-and-peak-finding.md).

## Sources
[S1] volume_ray.wgsl, https://github.com/pygfx/pygfx/blob/main/pygfx/renderers/wgpu/wgsl/volume_ray.wgsl, fetched 2026-10-29 (unofficial)
