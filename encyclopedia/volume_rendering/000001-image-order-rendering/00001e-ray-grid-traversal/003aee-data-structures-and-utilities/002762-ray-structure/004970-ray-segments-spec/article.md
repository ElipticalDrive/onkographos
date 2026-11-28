# Ray Segments Spec
Also called: RaySegmentsSpec

## Overview
Ray Segments Spec is a data structure representing ray segments [S1, unofficial]. It stores segment values, endpoint flags, validity, chunking information, and ray mappings [S1, unofficial]. Not given in the sources how it differs from sibling structures or its specific performance characteristics [G].

## Shared principles
The structure represents ray segments with the following fields [S1, unofficial]:
- `vals`: Values or positions for each segment [S1, unofficial].
- `is_left`: Boolean flags for left endpoints [S1, unofficial].
- `is_right`: Boolean flags for right endpoints [S1, unofficial].
- `is_valid`: Validity flags [S1, unofficial].
- `chunk_starts`: Start indices for chunks [S1, unofficial].
- `chunk_cnts`: Counts per chunk [S1, unofficial].
- `ray_indices`: Mapping to original rays [S1, unofficial].
Not given in the sources whether this layout supports sparse access patterns or facilitates efficient rendering [G].

## Subtopics

### [Ray Bundle](2179-ray-bundle.md)
Not given in the sources what it is, how it differs from siblings, or when to choose it [G].

### [RayBatch container](4187-raybatch-container.md)
Not given in the sources what it is, how it differs from siblings, or when to choose it [G].

### [Hit Record Structure](4953-hit-record-structure.md)
Not given in the sources what it is, how it differs from siblings, or when to choose it [G].

### [Plücker Ray Embedding](5132-pl-cker-ray-embedding.md)
Not given in the sources what it is, how it differs from siblings, or when to choose it [G].

## Background
Not given in the sources.

## Sources
[S1] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
