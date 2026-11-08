# WASM Data Packing

Also called: DataWorker Packing, Splat Buffer Allocation

## Overview
WASM Data Packing manages WebAssembly memory allocation and data packing for 3D Gaussian splat attributes (positions, rotations, scales, colors, selection) to prepare them for GPU upload or WASM-side processing [S1, unofficial]. It differs from standard CPU-side memory management by handling the specific binary layout requirements of the WebAssembly heap, ensuring that contiguous JavaScript typed arrays are correctly mapped to the linear memory regions expected by the compiled C/C++ rendering kernels [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `Splat` object containing attribute arrays (positions, rotations, scales, colors, selection) and a `vertexCount` representing the number of active Gaussians.
*   **Output:** Updated WebAssembly linear memory regions containing packed attribute data, with pointers returned or stored for subsequent rendering passes.

**Data Structures:**
The `Splat` class maintains the following fields [S1, unofficial]:
*   `offset`: number
*   `position`: `Float32Array` of size 3
*   `rotation`: `Float32Array` of size 4
*   `scale`: `Float32Array` of size 3
*   `selected`: boolean
*   `vertexCount`: number
*   `positions`: `Float32Array` (contiguous buffer for all splats)
*   `rotations`: `Float32Array` (contiguous buffer for all splats)
*   `scales`: `Float32Array` (contiguous buffer for all splats)
*   `colors`: `Uint8Array` (contiguous buffer for all splats)
*   `selection`: `Uint8Array` (contiguous buffer for all splats)

**Memory Allocation Layout:**
The packing process allocates specific contiguous regions in the WASM heap based on the `allocatedVertexCount` (the next power of 2 greater than or equal to `vertexCount`) [S1, unofficial]. The sizes are calculated as follows:

1.  **Positions:** `positionsPtr` allocated size = $3 \times \text{allocatedVertexCount} \times 4$ bytes.
2.  **Rotations:** `rotationsPtr` allocated size = $4 \times \text{allocatedVertexCount} \times 4$ bytes.
3.  **Scales:** `scalesPtr` allocated size = $3 \times \text{allocatedVertexCount} \times 4$ bytes.
4.  **Colors:** `colorsPtr` allocated size = $4 \times \text{allocatedVertexCount}$ bytes.
5.  **Selection:** `selectionPtr` allocated size = $\text{allocatedVertexCount}$ bytes.
6.  **Data:** `dataPtr` allocated size = $8 \times \text{allocatedVertexCount} \times 4$ bytes.
7.  **World Transforms:**
    *   `worldPositionsPtr`: $3 \times 4 \times 4$ bytes.
    *   `worldRotationsPtr`: $4 \times 4 \times 4$ bytes.
    *   `worldScalesPtr`: $3 \times 4 \times 4$ bytes.

**Numbered Steps:**
1.  **Initialization:** Call `initWasm()`. This loads the WASM module via `createDataModule()`. If the module is not yet loaded, the `pack()` function waits for loading to finish, initializes the WASM environment, and then proceeds [S1, unofficial].
2.  **Capacity Calculation:** Calculate `targetAllocatedVertexCount` as the smallest power of 2 greater than or equal to `splat.vertexCount`. This is computed using:
    $$ \text{targetAllocatedVertexCount} = 2^{\lceil \log_2(\text{splat.vertexCount}) \rceil} $$
    [S1, unofficial].
3.  **Reallocation Check:** If `targetAllocatedVertexCount` is greater than the current `allocatedVertexCount`:
    a. Free old WASM buffers using `wasmModule._free()` [S1, unofficial].
    b. Update `allocatedVertexCount` to `targetAllocatedVertexCount`.
    c. Allocate new buffers using `wasmModule._malloc()` for each attribute pointer defined in the layout above [S1, unofficial].
4.  **Data Copying:** Copy the attribute arrays from the JavaScript `Splat` object into the newly allocated WASM memory regions:
    a. Use `wasmModule.HEAPF32.set()` for float arrays (`positions`, `rotations`, `scales`) [S1, unofficial].
    b. Use `wasmModule.HEAPU8.set()` for byte arrays (`colors`, `selection`) [S1, unofficial].
5.  **Concurrency Handling:** Ensure concurrent calls to `pack()` are serialized. This is achieved via a loading flag and a `while` loop with `setTimeout` to yield to the event loop, preventing race conditions during WASM initialization and memory reallocation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class WASMDataPacker:
    def __init__(self, wasm_module):
        self.wasm_module = wasm_module
        self.allocated_vertex_count = 0
        self.is_loading = False
        self.loading_flag = False

    def init_wasm(self):
        """Loads the WASM module via createDataModule()."""
        # Implementation details of createDataModule() are external to this method
        # but must be called to ensure wasm_module is valid.
        pass

    def pack(self, splat):
        """
        Packs splat attributes into WASM memory.
        :param splat: Object containing vertexCount, positions, rotations, scales, colors, selection.
        """
        # Concurrency guard
        while self.loading_flag:
            # Yield to event loop to prevent race conditions
            await new Promise(resolve => setTimeout(resolve, 0))
        
        if not self.is_loading:
            self.is_loading = True
            self.init_wasm()
            self.is_loading = False

        vertex_count = splat.vertexCount
        
        # Calculate next power of 2
        if vertex_count <= 0:
            target_alloc = 0
        else:
            # Math.ceil(Math.log2(vertex_count))
            log2_val = 0
            temp = vertex_count - 1
            while temp > 0:
                temp >>= 1
                log2_val += 1
            target_alloc = 1 << log2_val

        # Check if reallocation is needed
        if target_alloc > self.allocated_vertex_count:
            # Free old buffers
            if self.allocated_vertex_count > 0:
                self.wasm_module._free(self.positions_ptr)
                self.wasm_module._free(self.rotations_ptr)
                self.wasm_module._free(self.scales_ptr)
                self.wasm_module._free(self.colors_ptr)
                self.wasm_module._free(self.selection_ptr)
                self.wasm_module._free(self.data_ptr)
                self.wasm_module._free(self.world_positions_ptr)
                self.wasm_module._free(self.world_rotations_ptr)
                self.wasm_module._free(self.world_scales_ptr)

            self.allocated_vertex_count = target_alloc

            # Allocate new buffers
            self.positions_ptr = self.wasm_module._malloc(3 * target_alloc * 4)
            self.rotations_ptr = self.wasm_module._malloc(4 * target_alloc * 4)
            self.scales_ptr = self.wasm_module._malloc(3 * target_alloc * 4)
            self.colors_ptr = self.wasm_module._malloc(4 * target_alloc)
            self.selection_ptr = self.wasm_module._malloc(target_alloc)
            self.data_ptr = self.wasm_module._malloc(8 * target_alloc * 4)
            self.world_positions_ptr = self.wasm_module._malloc(3 * 4 * 4)
            self.world_rotations_ptr = self.wasm_module._malloc(4 * 4 * 4)
            self.world_scales_ptr = self.wasm_module._malloc(3 * 4 * 4)

        # Copy data
        # Float32Array set into HEAPF32
        # Note: HEAPF32 is a view of the WASM memory. 
        # The offset must be calculated relative to the base pointer.
        # In Emscripten, HEAPF32.set(array, index) copies array to HEAPF32 starting at index.
        # Index is byte_offset / 4.
        
        if self.positions_ptr is not None:
            pos_offset = self.positions_ptr // 4
            self.wasm_module.HEAPF32.set(splat.positions, pos_offset)
            
        if self.rotations_ptr is not None:
            rot_offset = self.rotations_ptr // 4
            self.wasm_module.HEAPF32.set(splat.rotations, rot_offset)
            
        if self.scales_ptr is not None:
            scale_offset = self.scales_ptr // 4
            self.wasm_module.HEAPF32.set(splat.scales, scale_offset)
            
        # Uint8Array set into HEAPU8
        if self.colors_ptr is not None:
            self.wasm_module.HEAPU8.set(splat.colors, self.colors_ptr)
            
        if self.selection_ptr is not None:
            self.wasm_module.HEAPU8.set(splat.selection, self.selection_ptr)
```

**Parameters:**
*   `positionsPtr`: Allocated size = $3 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
*   `rotationsPtr`: Allocated size = $4 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
*   `scalesPtr`: Allocated size = $3 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
*   `colorsPtr`: Allocated size = $4 \times \text{allocatedVertexCount}$ bytes [S1, unofficial].
*   `selectionPtr`: Allocated size = $\text{allocatedVertexCount}$ bytes [S1, unofficial].
*   `dataPtr`: Allocated size = $8 \times \text{allocatedVertexCount} \times 4$ bytes [S1, unofficial].
*   `worldPositionsPtr`: Allocated size = $3 \times 4 \times 4$ bytes [S1, unofficial].
*   `worldRotationsPtr`: Allocated size = $4 \times 4 \times 4$ bytes [S1, unofficial].
*   `worldScalesPtr`: Allocated size = $3 \times 4 \times 4$ bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Concurrent Calls:** Concurrent calls to `pack()` can cause race conditions during WASM initialization and memory reallocation. This is mitigated by serializing calls using a loading flag and yielding to the event loop via `setTimeout` [S1, unofficial].
*   **Frequent Reallocation:** Memory is only freed and reallocated when the next power-of-2 capacity exceeds the current `allocatedVertexCount`. This prevents frequent reallocations for incremental vertex count changes [S1, unofficial].
*   **Zero Vertex Count:** If `vertexCount` is 0, the power-of-2 calculation should result in 0, and no buffers should be allocated or copied [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The allocation step is $O(1)$ amortized due to power-of-2 growth, but the memory copy step is $O(N)$ where $N$ is the number of vertices. The concurrency wait loop adds latency proportional to the time taken by other event loop tasks.
*   **Memory Complexity:** $O(N)$ for the WASM heap buffers, where $N$ is the next power of 2 greater than or equal to the number of active Gaussians.
*   **Strengths:** Efficiently manages memory layout for WebAssembly environments; reduces fragmentation by using power-of-2 allocation; handles concurrency safely.
*   **Limitations:** Requires WebAssembly environment; power-of-2 allocation can lead to memory waste if the vertex count is not close to a power of 2.
*   **When to use:** When rendering 3D Gaussian Splatting in a WebAssembly context (e.g., browser-based viewers) where data must be transferred from JavaScript to WASM/C++ kernels.
*   **When not to use:** In native C++ or CUDA environments where direct memory management is preferred.

## Background
WASM Data Packing is part of the `gsplat.js` library, an unofficial JavaScript/TypeScript implementation of 3D Gaussian Splatting for the web [S1, unofficial]. It enables the transfer of Gaussian attributes from JavaScript typed arrays into the linear memory of a WebAssembly module, facilitating GPU rendering in browser environments.

## Sources
[S1] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
