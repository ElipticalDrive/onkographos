# Ray-Layer Buffer Generation via Depth Peeling

Also called: VOI layer generation, peeling VOI definitions

## Overview
This method generates the ray-layer buffer required by Multi-layer Volume Ray Casting by depth-peeling Volume of Interest (VOI) definitions and grouping the resulting fragments into ordered layers based on VOI priority and material ID [S1]. It differs from standard GPU Constructive Solid Geometry (CSG) techniques by performing stencil-like containment tests inside a custom GPU program rather than using hardware stencil buffers, thereby preserving critical priority and ID information needed for correct layer placement and optimization [S1].

## How it works
**Inputs:**
- A set of depth-peeled fragments from all VOIs, clipping meshes, and Multi-Planar Reformation (MPR) meshes [S1].
- Each fragment carries its depth, facing direction (front/back), and associated VOI metadata (Material ID, Priority) [S1].

**Outputs:**
- A ray-layer buffer containing ordered entries for each ray, representing the sequence of volumes and surfaces encountered [S1].

**Data Structures:**
- `insideFlag`: An array indexed by material ID. It stores an integer count representing the difference between the number of front-facing and back-facing fragments for each material ID [S1].
- `rayLayerBuffer`: A list of layer entries per ray. Each entry contains depth and material ID [S1].

**Algorithm Steps:**
The generation is performed per ray in a GPU program [S1].

1. **Initialize:** Reset `insideFlag` for all material IDs to zero for the current ray [G]. Initialize `rayLayerBuffer` as empty [G].

