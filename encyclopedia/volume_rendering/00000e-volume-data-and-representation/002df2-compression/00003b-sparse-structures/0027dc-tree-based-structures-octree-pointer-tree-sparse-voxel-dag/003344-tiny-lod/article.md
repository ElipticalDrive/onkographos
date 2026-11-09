# Tiny LOD
## Overview
Tiny LOD constructs a hierarchical level-of-detail tree from input splat data, parameterized by a base LOD value and a merge filter flag [S1, unofficial]. It outputs the computed LOD tree, encoded opacity values, and a chunk tree initialized at index 0 [S1, unofficial]. It is distinguished from sibling methods by its explicit `merge_filter` parameter and its specific procedural pipeline [S1, unofficial].
## How it works
**Inputs:**
- `inner_data`: The raw splat data structure [S1, unofficial].
- `lod_base`: A floating-point value ($f32$) determining the base level of detail for the computation [S1, unofficial].
- `merge_filter`: A boolean flag ($bool$) controlling whether a merge filter is applied during the LOD tree computation [S1, unofficial].
- `log_callback`: A function for logging progress or errors [S1, unofficial].

**Outputs:**
- A computed LOD tree structure [S1, unofficial].
- Encoded opacity values associated with the LOD structure [S1, unofficial].
- A built chunk tree starting at index 0 [S1, unofficial].

**Data Structures:**
The specific internal layout of the LOD tree and chunk tree is not detailed in the sources [S1, unofficial]. The opacity values are encoded as part of the LOD structure [S1, unofficial].

**Steps:**
1. **Remove Invalid Splats:** Filter the `inner_data` to remove any splats that are marked as invalid [S1, unofficial].
2. **Compute LOD Tree:** Compute the level-of-detail tree using the `tiny_lod` library, passing the cleaned `inner_data`, the `lod_base` parameter, the `merge_filter` flag, and the `log_callback` [S1, unofficial].
3. **Encode Opacity:** Encode the opacity values for the computed LOD structure [S1, unofficial].
4. **Build Chunk Tree:** Construct the final chunk tree, initializing it at index 0 and using the `log_callback` for progress tracking [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tiny_lod_process(inner_data, lod_base, merge_filter, log_callback):
    # Step 1: Remove invalid splats from the inner data structure
    # [G] Assumes a validity flag on splats
    valid_data = [splat for splat in inner_data if splat.is_valid]
    
    # Step 2: Compute the level-of-detail tree
    # The sources specify calling the tiny_lod library with inner_data, lod_base, merge_filter, and log_callback [S1, unofficial].
    lod_tree = compute_lod_tree(valid_data, lod_base, merge_filter, log_callback)
    
    # Step 3: Encode the opacity values for the computed LOD structure
    # The sources do not specify the exact encoding algorithm [S1, unofficial].
    encoded_opacities = encode_opacity(lod_tree)
    
    # Step 4: Build the chunk tree starting at index 0
    chunk_tree = build_chunk_tree(lod_tree, start_index=0, log_callback=log_callback)
    
    return lod_tree, encoded_opacities, chunk_tree

def compute_lod_tree(data, lod_base, merge_filter, log_callback):
    # The internal algorithm is not detailed in the sources [S1, unofficial].
    pass

def encode_opacity(lod_tree):
    # The internal algorithm is not detailed in the sources [S1, unofficial].
    pass

def build_chunk_tree(lod_tree, start_index, log_callback):
    # The internal algorithm is not detailed in the sources [S1, unofficial].
    pass
```

**Parameters:**
- `lod_base`: $f32$. The base value for LOD computation. Typical values are not specified in the sources [S1, unofficial].
- `merge_filter`: $bool$. A boolean flag controlling whether to apply a merge filter during LOD tree computation. Default behavior is not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
Edge cases and pitfalls are not detailed in the sources [S1, unofficial].

## Subtopics

### [Trilinear Evaluation on Compressed Trees](7074-trilinear-evaluation-on-compressed-trees.md)
This method evaluates a signal value at an arbitrary point $\mathbf{x}$ within a compressed tree structure, supporting trilinear filtering and adaptive traversal. It serves as the evaluation engine for the Compressed Random-Access Tree (CRAT) framework, enabling fine-grain random queries required by rendering algorithms. Unlike block-based decompression schemes (e.g., DXT1) that rely on specialized hardware, this method uses adaptive tree traversal to reconstruct data on the fly, trading raw decompression speed for spatial coherence and high-quality filtering.

### Bhatt LOD
Bhatt LOD is a variant of the Tiny LOD method that uses the `bhatt_lod` library for computation [S1, unofficial]. It follows a similar pipeline: removing invalid splats, computing the LOD tree, encoding opacity, and building the chunk tree [S1, unofficial]. It differs from Tiny LOD by not exposing the `merge_filter` parameter, relying instead on the `lod_base` parameter for control [S1, unofficial].

**Steps:**
1. Remove invalid splats from the inner data structure [S1, unofficial].
2. Compute the level-of-detail tree using the `bhatt_lod` library, passing the inner data, `lod_base`, and a log callback [S1, unofficial].
3. Encode the opacity values for the computed LOD structure [S1, unofficial].
4. Build the chunk tree starting at index 0 with a log callback [S1, unofficial].

**Parameters:**
- `lod_base`: $f32$. The base value for LOD computation [S1, unofficial].

## Operating characteristics
Time and memory complexity, measured performance, scaling behavior, strengths, limitations, and usage recommendations are not detailed in the sources [S1, unofficial].

## Background
Origin, authors, dates, and specific application contexts are not detailed in the sources [S1, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
