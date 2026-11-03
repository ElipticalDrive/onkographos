# Voxel Material Classification

Also called: ClassifyVoxel

## Overview
Voxel Material Classification determines the physical interaction type of a voxel by unpacking its packed integer representation and mapping it to a discrete collision status. It solves the problem of efficiently querying voxel behavior (solid, liquid, or air) for physics and rendering pipelines without storing redundant metadata. It differs from material palette generation by focusing on immediate binary classification for simulation logic rather than generating texture atlases for rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A single 32-bit unsigned integer (`uint32_t`) representing a packed voxel.
- Output: An enumeration `CollisionSampleStatus` with values `KnownAir`, `KnownSolid`, `KnownLiquid`, or `UnknownBlocked`.

**Data Structures:**
The method relies on a helper function `Utils::UnpackMaterial` which accepts the `uint32_t` voxel and returns a `uint8_t` material identifier. The specific bit-packing layout of the input voxel is not defined in the sources; it is assumed to be consistent with the parent structure's packing scheme [S1, unofficial].

**Steps:**
1. **Unpack:** Call `Utils::UnpackMaterial(voxel)` to extract the material type as a `uint8_t`.
2. **Check Air:** If the unpacked material equals `Utils::Material::Air`, return `KnownAir`.
3. **Check Liquids:** If the unpacked material equals `Utils::Material::Water`, `Utils::Material::Lava`, or `Utils::Material::Oil`, return `KnownLiquid`.
4. **Default to Solid:** For any other material value, return `KnownSolid`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_voxel(voxel: uint32_t) -> CollisionSampleStatus:
    # Unpack material type from packed voxel data
    # Implementation details of UnpackMaterial are external [G]
    material = Utils::UnpackMaterial(voxel)
    
    # Check for empty space
    if material == Utils::Material::Air:
        return CollisionSampleStatus.KnownAir
    
    # Check for fluid types
    if material == Utils::Material::Water or \
       material == Utils::Material::Lava or \
       material == Utils::Material::Oil:
        return CollisionSampleStatus.KnownLiquid
    
    # Default to solid for all other materials
    return CollisionSampleStatus.KnownSolid
```

**Edge Cases and Pitfalls:**
- **Unknown Materials:** Any material ID not explicitly listed as Air or Liquid defaults to Solid. This may incorrectly classify custom or undefined materials as solid [S1, unofficial].
- **Unpack Implementation:** The correctness of classification depends entirely on `Utils::UnpackMaterial` correctly extracting the material field. If the packing format changes, this function must be updated [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel. The operation involves a single function call and a few integer comparisons.
- **Memory Complexity:** $O(1)$. No additional memory is allocated per call.
- **Strengths:** Extremely fast, suitable for real-time physics and collision detection loops where latency is critical.
- **Limitations:** Limited to three categories (Air, Liquid, Solid). It does not provide granular material properties (e.g., friction, density) required for advanced simulation or rendering [S1, unofficial].

## Background
This method is part of the voxel rendering pipeline described in the `asajid2-cell/voxelrender` repository. It serves as a foundational step for chunk generation and simulation tests, ensuring that voxels are correctly categorized before further processing [S1, unofficial].

## Sources
[S1] asajid2-cell/voxelrender, https://github.com/asajid2-cell/voxelrender, fetched 2026-10-27