2. **Process Fragments:** Iterate through the depth-peeled fragments for the current ray in depth order (front to back) [S1]. For each fragment:
    a. **Update Inside Flags:** Increment `insideFlag[fragment.materialID]` if the fragment is front-facing; decrement it if back-facing [S1]. This update occurs regardless of whether the fragment is ultimately discarded or written to the buffer [S1].
    b. **Determine Enclosure:** Scan the `insideFlag` array to find the material ID of the enclosing VOI with the highest priority. Let this be `currentId` [S1].
       - *Note:* A positive `insideFlag` value indicates the ray is inside that VOI. The "highest priority" logic resolves overlaps where multiple VOIs might have non-zero flags [S1].
    c. **Handle Clipping Meshes:**
       - If the fragment is a clipping mesh and is front-facing: Clear all previous entries in `rayLayerBuffer` [S1].
       - If the fragment is a clipping mesh and is back-facing: Mark layer generation as finished for the current ray [S1]. Stop processing further fragments for this ray.
    d. **Handle MPR Meshes:**
       - If the fragment is an MPR mesh: Write it directly to the `rayLayerBuffer` [S1].
       - *Special Case:* If the MPR is currently inside a non-empty layer (i.e., `rayLayerBuffer` is not empty), append an additional entry with the same depth as the MPR but using the material ID of the previous layer entry. This ensures the intersected layer continues past the MPR [S1].
    e. **Write Volume Layers:**
       - If the fragment is a VOI fragment: Check if `currentId` differs from the material ID of the last entry in `rayLayerBuffer` (or if the buffer is empty) [S1].
       - If they differ, write the current fragment to the `rayLayerBuffer` with `currentId` as its material ID [S1].
       - *Note:* A back-facing fragment indicates the exit of a contributing VOI. Consequently, `currentId` will typically differ from the ID of the VOI being exited, triggering a layer transition or termination [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_ray_layers(peeled_fragments, clipping_meshes, mpr_meshes, voi_metadata):
    """
    Generates the ray-layer buffer for a single ray.
    peeled_fragments: List of (depth, facing, material_id, is_voi, is_clipping, is_mpr)
    voi_metadata: Dict mapping material_id to priority
    """
    inside_flags = {} # Map material_id -> int count
    ray_layer_buffer = []
    layer_generation_finished = False

    # Initialize flags for all known material IDs [G]
    for mid in voi_metadata.keys():
        inside_flags[mid] = 0

    # Sort fragments by depth (front to back) [G]
    sorted_fragments = sorted(peeled_fragments + clipping_meshes + mpr_meshes, key=lambda x: x.depth)

    for frag in sorted_fragments:
        if layer_generation_finished:
            break

        mid = frag.material_id

        # Step 2a: Update inside flags
        if frag.facing == 'front':
            inside_flags[mid] = inside_flags.get(mid, 0) + 1
        elif frag.facing == 'back':
            inside_flags[mid] = inside_flags.get(mid, 0) - 1

        # Step 2b: Determine current enclosing VOI with highest priority
        current_id = None
        max_priority = -1
        
        # Scan insideFlags to find enclosing VOI with highest priority [S1]
        # Only consider IDs with positive inside flags as "enclosing"
        for candidate_id, count in inside_flags.items():
            if count > 0:
                priority = voi_metadata.get(candidate_id, 0)
                if priority > max_priority:
                    max_priority = priority
                    current_id = candidate_id
        
        # If no VOI is enclosing, current_id remains None

        # Step 2c: Handle Clipping Meshes
        if frag.is_clipping:
            if frag.facing == 'front':
                # Clear all previous layers [S1]
                ray_layer_buffer.clear()
            elif frag.facing == 'back':
                # Mark layer generation as finished [S1]
                layer_generation_finished = True
            continue

        # Step 2d: Handle MPR Meshes
        if frag.is_mpr:
            # Write MPR out directly [S1]
            ray_layer_buffer.append({'depth': frag.depth, 'material_id': mid})
            
            # If MPR is inside a non-empty layer, append entry with previous material ID [S1]
            if len(ray_layer_buffer) > 1:
                prev_material_id = ray_layer_buffer[-2]['material_id']
                ray_layer_buffer.append({'depth': frag.depth, 'material_id': prev_material_id})
            continue

        # Step 2e: Handle VOI Fragments
        if frag.is_voi:
            # If currentId differs from previous layer's material ID, write fragment [S1]
            if not ray_layer_buffer:
                if current_id is not None:
                    ray_layer_buffer.append({'depth': frag.depth, 'material_id': current_id})
            else:
                prev_material_id = ray_layer_buffer[-1]['material_id']
                if current_id != prev_material_id:
                    # If current_id is None, it might mean we exited all volumes.
                    # The source implies writing with currentId. If currentId is None,
                    # this step might be skipped or handled as empty space.
                    # Based on "If currentId is different... write... with currentId",
                    # we assume currentId is valid or the condition prevents writing.
                    if current_id is not None:
                        ray_layer_buffer.append({'depth': frag.depth, 'material_id': current_id})
```

**Parameters:**
- VOI Priority: Determines the order of precedence when multiple VOIs enclose the same point [S1].
- Material ID: Identifies the specific volume or surface type for transfer function application [S1].

**Edge Cases and Pitfalls:**
- **Standard Stencil Buffer Limitations:** Standard GPU stencil buffers cannot easily handle the requirement to track both priority and ID for each VOI, nor can they easily support the optimizations required to reduce layer complexity [S1].
- **MPR Intersection:** If an MPR intersects a non-empty layer, the layer must continue with the previous material ID after the MPR depth to maintain correct volume rendering semantics [S1].
- **Clipping:** Front-facing clipping meshes reset the layer history, while back-facing ones terminate it. Failure to handle these correctly leads to incorrect visibility [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of depth-peeled fragments per ray, as each fragment is processed once to update flags and potentially write to the buffer [S1].
- **Memory Complexity:** Proportional to the number of material IDs for the `insideFlag` array and the number of layers in the `rayLayerBuffer` [S1].
- **Strengths:** Allows for complex VOI hierarchies with priorities and supports optimizations like bounding box hulls to reduce layer complexity [S1].
- **Limitations:** Requires custom GPU programming for the stencil-like logic, as hardware stencil buffers are insufficient [S1].

## Background
This method was introduced in the context of Multi-layer Volume Ray Casting on GPU [S1]. It addresses the limitations of standard GPU-based CSG techniques by integrating layer generation directly into the ray casting pipeline, enabling efficient handling of multiple overlapping volumes with distinct properties [S1].

## Sources
[S1] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
