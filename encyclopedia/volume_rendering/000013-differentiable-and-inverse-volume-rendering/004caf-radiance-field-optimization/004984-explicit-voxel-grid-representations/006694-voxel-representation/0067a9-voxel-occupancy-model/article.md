# Voxel Occupancy Model

Also called: Material-Occupancy Model

## Overview
The Voxel Occupancy Model represents 3D space by assigning each voxel a discrete material type and a continuous occupancy value indicating the density of matter within that voxel [S1, unofficial]. It solves the problem of storing and manipulating terrain or volumetric geometry on low-end platforms where memory is constrained, providing sub-voxel flexibility for tools like erosion that pure material models lack [S1, unofficial]. It differs from Signed Distance Field (SDF) representations by avoiding the complex mental model of distance clamping and far-field filling, and from Hermite data by offering a simpler API for content creation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D grid of voxels.
*   **Output:** A compressed memory layout suitable for rendering or meshing, and a logical model for terrain generation tools.

**Data Structures:**
Each voxel is stored as a 2-byte structure:
1.  **Material Index (1 byte):** An integer index into a global material table. The table size is fixed at 23 entries [S1, unofficial].
2.  **Occupancy (1 byte):** A quantized 8-bit unsigned integer representing the occupancy value [S1, unofficial].

**Logical Definition:**
*   **Material:** An enumeration defining the type of matter (e.g., dirt, stone, air). The material also influences how terrain is shaped around the voxel [S1, unofficial].
*   **Occupancy:** A floating-point value in the range $[0, 1]$.
    *   $0.0$: The voxel is completely empty.
    *   $1.0$: The voxel contains the maximum possible amount of matter. Note that due to the meshing algorithm's "overflow" behavior, occupancy $1.0$ does not imply the voxel is a solid cube; it simply means it is full relative to the algorithm's capacity [S1, unofficial].

**Encoding and Decoding:**
To conserve space, the floating-point occupancy is quantized to 8 bits. The encoding and decoding rules handle the ambiguity of empty voxels by using an explicit "Air" material [S1, unofficial].

Let $v$ be the stored 8-bit unsigned integer value ($0 \le v \le 255$).
Let $m$ be the material index.

The logical occupancy $O$ is calculated as:

$$
O = \begin{cases} 
0.0 & \text{if } m = \text{Air} \\
\frac{v + 1}{256.0} & \text{otherwise}
\end{cases}
$$

This specific decoding formula $(v + 1) / 256.0$ is chosen to ensure that the occupancy is precisely represented in floating point, avoiding the precision loss associated with $v / 255.0$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_voxel(material_index: int, occupancy: float) -> tuple[int, int]:
    """
    Encodes a voxel into 2 bytes.
    
    Args:
        material_index: Index into the material table (0-22).
        occupancy: Logical occupancy value in [0.0, 1.0].
        
    Returns:
        Tuple of (material_byte, occupancy_byte).
    """
    # Determine if this is the Air material
    # Note: The specific integer ID for 'Air' depends on the material table definition.
    # We assume a constant AIR_MATERIAL_ID is defined elsewhere.
    is_air = (material_index == AIR_MATERIAL_ID)
    
    if is_air:
        # If air, occupancy is logically 0.0. 
        # The storage value can be anything, but 0 is standard.
        occ_byte = 0
    else:
        # Clamp occupancy to [0.0, 1.0]
        clamped_occ = max(0.0, min(1.0, occupancy))
        
        # Encode: v = floor(occupancy * 256.0) - 1
        # Inverse of decode: v = occupancy * 256.0 - 1
        v = int(clamped_occ * 256.0 - 1.0)
        
        # Clamp v to valid 8-bit range [0, 255]
        v = max(0, min(255, v))
        occ_byte = v
        
    return (material_index, occ_byte)

def decode_voxel(material_byte: int, occupancy_byte: int) -> tuple[int, float]:
    """
    Decodes a voxel from 2 bytes.
    
    Args:
        material_byte: The material index.
        occupancy_byte: The quantized occupancy value.
        
    Returns:
        Tuple of (material_index, logical_occupancy).
    """
    is_air = (material_byte == AIR_MATERIAL_ID)
    
    if is_air:
        logical_occupancy = 0.0
    else:
        # Decode: occupancy = (v + 1) / 256.0
        logical_occupancy = (occupancy_byte + 1) / 256.0
        
    return (material_byte, logical_occupancy)
```

**Edge Cases and Pitfalls:**
*   **Ambiguity of Empty Voxels:** The model resolves the ambiguity of whether an empty voxel is defined by `occupancy=0` or `material=Air` by mandating the use of an explicit Air material [S1, unofficial].
*   **Meshing Overflow:** Occupancy $1.0$ does not guarantee a cubic shape. The meshing algorithm allows matter to "overflow" between voxels, so the visual representation may differ from the logical occupancy value [S1, unofficial].
*   **Precision:** Using $v/255.0$ for decoding would result in non-precise floating-point representations. The $(v+1)/256.0$ formula is critical for precision [S1, unofficial].

## Subtopics

### [Gaussian Process Occupancy Model](23173-gaussian-process-occupancy-model.md)
The Gaussian Process Occupancy Model defines voxel occupancy as the probability that a point lies on or inside an object, derived from a Gaussian Process (GP) over a geometric classification function.

## Operating characteristics
*   **Memory:** 2 bytes per voxel. This is highly compact, suitable for low-end platforms [S1, unofficial].
*   **Flexibility:** Provides sub-voxel flexibility for tools (e.g., erosion) that pure material models lack, as voxels must be large for practical solutions on low-end hardware [S1, unofficial].
*   **Complexity:** Simpler mental model than Signed Distance Fields (no distance clamping or far-field filling logic) and simpler API than Hermite data (no complex CSG-based tooling required) [S1, unofficial].
*   **Limitations:** Loses the ability to precisely define sharp edges on the surface compared to Hermite data, which is compensated for by using the material value to define geometry shaping [S1, unofficial].

## Background
The Voxel Occupancy Model was developed for voxel terrain storage, specifically targeting low-end platforms where memory and computational resources are limited [S1, unofficial]. It was chosen over other models after experimentation:
*   **Rejected:** Pure Material Model (lacked sub-voxel flexibility for tools like erosion) [S1, unofficial].
*   **Rejected:** Hermite Data (lacked a sufficient API for tools, exposing raw data was too complex) [S1, unofficial].
*   **Rejected:** Signed Distance Field (more complicated mental model, similar content results) [S1, unofficial] [S1, unofficial].

The model was settled upon to balance storage efficiency, tool flexibility, and API simplicity [S1, unofficial].

## Sources
[S1] Voxel terrain: storage, https://zeux.io/2017/03/27/voxel-terrain-storage/, fetched 2026-10-29 (unofficial)
