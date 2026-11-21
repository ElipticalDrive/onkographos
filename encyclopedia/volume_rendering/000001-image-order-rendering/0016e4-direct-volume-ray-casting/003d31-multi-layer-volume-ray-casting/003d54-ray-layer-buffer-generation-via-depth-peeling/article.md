# Ray-Layer Buffer Generation via Depth Peeling

Also called: VOI layer generation, peeling VOI definitions

## Overview
This method generates the ray-layer buffer for Multi-layer Volume Ray Casting by depth-peeling Volume of Interest (VOI) definitions and grouping the resulting fragments into ordered layers based on material ID and priority [S1]. It solves the problem of correctly determining the visibility and enclosure of nested, potentially self-intersecting VOIs on the GPU, where standard stencil buffers fail to preserve the critical priority and ID information required for per-layer transfer function application [S1]. Unlike standard GPU-based Constructive Solid Geometry (CSG) which uses hardware stencil buffers to count front/back fragments, this method performs similar logic inside a GPU fragment program to maintain full control over layer complexity and material assignment [S1].

## How it works
**Inputs:**
- A set of VOI definitions, each associated with a unique Material ID (mtId) and a priority value.
- Clipping meshes (optional, used to define view frustum or user-defined boundaries).
- Multi-Planar Reformation (MPR) meshes (optional, representing explicit planar data).
- A depth-peeling renderer that outputs fragments with depth, facing (front/back), and mesh ID.

**Outputs:**
- A ray-layer buffer: an array of entries per ray (pixel), where each entry contains a depth, a material ID, and potentially other metadata required for ray casting.

**Data Structures:**
- `insideFlag` array: An array indexed by Material ID. It stores the net count of front-facing minus back-facing fragments for each VOI.
- `rayLayerBuffer`: The output structure storing the sequence of layers for each ray.

**Algorithm Steps:**
1. **Peeling:** Render the scene using a depth-peeling algorithm [Eve01]. For each pass $k$, render only fragments with depth greater than the $k$-th peeled depth.
2. **Inside Flag Update:** For every fragment generated (even if discarded by the peeling depth test), update the `insideFlag` array for its associated Material ID:
   - If the fragment is front-facing, increment `insideFlag[mtId]`.
   - If the fragment is back-facing, decrement `insideFlag[mtId]`.
   This count represents the winding number or enclosure status of the fragment relative to the VOI [S1].
3. **Determine Current Material:** Scan the `insideFlag` array to find the Material ID of the enclosing VOI with the highest priority. Let this be `currentId`.
   - A VOI is considered enclosing if its `insideFlag` is non-zero (indicating the ray is inside the volume defined by that VOI).
   - Priority determines which VOI takes precedence if multiple VOIs enclose the same point [S1].
