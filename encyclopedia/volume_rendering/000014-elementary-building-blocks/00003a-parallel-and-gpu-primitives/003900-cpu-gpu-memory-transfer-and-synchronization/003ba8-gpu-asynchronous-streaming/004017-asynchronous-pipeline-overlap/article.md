# Asynchronous Pipeline (Overlap)

Also called: Concurrent SSD Read and GPU Rendering, Hierarchical Asynchronous Execution, Overlap, Overlap operations

## Overview
A pipelining strategy that overlaps SSD read/prefetch, Host-to-Device (H2D) staging, and GPU rendering/backpropagation to hide I/O latency [S1, unofficial]. It addresses the performance bottleneck of out-of-core volume roaming, where sequential execution of cache scanning, data fetching, and rendering drastically reduces frame rates [S1, unofficial]. It sets itself apart from sequential implementations by using multi-threading and asynchronous I/O to allow rendering to proceed while data is still being fetched from storage or the network [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of data blocks (e.g., 3D Gaussian splats or volume chunks) residing on an SSD, a camera trajectory defining the visible region, and a working set size limit.
*   **Output:** Rendered frames and optimized parameters, with minimal PCIe traffic and GPU idle time.

**Data Structures:**
*   **CPU Cache:** Manages block residency and dirty tracking. Uses an index structure (e.g., Morton code based) to map spatial regions to block locations on disk [S1, unofficial].
*   **Double-Buffered GPU Block Buffers:** Two sets of VRAM buffers. One is actively used for rendering the current iteration, while the other is populated with the next iteration's data [S1, unofficial].
*   **CUDA Streams:** Separate streams for GPU compute and memory copies (H2D/D2H) to enable true concurrency [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Load the initial working set of blocks from SSD to CPU cache. Allocate double-buffered GPU buffers. Initialize separate CUDA streams for compute and copy operations.
2.  **Iteration Start:** Determine the visible block union for the current camera position. Identify blocks that need to be fetched (not in GPU) and blocks that need to be evicted (in GPU but not visible).
3.  **Concurrent Execution (Overlap):**
    *   **Stage 1 (SSD Read/Prefetch):** Dedicated I/O threads read the required blocks from SSD into the CPU cache. Prefetching may occur for the *next* iteration's likely visible blocks [S1, unofficial].
    *   **Stage 2 (H2D Transfer):** Asynchronously copy the newly fetched blocks from CPU cache to the *next* GPU buffer using a dedicated CUDA copy stream [S1, unofficial].
    *   **Stage 3 (GPU Compute):** Render and/or backpropagate using the *current* GPU buffer using a dedicated CUDA compute stream [S1, unofficial].
    *   **Stage 4 (D2H Transfer & Flush):** Asynchronously copy evicted blocks from the *current* GPU buffer back to CPU cache (if dirty) and trigger asynchronous SSD flushes from CPU cache to disk [S1, unofficial].
4.  **Buffer Swap:** After compute completes, swap the current and next GPU buffers. The "next" buffer becomes the "current" one for the next iteration.
5.  **Repeat:** Go to Step 2 for the next frame/iteration.

**Formulas:**
*   **Iteration Time (Overlap):** $T_{iter} \approx \max(T_{compute}, T_{IO} + T_{H2D} + T_{D2H})$ [S1, unofficial].
*   **Iteration Time (Serial):** $T_{iter} = T_{IO} + T_{H2D} + T_{compute} + T_{D2H}$ [S1, unofficial].
*   **PCIe Traffic:** Minimized by differential streaming, transferring only changed blocks [S1, unofficial].

**Parameters:**
*   **Number of I/O Threads:** Typically matches the number of CPU cores or SSD channels [S1, unofficial].
*   **CUDA Streams:** At least two (one for compute, one for copy) [S1, unofficial].
*   **Buffer Size:** Sized to hold the maximum expected working set of blocks [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure double-buffering logic correctly handles block validity to avoid rendering stale data [S1, unofficial].
*   **Boundaries:** Handle cases where the visible set changes drastically between frames, requiring full buffer reloads [S1, unofficial].
*   **Degenerate Input:** If the working set fits entirely in VRAM, overlap overhead may be negligible, but the pipeline should still function correctly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.cuda as cuda

class AsynchronousPipeline:
    def __init__(self, num_blocks, block_size, num_streams=2):
        self.num_blocks = num_blocks
        self.block_size = block_size
        self.num_streams = num_streams
        
        # Allocate double-buffered GPU memory
        self.gpu_buffers = [cuda.empty((num_blocks, block_size), dtype=torch.float32) for _ in range(2)]
        self.cpu_cache = [None] * num_blocks  # Placeholder for CPU cache management
        self.streams = [cuda.Stream() for _ in range(num_streams)]
        
        self.current_buffer_idx = 0
        self.next_buffer_idx = 1
        
    def load_block_from_disk(self, block_id):
        # [G] Placeholder for actual SSD read logic
        # In practice, this would be handled by dedicated I/O threads
        pass
        
    def render(self, buffer_idx, stream_idx):
        # [G] Placeholder for actual GPU compute (rendering/backprop)
        # This would use self.gpu_buffers[buffer_idx]
        pass
        
    def transfer_h2d(self, block_ids, stream_idx):
        # [G] Placeholder for H2D transfer
        # This would copy data from CPU cache to self.gpu_buffers[next_buffer_idx]
        pass
        
    def transfer_d2h(self, block_ids, stream_idx):
        # [G] Placeholder for D2H transfer
        # This would copy evicted blocks from self.gpu_buffers[current_buffer_idx] to CPU cache
        pass
        
    def flush_dirty_blocks(self, stream_idx):
        # [G] Placeholder for async SSD flush
        pass
        
    def step(self, visible_blocks, evicted_blocks):
        # Step 1: Prefetch and H2D for next iteration
        with self.streams[0]:
            for block_id in visible_blocks:
                if block_id not in self.cpu_cache or self.cpu_cache[block_id] is None:
                    # [G] In real implementation, this would be triggered by I/O threads
                    # and data would be ready in CPU cache
                    pass
            # Transfer new blocks to next buffer
            self.transfer_h2d(visible_blocks, 0)
            
        # Step 2: Render current iteration
        with self.streams[1]:
            self.render(self.current_buffer_idx, 1)
            
        # Step 3: D2H and Flush for current iteration
        with self.streams[0]:
            self.transfer_d2h(evicted_blocks, 0)
            self.flush_dirty_blocks(0)
            
        # Step 4: Swap buffers
        self.current_buffer_idx, self.next_buffer_idx = self.next_buffer_idx, self.current_buffer_idx

# Usage example
pipeline = AsynchronousPipeline(num_blocks=1000, block_size=1024)
for frame in range(100):
    visible = get_visible_blocks(frame)
    evicted = get_evicted_blocks(frame)
    pipeline.step(visible, evicted)
```

## Subtopics

### Concurrent 3-Stage Pipeline
A pipeline architecture that maximizes hardware utilization by running three stages (Image Preparation, TripoSR Inference, Voxelization) simultaneously using queues and separate process pools [S2, unofficial]. It sets itself apart by orchestrating CPU and GPU workloads across different processes to prevent idle time [S2, unofficial]. Choose this when dealing with heterogeneous workloads involving both CPU-intensive (voxelization) and GPU-intensive (inference) tasks [S2, unofficial]. On an M4 Pro with 48 GB RAM, processing takes ~5 seconds per emoji, with ~225 emojis completed in ~19 minutes. GPU utilization is ~80%, CPU ~60%, RAM ~23 GB [S2, unofficial]. The GPU is the bottleneck because TripoSR inference is sequential, whereas CPU stages can be parallelized and overlapped with GPU work to prevent idle time [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Iteration time is dominated by the maximum of compute time and I/O time, rather than their sum [S1, unofficial].
*   **Memory Complexity:** Requires double the GPU memory for the working set due to double-buffering [S1, unofficial].
*   **Measured Performance:** In TideGS, serializing data movement and compute increases iteration time to 210.5 ms compared to 90.7 ms with overlap, while keeping PCIe traffic at 0.10 GB/iter [S1, unofficial].
*   **Scaling:** Scales well with SSD bandwidth and PCIe throughput. Performance improves as I/O latency becomes a larger fraction of total iteration time [S1, unofficial].
*   **Strengths:** Hides I/O latency, reduces GPU idle time, minimizes PCIe traffic through differential streaming [S1, unofficial].
*   **Limitations:** Increased implementation complexity, higher memory usage due to double-buffering, potential for I/O bottlenecks if SSD bandwidth is insufficient [S1, unofficial].
*   **When to Use:** Out-of-core rendering/optimization where the working set exceeds VRAM capacity [S1, unofficial].
*   **When Not to Use:** When the working set fits entirely in VRAM, or when I/O latency is negligible compared to compute time [S1, unofficial].

## Background
Originated in the context of out-of-core 3D Gaussian Splatting optimization, specifically in the TideGS paper [S1, unofficial]. It addresses the challenge of training models with over one billion primitives that cannot fit in GPU memory [S1, unofficial]. The technique leverages the asynchronous capabilities of modern GPUs (e.g., Fermi architecture and later) and SSDs to create a smooth data pipeline [S1, unofficial].

## Sources
[S1] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
[S2] mohamedmansour/image-to-voxel, https://github.com/mohamedmansour/image-to-voxel, fetched 2026-10-26 (unofficial)
