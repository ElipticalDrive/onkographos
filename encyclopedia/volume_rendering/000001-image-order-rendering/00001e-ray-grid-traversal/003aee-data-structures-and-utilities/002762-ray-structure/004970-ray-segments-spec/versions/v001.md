# Ray Segments Spec
Also called: RaySegmentsSpec

## Overview
Ray Segments Spec is a data structure designed to represent ray segments efficiently, particularly in the context of volume rendering and neural radiance fields. It stores segment values, endpoint flags, validity, chunking information, and ray mappings. It differs from sibling structures like [Ray Bundle](2179-ray-bundle.md) by focusing on explicit segment-level metadata rather than raw ray parameters, and from [RayBatch container](4187-raybatch-container.md) by providing granular control over segment validity and chunking.

## Shared principles
The structure relies on parallel arrays to store segment data, enabling efficient indexing and filtering. Key fields include:
- `vals`: A tensor or array containing the values (e.g., positions, densities) for each segment.
- `is_left`: Boolean flags indicating if a segment endpoint is a left boundary.
- `is_right`: Boolean flags indicating if a segment endpoint is a right boundary.
- `is_valid`: Boolean flags marking whether a segment is valid for processing.
- `chunk_starts`: Indices marking the start of each chunk within the `vals` array.
- `chunk_cnts`: Counts of segments per chunk.
- `ray_indices`: Mapping from segment indices back to their originating rays.

This layout supports sparse and structured data access patterns common in volume rendering pipelines.

## Subtopics

### [Ray Bundle](2179-ray-bundle.md)
Stores raw ray parameters (origin, direction) and associated data. Choose when ray-level aggregation is needed.

### [RayBatch container](4187-raybatch-container.md)
Optimized for batched ray processing with uniform structures. Choose for high-throughput rendering.

### [Hit Record Structure](4953-hit-record-structure.md)
Stores intersection results. Choose when hit detection is primary.

## Background
The Ray Segments Spec structure appears in implementations of neural radiance fields, such as those in the CNC project [S1, unofficial]. It facilitates efficient rendering by organizing segments into chunks and managing validity flags.

## Sources
[S1] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
