# WASM Data Packing

Also called: DataWorker Packing, Splat Buffer Allocation

## Overview
WASM Data Packing manages WebAssembly memory allocation and data packing for 3D Gaussian splat attributes (positions, rotations, scales, colors, selection) to prepare them for GPU upload or WASM-side processing [S1, unofficial]. It differs from pure GPU-side allocation by handling the initial serialization of JavaScript/TypeScript data structures into a contiguous linear buffer within the WebAssembly heap, ensuring cache coherency and minimizing transfer overhead before the data reaches the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `Splat` object containing attribute arrays (positions, rotations, scales, colors, selection) and a `vertexCount`.
- **Output:** A contiguous block of memory in the WebAssembly heap (`HEAPF32` and `HEAPU8`) organized in a Structure-of-Arrays (SoA) layout, ready for direct memory access by WASM functions or GPU buffers.

**Data Structures:**
The `Splat` class fields include:
- `offset`: number
- `position`: `Float32Array[3]`
- `rotation`: `Float32Array[4]`
- `scale`: `Float32Array[3]`
- `selected`: boolean
- `vertexCount`: number
- `positions`: `Float32Array` (N x 3)
- `rotations`: `Float32Array` (N x 4)
- `scales`: `Float32Array` (N x 3)
- `colors`: `Uint8Array` (N x 4)
- `selection`: `Uint8Array` (N)

**Buffer Layout and Sizes:**
The packed memory is divided into specific regions based on `allocatedVertexCount` (the next power of 2 greater than or equal to `vertexCount`). Each float occupies 4 bytes.

1.  **positionsPtr**: Size = $3 \times \text{allocatedVertexCount} \times 4$ bytes.
2.  **rotationsPtr**: Size = $4 \times \text{allocatedVertexCount} \times 4$ bytes.
3.  **scalesPtr**: Size = $3 \times \text{allocatedVertexCount} \times 4$ bytes.
4.  **colorsPtr**: Size = $4 \times \text{allocatedVertexCount}$ bytes.
5.  **selectionPtr**: Size = $\text{allocatedVertexCount}$ bytes.
6.  **dataPtr**: Size = $8 \times \text{allocatedVertexCount} \times 4$ bytes.
7.  **worldPositionsPtr**: Size = $3 \times 4 \times 4 = 48$ bytes.
8.  **worldRotationsPtr**: Size = $4 \times 4 \times 4 = 64$ bytes.
9.  **worldScalesPtr**: Size = $3 \times 4 \times 4 = 48$ bytes.

**Numbered Steps:**
1.  **Initialization:** Call `initWasm()`. This loads the WASM module via `createDataModule()`. If the module is not yet loaded, `pack()` waits for loading to finish, initializes WASM, and proceeds [S1, unofficial].
2.  **Capacity Calculation:** Calculate `targetAllocatedVertexCount` as the next power of 2 greater than or equal to `splat.vertexCount`:
    $$ \text{targetAllocatedVertexCount} = 2^{\lceil \log_2(\text{splat.vertexCount}) \rceil} $$
    [S1, unofficial]
3.  **Memory Reallocation:** If `targetAllocatedVertexCount` is larger than the current `allocatedVertexCount`:
    a. Free old WASM buffers using `wasmModule._free()`.
    b. Update `allocatedVertexCount` to `targetAllocatedVertexCount`.
    c. Allocate new buffers using `wasmModule._malloc()` for each attribute region [S1, unofficial].
