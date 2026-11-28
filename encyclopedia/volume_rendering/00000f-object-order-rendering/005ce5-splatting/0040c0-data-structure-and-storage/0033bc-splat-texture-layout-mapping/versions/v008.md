# Splat Texture Layout & Mapping

Also called: Splat Data Texture Mapping

## Overview
This method defines the GPU-accessible memory layout for volumetric splat primitives, mapping discrete splat data into 3D texture structures to enable efficient attribute fetching during rasterization [S1, unofficial]. It differs from raw buffer storage by utilizing texture hardware for coordinate-based lookups, which is particularly beneficial when combined with hardware-accelerated atlases [S1, unofficial].

## How it works
**Inputs:** A set of splat primitives with an initial capacity.
**Outputs:** Two 3D textures backed by `Uint32Array` buffers, containing the mapped splat data.

**Data Structures:**
Each splat corresponds to two 16-byte texture records within the 3D textures [S1, unofficial]. The total capacity is determined by the texture dimensions.

**Steps:**
1.  **Calculate Capacity:** Determine the maximum splat capacity supported by the target texture dimensions using `getTextureSize(inputCapacity).maxSplats` [S1, unofficial].
2.  **Allocate Memory:** If the input capacity differs from `maxSplats`, allocate new `Uint32Array` buffers of size `maxSplats * 4` [S1, unofficial].
3.  **Copy Data:** Copy the original splat data into the newly allocated buffers [S1, unofficial].
4.  **Map Coordinates:** Map each splat index to a 3D texture coordinate `(ivec3)` [S2, unofficial]. This mapping must handle paging or tiling for large splat counts that exceed single texture volume limits [S2, unofficial].

**Formulas:**
The layer index within the 3D texture is derived from the splat's height and the texture width bits [S1, unofficial]:

$$
\text{layerBits} = \text{SPLAT\_TEX\_WIDTH\_BITS} + \lceil \log_2(\text{height}) \rceil
$$

$$
\text{sourceLayerMask} = 2^{\text{layerBits}} - 1
$$

**Parameters:**
*   `SPLAT_TEX_WIDTH_BITS`: Not given in the sources.
*   `height`: The height dimension of the splat or texture layer, used to calculate layer bits [S1, unofficial].

**Edge Cases:**
*   **Capacity Mismatch:** If the input capacity does not match the texture's maximum splat capacity, reallocation and copying are required [S1, unofficial].
*   **Large Splat Counts:** For counts exceeding the texture volume, explicit paging or tiling logic must be implemented in the coordinate calculation function [S2, unofficial].

## Subtopics

### [Hardware Accelerated Texture Atlases](8597-hardware-accelerated-texture-atlases.md)
This variant organizes per-primitive texture data into large 2D atlas textures to leverage GPU hardware filtering and caching. It builds on Splat Texture Layout & Mapping by consolidating individual primitive textures into a unified structure, enabling efficient, filtered attribute fetching during rendering.

## Operating characteristics
**Time and Memory Complexity:**
Memory usage is proportional to `maxSplats * 4` (for the `Uint32Array` backing) plus the texture storage overhead [S1, unofficial]. Access time depends on texture sampling hardware efficiency [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Leverages GPU hardware filtering and caching; avoids multiple texture bindings and software-based sampling overhead [S1, unofficial].
*   **Limitations:** Requires careful management of texture dimensions and capacity limits; paging logic adds complexity for very large scenes [S2, unofficial].

**When to Use:**
Use when efficient attribute fetching via hardware filtering is required and the scene size fits within texture volume constraints or can be paged [S1, unofficial].

## Background
This method is part of the Gaussian Splatting rendering pipeline, specifically within the data structure and storage layer [S1, unofficial]. It is implemented in libraries such as `gaussian-splat-lite` and `spark` [S1, unofficial].

## Sources
[S1] Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S2] spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
