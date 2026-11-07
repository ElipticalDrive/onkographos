# Streaming Binary Greedy Meshing

Also called: SBUG streaming step, Streaming Binary Greedy Meshing (SBUG) Core Loop

## Overview
Streaming Binary Greedy Meshing (SBGM) is a performance-oriented variant of Binary Greedy Meshing that combines face visibility generation and greedy merging into a single streaming traversal [S1, unofficial]. It retains only two rolling face data slices instead of six full chunk face arrays, significantly improving cache locality by ensuring data is consumed immediately after generation [S1, unofficial].

## How it works
SBGM processes a voxel chunk one x-slice at a time. It maintains two `FaceSlice` buffers (current and next) and emits quads for all six face directions as compatible runs terminate [S1, unofficial].

**Inputs and Outputs:**
- **Input:** `voxels` (`uint8_t*`), `opaque` (`uint64_t*`), `MeshData& md`.
- **Output:** `md.vertices` (array of 6 `uint64_t*` buffers), `md.face_vertex_counts` (array of 6 `size_t`), `md.vertex_count` (`size_t`).

**Data Structures:**
- **Chunk Dimensions:** Chunk Size `CS = 62`, Padded Size `CS_P = 64`, Padded Area `CS_P2 = 4096` [S1, unofficial].
- **Mask:** `P_MASK = ~(1ull << 63 | 1ull)` excludes top and bottom bits of a 64-bit integer [S1, unofficial].
- **FaceSlice:**
  ```cpp
  struct FaceSlice {
      uint64_t x_neg[CS_P];
      uint64_t x_pos[CS_P];
      uint64_t z_neg[CS_P];
      uint64_t z_pos[CS_P];
      uint64_t y_neg[CS_P];
      uint64_t y_pos[CS_P];
      uint8_t y_neg_span[CS_P2];
      uint8_t y_pos_span[CS_P2];
  };
  ```
- **MeshData:** Contains merge state arrays (`forward_merged`, `pos_forward_merged`, `y_right_merged`, etc.) and vertex buffers. Offsets are defined by `static_assert` constants (e.g., `MERGED_POS_FORWARD`) [S1, unofficial].
- **Outputs:** Array of 6 pointers to `uint64_t`, one for each face direction (0: X+, 1: X-, 2: Z+, 3: Z-, 4: Y+, 5: Y-) [S1, unofficial].

**Core Loop:**
1. Initialize two `FaceSlice` buffers. Build slices for x=1 and x=2.
2. Iterate x from 1 to CS.
3. For each x, determine the `right` neighbor slice: if `x < CS`, `right = next`; otherwise `right = &zero_slice` [S1, unofficial].
4. For each z from 1 to CS:
   - Calculate `slice_voxels = voxels + x * CS_P2` and `col = slice_voxels + z * CS_P` [S1, unofficial].
   - Calculate `span_base = z * CS_P` [S1, unofficial].
   - If `current->x_neg[z]` is non-zero, call `mesh_x_bits` with `col`, `-1`, `x`, `z`, `current->x_neg[z]`, `current->x_neg[z+1]`, `merged + MERGED_FORWARD`, and `outputs[1]` [S1, unofficial].
   - If `current->x_pos[z]` is non-zero, call `mesh_x_bits` with `col`, `+1`, `x`, `z`, `current->x_pos[z]`, `current->x_pos[z+1]`, `merged + MERGED_POS_FORWARD`, and `outputs[0]` [S1, unofficial].
   - If `current->y_neg[z]` is non-zero, call `mesh_y_end_bits_x_major` with `col`, `-1`, `x`, `z`, `current->y_neg[z]`, `right->y_neg[z]`, `current->y_neg_span + span_base`, `right->y_neg_span + span_base`, `merged + MERGED_Y_RIGHT + span_base`, and `outputs[5]` [S1, unofficial].
   - If `current->y_pos[z]` is non-zero, call `mesh_y_end_bits_x_major` with `col`, `+1`, `x`, `z`, `current->y_pos[z]`, `right->y_pos[z]`, `current->y_pos_span + span_base`, `right->y_pos_span + span_base`, `merged + MERGED_Y_POS_RIGHT + span_base`, and `outputs[4]` [S1, unofficial].
   - If `current->z_neg[z]` is non-zero, call `mesh_z_bits` with `col`, `-1`, `z`, `x`, `current->z_neg[z]`, `right->z_neg[z]`, `merged + MERGED_Z_FORWARD + span_base`, and `outputs[3]` [S1, unofficial].
   - If `current->z_pos[z]` is non-zero, call `mesh_z_bits` with `col`, `+1`, `z`, `x`, `current->z_pos[z]`, `right->z_pos[z]`, `merged + MERGED_Z_POS_FORWARD + span_base`, and `outputs[2]` [S1, unofficial].