4. **Layer Buffer Writing:**
   - Compare `currentId` with the material ID of the previous layer in the `rayLayerBuffer` for the current ray.
   - If `currentId` differs from the previous layer's ID, write a new entry to the buffer with `currentId` and the current fragment's depth.
   - Note: A back-facing fragment indicates the exit of a contributing VOI. Consequently, `currentId` will change (e.g., from the VOI's ID to the background or a different enclosing VOI) [S1].
5. **Clipping Mesh Handling:**
   - If the fragment belongs to a clipping mesh:
     - If front-facing: Clear all previous layers in the `rayLayerBuffer` for this ray (resetting the state).
     - If back-facing: Mark layer generation as finished for this ray (no further layers are added for this ray).
6. **MPR Mesh Handling:**
   - If the fragment belongs to an MPR mesh:
     - Write the MPR layer directly to the buffer.
     - If the MPR is inside a non-empty layer (i.e., `currentId` is not background), append an entry with the same depth as the MPR but using the previous material ID. This ensures the intersected layer continues after the MPR [S1].
7. **Optimization (Empty Space Merging):**
   - After generating layers, scan for empty segments between adjacent `mtIdVolume` layers.
   - If the depth distance between two adjacent volume layers is shorter than a given threshold, merge them into a single layer to reduce buffer complexity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
insideFlags = {mtId: 0 for mtId in all_material_ids}
rayLayerBuffer = [] # List of layers for the current ray
previousLayerId = None

# Called for each peeled fragment in depth order
def process_fragment(fragment):
    global previousLayerId
    
    # 1. Update inside flags for the fragment's material ID
    mtId = fragment.mesh_id
    if fragment.is_front_facing:
        insideFlags[mtId] += 1
    else:
        insideFlags[mtId] -= 1

    # 2. Determine current enclosing material ID
    # Find mtId with non-zero insideFlag and highest priority
    currentId = find_highest_priority_enclosing_id(insideFlags)

    # 3. Handle Clipping Meshes
    if fragment.is_clipping_mesh:
        if fragment.is_front_facing:
            # Clear previous layers
            rayLayerBuffer.clear()
            previousLayerId = None
        else:
            # Mark generation as finished
            return # Stop processing further fragments for this ray
        
        return

    # 4. Handle MPR Meshes
    if fragment.is_mpr_mesh:
        # Write MPR layer
        add_layer_to_buffer(fragment.depth, fragment.mpr_material_id)
        
        # If inside a non-empty layer, append continuation entry
        if previousLayerId is not None and previousLayerId != background_id:
            add_layer_to_buffer(fragment.depth, previousLayerId)
        
        previousLayerId = fragment.mpr_material_id
        return

    # 5. Handle VOI Fragments
    # If currentId differs from previous layer, write new layer
    if currentId != previousLayerId:
        add_layer_to_buffer(fragment.depth, currentId)
        previousLayerId = currentId

def add_layer_to_buffer(depth, material_id):
    rayLayerBuffer.append({'depth': depth, 'material_id': material_id})

def find_highest_priority_enclosing_id(flags):
    best_id = background_id
    best_priority = -infinity
    for mtId, count in flags.items():
        if count != 0: # Enclosing
            priority = get_priority(mtId)
            if priority > best_priority:
                best_priority = priority
                best_id = mtId
    return best_id
```

**Edge Cases and Pitfalls:**
- **Identical Depth Fragments:** Standard peeling may output only one fragment when two fragments have identical depths, leading to incorrect `insideFlags` [S1]. This occurs when surface common points project exactly to pixel centers or when back-facing surfaces touch front-facing surfaces of the same VOI (common at bounding box corners) [S1].
- **Hardware Depth Test Limitations:** Modifying depth in a fragment program may disable hardware depth testing on some GPUs [S1].

## Subtopics

### Identical Depth Fragment Handling
A method to resolve ambiguity when two fragments at the same pixel have identical depths, which causes standard peeling to output only one and produce incorrect `insideFlags` [S1]. It encodes depth, facing, and mesh ID into a new number (similar to ray-layer buffer entry encoding) and sets it as the modified depth in the peeling fragment program [S1]. If hardware does not support depth testing with modified depth, the encoded value is written to a color channel and compared using MIN or MAX operations of the frame buffer [S1]. This issue is rare but common at corners/edges of non-empty-space bounding boxes where back-facing surfaces touch front-facing surfaces of the same VOI [S1].

## Operating characteristics
- **Time Complexity:** Proportional to the number of depth peeling passes ($N$) times the number of pixels, plus the cost of scanning `insideFlags` per fragment. The number of passes depends on the maximum number of overlapping VOIs [S1].
- **Memory Complexity:** Requires storage for `insideFlags` (size proportional to number of material IDs) and the ray-layer buffer (size proportional to number of layers per ray).
- **Strengths:** Correctly handles nested, self-intersecting VOIs and preserves material priority information which hardware stencil buffers cannot [S1]. Supports clipping and MPR meshes explicitly [S1].
- **Limitations:** Requires multiple rendering passes for depth peeling. Identical depth handling adds complexity and potential hardware compatibility issues [S1].
- **When to use:** When rendering complex scenes with nested, overlapping volumes where material priority and correct enclosure detection are critical, and standard CSG stencil methods are insufficient [S1].

## Background
The method originates from the work on Multi-layer Volume Ray Casting on GPU, specifically the layer generation phase [S1]. It was developed to address the limitations of standard GPU-based CSG stencil buffers in the context of volume rendering, where each VOI has a priority and ID critical for layer placement [S1]. The approach builds on depth peeling techniques [Eve01] and adapts them for volume data structures.

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
