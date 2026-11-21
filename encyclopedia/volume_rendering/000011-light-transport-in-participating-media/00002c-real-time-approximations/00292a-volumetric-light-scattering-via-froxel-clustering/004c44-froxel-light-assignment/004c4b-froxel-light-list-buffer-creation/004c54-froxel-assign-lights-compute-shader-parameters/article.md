# Froxel Assign Lights Compute Shader Parameters

Also called: FFroxelAssignLightsCS::FParameters

## Overview
Defines the shader parameter structure for the compute shader that assigns lights to froxels during the volumetric light scattering pass [S1, unofficial]. It provides the GPU kernel with references to the froxel grid state, view parameters, light data, offsets, and atomic counters required for populating the light index buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   `FroxelSharedUB`: `SHADER_PARAMETER_STRUCT_REF` referencing the froxel shared uniform buffer [S1, unofficial].
*   `View`: `SHADER_PARAMETER_STRUCT_REF` referencing the view uniform buffer [S1, unofficial].
*   `Lights`: `SRV` pointing to `StructuredBuffer<FFroxelLightData>` [S1, unofficial].
*   `FroxelLightOffsets`: `SRV` pointing to `StructuredBuffer<uint>` [S1, unofficial].
*   `TotalIndices`: `SRV` pointing to `StructuredBuffer<uint>` [S1, unofficial].
*   `FroxelHeads`: `UAV` pointing to `RWStructuredBuffer<uint>` [S1, unofficial].
*   `FroxelLightIndices`: `UAV` pointing to `RWStructuredBuffer<uint>` [S1, unofficial].

**Data Structures:**
The parameter structure `FFroxelAssignLightsCS::FParameters` contains the fields listed above [S1, unofficial].

**Numbered Steps:**
Not given in the sources.

**Formulas:**
Not given in the sources.

**Parameters:**
*   `FroxelSharedUB`: Shared uniform buffer reference [S1, unofficial].
*   `View`: View uniform buffer reference [S1, unofficial].
*   `Lights`: Structured buffer of light data [S1, unofficial].
*   `FroxelLightOffsets`: Structured buffer of offsets [S1, unofficial].
*   `TotalIndices`: Structured buffer of total assignment count [S1, unofficial].
*   `FroxelHeads`: Unordered access view of atomic counters [S1, unofficial].
*   `FroxelLightIndices`: Unordered access view for assigned light indices [S1, unofficial].
Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
The compute shader kernel logic is not provided in the sources [S1, unofficial]. The parameter structure is defined as follows:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class FFroxelAssignLightsCS_FParameters:
    def __init__(self, froxel_shared_ub, view_ub, lights_srv, offsets_srv, total_indices_srv, heads_uav, indices_uav):
        self.FroxelSharedUB = froxel_shared_ub
        self.View = view_ub
        self.Lights = lights_srv
        self.FroxelLightOffsets = offsets_srv
        self.TotalIndices = total_indices_srv
        self.FroxelHeads = heads_uav
        self.FroxelLightIndices = indices_uav

# Kernel logic not provided in sources [S1, unofficial]
```

## Subtopics

### Froxel Total Indices Compute Shader Parameters
Defines the parameter structure for the compute shader that calculates the total number of light-froxel assignments [S1, unofficial].

**Layout:**
`FFroxelTotalIndicesCS::FParameters` contains:
1. `FroxelSharedUB`: `SHADER_PARAMETER_STRUCT_REF` [S1, unofficial].
2. `FroxelLightCounts`: `SRV` pointing to `StructuredBuffer<uint>` [S1, unofficial].
3. `FroxelLightOffsets`: `SRV` pointing to `StructuredBuffer<uint>` [S1, unofficial].
4. `TotalIndices`: `UAV` pointing to `RWStructuredBuffer<uint>` [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