5. Swap `current` and `next` pointers [S1, unofficial].
6. If `x + 2 <= CS`, call `build_face_slice(voxels, opaque, x + 2, *next)` to prepare the next slice's data [S1, unofficial].

**Helper Functions:**
- **`build_face_slice`:** Loads column occupancy bits for left, mid, and right slices using AVX2 (`_mm256_loadu_si256`), computes `x_neg` and `x_pos` by masking with `P_MASK` and neighbor occupancy, stores results in `slice.x_neg/z` and `slice.x_pos/z`, and sets `slice.x_neg[CS+1]` and `slice.x_pos[CS+1]` to 0 as sentinels [S1, unofficial]. Calls `build_y_face_slice` [S1, unofficial].
- **`build_y_face_slice`:** Iterates z from 1 to CS, computing `neg_current` and `pos_current` bits from column occupancy and neighbors, calls `build_y_face_column` for each z, updates `prev_occ/current_occ/next_occ`, and handles z=CS sentinel by setting `neg_next/pos_next` to 0 [S1, unofficial].
- **`build_y_face_column`:** Processes `neg_bits` and `pos_bits` separately: for each bit y, checks if neighbor has same material; if so, increments forward span and removes bit; else stores forward span to `span_end`, resets forward span to 0, and updates terminal bits [S1, unofficial].
- **`mesh_x_bits`:** Iterates `bits_here`: finds lowest set bit y, checks if `bits_next` has same bit and same material; if so, increments `forward_span` and continues; else computes height by comparing `bits_here` and `bits_next`, merges forward spans, creates quad via `get_quad`, and removes processed bits [S1, unofficial].
- **`mesh_z_bits`:** Iterates `bits_here`: finds lowest set bit y, checks if `bits_next` has same bit and same material; if so, increments `forward_span` and continues; else computes height by comparing `bits_here` and `bits_next`, merges forward spans, creates quad via `get_quad`, and removes processed bits [S1, unofficial].
- **`mesh_y_end_bits_x_major`:** Iterates `bits_here`: finds bit y, checks if `bits_right` has same bit, `forward_span` equals `span_right[y]`, and same material; if so, increments `right_span`; else computes `start_z/start_x/length_z/width_x` from spans, creates quad via `get_quad`, resets `right_span`, and removes processed bits [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh(voxels, opaque, md):
    slices = [FaceSlice(), FaceSlice()]
    zero_slice = FaceSlice()
    current = slices[0]
    next_slice = slices[1]
    
    build_face_slice(voxels, opaque, 1, current)
    build_face_slice(voxels, opaque, 2, next_slice)
    
    outputs = [md.vertices[i] for i in range(6)]
    begins = [outputs[i] for i in range(6)]
    
    for x in range(1, CS + 1):
        if x < CS:
            right = next_slice
        else:
            right = zero_slice
            
        for z in range(1, CS + 1):
            col = voxels + x * CS_P2 + z * CS_P
            span_base = z * CS_P
            
            if current.x_neg[z]:
                outputs[1] = mesh_x_bits(col, -1, x, z, current.x_neg[z], current.x_neg[z+1], md.forward_merged + MERGED_FORWARD, outputs[1])
            if current.x_pos[z]:
                outputs[0] = mesh_x_bits(col, 1, x, z, current.x_pos[z], current.x_pos[z+1], md.pos_forward_merged + MERGED_POS_FORWARD, outputs[0])
            if current.y_neg[z]:
                outputs[5] = mesh_y_end_bits_x_major(col, -1, x, z, current.y_neg[z], right.y_neg[z], current.y_neg_span + span_base, right.y_neg_span + span_base, md.y_right_merged + MERGED_Y_RIGHT + span_base, outputs[5])
            if current.y_pos[z]:
                outputs[4] = mesh_y_end_bits_x_major(col, 1, x, z, current.y_pos[z], right.y_pos[z], current.y_pos_span + span_base, right.y_pos_span + span_base, md.y_pos_right_merged + MERGED_Y_POS_RIGHT + span_base, outputs[4])
            if current.z_neg[z]:
                outputs[3] = mesh_z_bits(col, -1, z, x, current.z_neg[z], right.z_neg[z], md.z_forward_merged + MERGED_Z_FORWARD + span_base, outputs[3])
            if current.z_pos[z]:
                outputs[2] = mesh_z_bits(col, 1, z, x, current.z_pos[z], right.z_pos[z], md.z_pos_forward_merged + MERGED_Z_POS_FORWARD + span_base, outputs[2])
                
        tmp = current
        current = next_slice
        next_slice = tmp
        
        if x + 2 <= CS:
            build_face_slice(voxels, opaque, x + 2, next_slice)
            
    for face in range(6):
        md.face_vertex_counts[face] = outputs[face] - begins[face]
        md.vertex_count += md.face_vertex_counts[face]
        
    return md.vertex_count

def build_face_slice(voxels, opaque, x, slice):
    left = opaque + (x - 1) * CS_P
    mid = opaque + x * CS_P
    right = opaque + (x + 1) * CS_P
    p_mask = _mm256_set1_epi64x(P_MASK)
    
    for z in range(0, CS_P, 4):
        occ = _mm256_loadu_si256(mid + z)
        column_bits = _mm256_and_si256(occ, p_mask)
        left_occ = _mm256_loadu_si256(left + z)
        right_occ = _mm256_loadu_si256(right + z)
        _mm256_storeu_si256(slice.x_neg + z, _mm256_andnot_si256(left_occ, column_bits))
        _mm256_storeu_si256(slice.x_pos + z, _mm256_andnot_si256(right_occ, column_bits))
        
    slice.x_neg[CS + 1] = 0
    slice.x_pos[CS + 1] = 0
    
    build_y_face_slice(voxels, opaque, x, slice.y_neg, slice.y_pos, slice.y_neg_span, slice.y_pos_span, slice.z_neg, slice.z_pos)

def build_y_face_slice(voxels, opaque, x, neg_end, pos_end, neg_span_end, pos_span_end, z_neg, z_pos):
    neg_forward = [0] * CS_P
    pos_forward = [0] * CS_P
    
    prev_occ = column_mask(opaque, x, 0)
    current_occ = column_mask(opaque, x, 1)
    column_bits = current_occ & P_MASK
    neg_current = column_bits & ~(current_occ << 1)
    pos_current = column_bits & ~(current_occ >> 1)
    col = voxels + x * CS_P2 + CS_P
    
    for z in range(1, CS):
        next_occ = column_mask(opaque, x, z + 1)
        next_bits = next_occ & P_MASK
        neg_next = next_bits & ~(next_occ << 1)
        pos_next = next_bits & ~(next_occ >> 1)
        
        build_y_face_column(col, z * CS_P, neg_current, pos_current, neg_next, pos_next, neg_forward, pos_forward, neg_end + z, pos_end + z, neg_span_end, pos_span_end)
        
        prev_occ = current_occ
        current_occ = next_occ
        column_bits = next_bits
        neg_current = neg_next
        pos_current = pos_next
        col += CS_P
        
    z_neg[CS] = column_bits & ~prev_occ
    z_pos[CS] = column_bits & ~column_mask(opaque, x, CS + 1)
    build_y_face_column(col, CS * CS_P, neg_current, pos_current, 0, 0, neg_forward, pos_forward, neg_end + CS, pos_end + CS, neg_span_end, pos_span_end)

def build_y_face_column(col, span_base, neg_current, pos_current, neg_next, pos_next, neg_forward, pos_forward, neg_end, pos_end, neg_span_end, pos_span_end):
    neg_bits = neg_current
    while neg_bits:
        y = _tzcnt_u64(neg_bits)
        y_mask = _blsi_u64(neg_bits)
        neg_bits &= neg_bits - 1
        
        material = col[y]
        if (neg_next & y_mask) != 0 and material == col[y + CS_P]:
            neg_forward[y] += 1
        else:
            neg_span_end[span_base + y] = neg_forward[y]
            neg_forward[y] = 0
            
    pos_bits = pos_current
    while pos_bits:
        y = _tzcnt_u64(pos_bits)
        y_mask = _blsi_u64(pos_bits)
        pos_bits &= pos_bits - 1
        
        material = col[y]
        if (pos_next & y_mask) != 0 and material == col[y + CS_P]:
            pos_forward[y] += 1
        else:
            pos_span_end[span_base + y] = pos_forward[y]
            pos_forward[y] = 0
            
    neg_end[0] = neg_current & ~(neg_next | (neg_next << 1) | (neg_next >> 1)) # Sentinel logic simplified
    pos_end[0] = pos_current & ~(pos_next | (pos_next << 1) | (pos_next >> 1)) # Sentinel logic simplified

def mesh_x_bits(col, sign, x, z, bits_here, bits_next, forward_merged, out):
    while bits_here:
        y = _tzcnt_u64(bits_here)
        y_mask = _blsi_u64(bits_here)
        material = col[y]
        forward_span = forward_merged[y]
        
        if (bits_next & y_mask) != 0 and material == col[y + CS_P]:
            forward_span += 1
            bits_here &= ~y_mask
            continue
            
        height_bits = (bits_here & ~bits_next) >> y
        max_height = _tzcnt_u64(~height_bits)
        height = 1
        while height < max_height:
            next_y = y + height
            if forward_span != forward_merged[next_y]:
                break
            if material != col[next_y]:
                break
            forward_merged[next_y] = 0
            height += 1
            
        quad_mask = _bzhi_u64(~0, height) << y
        bits_here &= ~quad_mask
        
        width = forward_span + 1
        start_z = z - forward_span
        forward_span = 0
        
        if sign > 0:
            out[0] = get_quad(z, start_z - 1 + width, y - 1, width, height, material)
        else:
            out[0] = get_quad(z - 1, start_z - 1, y - 1, width, height, material)
        out += 1
        
    return out

def mesh_z_bits(col, sign, z, x, bits_here, bits_next, forward_merged, out):
    while bits_here:
        y = _tzcnt_u64(bits_here)
        y_mask = _blsi_u64(bits_here)
        material = col[y]
        forward_span = forward_merged[y]
        
        if (bits_next & y_mask) != 0 and material == col[y + CS_P2]:
            forward_span += 1
            bits_here &= ~y_mask
            continue
            
        height_bits = (bits_here & ~bits_next) >> y
        max_height = _tzcnt_u64(~height_bits)
        height = 1
        while height < max_height:
            next_y = y + height
            if forward_span != forward_merged[next_y]:
                break
            if material != col[next_y]:
                break
            forward_merged[next_y] = 0
            height += 1
            
        quad_mask = _bzhi_u64(~0, height) << y
        bits_here &= ~quad_mask
        
        width = forward_span + 1
        start_x = x - forward_span
        forward_span = 0
        
        if sign > 0:
            out[0] = get_quad(z, start_x - 1 + width, y - 1, width, height, material)
        else:
            out[0] = get_quad(z - 1, start_x - 1, y - 1, width, height, material)
        out += 1
        
    return out

def mesh_y_end_bits_x_major(col, sign, x, z, bits_here, bits_right, span_here, span_right, right_merged, out):
    while bits_here:
        y = _tzcnt_u64(bits_here)
        y_mask = 1 << y
        bits_here &= bits_here - 1
        
        material = col[y]
        forward_span = span_here[y]
        right_span = right_merged[y]
        
        if (bits_right & y_mask) != 0 and forward_span == span_right[y] and material == col[y + CS_P2]:
            right_span += 1
            continue
            
        start_z = z - forward_span
        start_x = x - right_span
        length_z = forward_span + 1
        width_x = right_span + 1
        right_span = 0
        
        if sign > 0:
            out[0] = get_quad(start_z - 1 + length_z, start_x - 1, y, length_z, width_x, material)
        else:
            out[0] = get_quad(start_z - 1, start_x - 1, y - 1, length_z, width_x, material)
        out += 1
        
    return out

def get_quad(x, y, z, w, h, type):
    return (type << 32) | (h << 24) | (w << 18) | (z << 12) | (y << 6) | x

def column_mask(opaque, x, z):
    return opaque[z + x * CS_P]
```

**Edge Cases and Pitfalls:**
- **Sentinels:** `z=CS` is padding, so y spans must terminate here by setting `neg_next/pos_next` to 0 in `build_y_face_column` [S1, unofficial]. `z=CS+1` sentinel: `slice.x_neg[CS+1]` and `slice.x_pos[CS+1]` are set to 0 to stop forward merges at the last interior column [S1, unofficial].
- **Quad Counts:** Quad counts may differ between SBGM and BGM because SBGM merges adjacent faces in a different order, though the benchmark validator confirms functionally equivalent coverage [S1, unofficial].

## Subtopics

### [Binary Greedy Voxel Meshing with Ambient Occlusion](4465-binary-greedy-voxel-meshing-with-ambient-occlusion.md)
Extends SBGM by integrating a 4-state vertex model that bakes occlusion intensity directly into vertex attributes, distinguishing it from sibling methods that output uniform `ao=0` for all opaque blocks.

## Operating characteristics
- **Time Complexity:** O(V) where V is the number of voxels, but with significantly reduced constant factors due to cache efficiency.
- **Memory Complexity:** O(CS_P^2) for the two rolling slices, compared to O(CS_P^3) for full chunk masks in BGM.
- **Performance:** On a Ryzen 5 5600x with Windows 10, MSVC 19.44.35221.0, on a 64x64 grid at 8 iterations, SBGM achieves 37.52 us/chunk median throughput (26,650 chunks/s) compared to BGM's 100.27 us/chunk, a 2.67x speedup [S1, unofficial].
- **Strengths:** Improved cache locality by reducing exposed face data to two 11 KiB slices that are immediately consumed, avoiding L1 cache evictions [S1, unofficial].
- **Limitations:** Requires AVX2/BMI/BMI2 support [S1, unofficial]. Quad counts may differ from BGM due to merge order [S1, unofficial].

## Background
SBGM is a performance-oriented variant of Binary Greedy Meshing (BGM) [S1, unofficial]. Traditional BGM builds six full-chunk face masks (180 KiB total) which do not fit in L1 data cache (32/48 KiB) and get evicted before meshing [S1, unofficial]. SBGM traverses a chunk one x-slice at a time, reducing exposed face data to two 11 KiB slices that are immediately consumed for meshing, improving cache locality [S1, unofficial].

## Sources
[S1] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
