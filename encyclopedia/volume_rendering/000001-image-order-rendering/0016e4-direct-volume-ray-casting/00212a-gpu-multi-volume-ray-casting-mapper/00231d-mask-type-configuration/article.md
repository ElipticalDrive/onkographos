# Mask Type Configuration

Also called: Mask Type Selection, SetMaskTypeToBinary, SetMaskTypeToLabelMap

## Overview
Configures the masking strategy for the [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) to determine how voxel visibility is evaluated during ray casting. It selects between a binary mask, which treats all non-zero values as visible, and a label map mask, which allows specific scalar values to define visibility regions. This configuration is distinct from [Label Map Masking for Volume Rendering](4788-label-map-masking-for-volume-rendering.md) by being a property of the mapper's internal state rather than a post-processing or composite blend operation.

## How it works
**Inputs and Outputs:**
- Input: An integer constant representing the desired mask type.
- Output: Updates the internal `MaskType` member variable of the mapper instance.

**Data Structures:**
The mapper maintains an integer field `MaskType` which stores the current configuration. The valid values are defined as enumeration constants within the `vtkGPUMultiVolumeRayCastMapper` class:
- `vtkGPUMultiVolumeRayCastMapper::BinaryMaskType`
- `vtkGPUMultiVolumeRayCastMapper::LabelMapMaskType`

**Steps:**
1. Instantiate or obtain a reference to a `vtkGPUMultiVolumeRayCastMapper` object.
2. Assign the desired mask type constant to the `MaskType` member variable.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming mapper is an instance of vtkGPUMultiVolumeRayCastMapper

# To set binary mask type
mapper.MaskType = vtkGPUMultiVolumeRayCastMapper.BinaryMaskType

# To set label map mask type
mapper.MaskType = vtkGPUMultiVolumeRayCastMapper.LabelMapMaskType
```

**Parameters:**
- `MaskType`: Integer enum.
  - `BinaryMaskType`: Uses a binary mask where any non-zero voxel in the mask volume is considered visible.
  - `LabelMapMaskType`: Uses a label map mask where specific integer labels in the mask volume define visibility.

**Edge Cases and Pitfalls:**
- The mask volume must be provided via `SetTransformedInput` or `SetTransformedInput2` depending on the mapper's multi-volume configuration.
- If the mask type is changed after rendering has begun, the internal state may need to be invalidated or re-initialized, though specific invalidation logic is not detailed in the sources [S1, S2, unofficial].
- The sources do not specify the behavior if an invalid integer value is assigned to `MaskType` [S1, S2, unofficial].

## Subtopics

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Strengths: Allows flexible selection of masking semantics without changing the core ray casting algorithm.
- Limitations: The sources do not provide performance metrics or memory overhead details for either mask type [S1, S2, unofficial].

## Background
This configuration method is part of the VTK (Visualization Toolkit) `vtkGPUMultiVolumeRayCastMapper` class, which supports GPU-accelerated volume rendering with multiple volumes and masking capabilities. The specific methods for setting the mask type are documented in unofficial VTK-related repositories and code samples [S1, S2, unofficial].

## Sources
[S1] karlkrissian/vtkMultiVolRen, https://github.com/karlkrissian/vtkMultiVolRen, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