4.  **Data Copying:** Copy splat attribute arrays into the newly allocated WASM memory regions:
    a. Use `wasmModule.HEAPF32.set()` for float arrays (positions, rotations, scales).
    b. Use `wasmModule.HEAPU8.set()` for byte arrays (colors, selection) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack(splat):
    # Ensure WASM module is loaded
    if not wasm_module_loaded:
        init_wasm()
        wait_for_wasm_load()

    # Calculate next power of 2 capacity
    vertex_count = splat.vertex_count
    if vertex_count == 0:
        target_capacity = 1
    else:
        # Math.ceil(Math.log2(vertex_count))
        log2_val = math.log2(vertex_count)
        ceil_log2 = math.ceil(log2_val)
        target_capacity = 2 ** ceil_log2

    # Check if reallocation is needed
    if target_capacity > allocated_vertex_count:
        # Free old buffers
        if allocated_vertex_count > 0:
            wasm_module._free(positions_ptr)
            wasm_module._free(rotations_ptr)
            wasm_module._free(scales_ptr)
            wasm_module._free(colors_ptr)
            wasm_module._free(selection_ptr)
            wasm_module._free(data_ptr)
            wasm_module._free(world_positions_ptr)
            wasm_module._free(world_rotations_ptr)
            wasm_module._free(world_scales_ptr)

        # Update capacity
        allocated_vertex_count = target_capacity

        # Allocate new buffers
        positions_ptr = wasm_module._malloc(3 * allocated_vertex_count * 4)
        rotations_ptr = wasm_module._malloc(4 * allocated_vertex_count * 4)
        scales_ptr = wasm_module._malloc(3 * allocated_vertex_count * 4)
        colors_ptr = wasm_module._malloc(4 * allocated_vertex_count)
        selection_ptr = wasm_module._malloc(allocated_vertex_count)
        data_ptr = wasm_module._malloc(8 * allocated_vertex_count * 4)
        world_positions_ptr = wasm_module._malloc(48)
        world_rotations_ptr = wasm_module._malloc(64)
        world_scales_ptr = wasm_module._malloc(48)

    # Copy data into WASM heap
    # Positions: Float32Array
    wasm_module.HEAPF32.set(splat.positions, positions_ptr // 4)
    
    # Rotations: Float32Array
    wasm_module.HEAPF32.set(splat.rotations, rotations_ptr // 4)
    
    # Scales: Float32Array
    wasm_module.HEAPF32.set(splat.scales, scales_ptr // 4)
    
    # Colors: Uint8Array
    wasm_module.HEAPU8.set(splat.colors, colors_ptr)
    
    # Selection: Uint8Array
    wasm_module.HEAPU8.set(splat.selection, selection_ptr)

    # Note: world_* pointers are allocated but their population logic is not specified in the source [G]
    # Typically these would be populated with identity or current camera transforms if needed by WASM
```

**Parameters:**
- `positionsPtr`: allocated size = $3 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
- `rotationsPtr`: allocated size = $4 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
- `scalesPtr`: allocated size = $3 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
- `colorsPtr`: allocated size = $4 \times \text{allocatedVertexCount}$ bytes [S1, unofficial].
- `selectionPtr`: allocated size = $\text{allocatedVertexCount}$ bytes [S1, unofficial].
- `dataPtr`: allocated size = $8 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
- `worldPositionsPtr`: allocated size = 48 bytes [S1, unofficial].
- `worldRotationsPtr`: allocated size = 64 bytes [S1, unofficial].
- `worldScalesPtr`: allocated size = 48 bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Concurrent Calls:** Concurrent calls to `pack()` are serialized via a loading flag and a `while` loop with `setTimeout` to yield to the event loop, preventing race conditions during WASM initialization [S1, unofficial].
- **Memory Efficiency:** Memory is only freed and reallocated when the next power-of-2 capacity exceeds the current `allocatedVertexCount`, preventing frequent reallocations for incremental vertex count changes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Allocation is $O(1)$ amortized due to power-of-2 growth, but reallocation is $O(N)$ where $N$ is the number of vertices. Copying is $O(N)$.
- **Memory Complexity:** $O(N)$, where $N$ is the next power of 2 greater than or equal to the number of splats.
- **Strengths:** Reduces JavaScript-to-GPU transfer overhead by pre-packaging data in a contiguous WASM heap. Power-of-2 allocation minimizes fragmentation and reallocation frequency.
- **Limitations:** Requires WebAssembly environment. Power-of-2 allocation can lead to memory waste if the vertex count is not a power of 2.
- **When to use:** When rendering 3D Gaussian Splats in a WebAssembly-based environment where data needs to be prepared for GPU upload or WASM-side processing.
- **When not to use:** In pure CPU-based rendering or environments without WebAssembly support.

## Background
WASM Data Packing is part of the `gsplat.js` library, which provides JavaScript/TypeScript bindings for 3D Gaussian Splatting. It was developed to enable efficient rendering of Gaussian Splats in web browsers using WebAssembly [S1, unofficial].

## Sources
[S1] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
