# Threaded Chunk Rendering

Also called: RendererManager

## Overview
Threaded Chunk Rendering is a CPU-side parallelization strategy that distributes volumetric rendering tasks across a fixed-size thread pool to maximize multi-core utilization while managing synchronization overhead [S1, unofficial]. It differs from [CPU Multithreaded Rendering](6052-cpu-multithreaded-rendering.md) by explicitly decoupling task generation from execution via a queue-based manager, and from [Multi-GPU and Cluster Rendering](10538-multi-gpu-and-cluster-rendering.md) by remaining strictly within a single process and CPU core set [S1, unofficial]. The method solves the problem of dynamic load balancing in chunk-based voxel worlds by reusing renderer instances rather than spawning threads per frame [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A grid of loaded world chunks (dimensions $0 \le i, j, k < \text{LOADED\_WORLD\_SIZE}$) and a pool of pre-allocated or lazily-created Renderer objects [S1, unofficial].
- Output: Rendered frames for each chunk, composited into the final image (typically using Z-buffering to handle overdraw) [S1, unofficial].

**Data Structures:**
The `RendererManager` maintains the following state:
- `freeRenderers`: A thread-safe queue (`ThreadsafeQueue<Renderer*>`) holding available renderer instances [S1, unofficial].
- `activeRenderers`: An atomic integer (`std::atomic<int>`) initialized to 0, tracking currently executing render tasks [S1, unofficial].
- `cv`: A condition variable (`std::condition_variable`) used to block the main loop or manage synchronization when the pool is idle [S1, unofficial].
- `mtx`: A mutex (`std::mutex`) protecting the condition variable and queue state [S1, unofficial].
- `MAX_RENDERERS`: A constant defining the maximum concurrent threads (typically 20) [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Create `MAX_RENDERERS` Renderer instances and push them all into `freeRenderers`. Initialize `activeRenderers` to 0.
2. **Task Distribution**: Iterate over all loaded chunks $(i, j, k)$. For each chunk, invoke `renderChunk(chunk)`.
3. **Renderer Acquisition**: Inside `renderChunk`:
   - Lock `mtx` and wait on `cv` while `activeRenderers >= MAX_RENDERERS` AND `freeRenderers` is empty. This prevents thread explosion and ensures pool reuse [S1, unofficial].
   - Pop a `Renderer*` from `freeRenderers`.
   - Increment `activeRenderers`.
   - Unlock `mtx`.
4. **Execution**: Spawn a detached `std::thread` that executes `Chunk::render(chunk, renderer)`.
   - Inside `Chunk::render`: Call `renderer->renderChunk(this)`.
   - Upon completion, call `rendererManager.freeRenderer(renderer)` [S1, unofficial].
5. **Renderer Release**: Inside `freeRenderer(renderer)`:
   - Lock `mtx`.
   - Push `renderer` back into `freeRenderers`.
   - Decrement `activeRenderers`.
   - If `activeRenderers` reaches 0, notify `cv` (useful for frame synchronization) [S1, unofficial].
   - Unlock `mtx`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class RendererManager:
    def __init__(self, max_renderers=20):
        self.free_renderers = ThreadsafeQueue()
        self.active_renderers = AtomicInt(0)
        self.cv = ConditionVariable()
        self.mtx = Mutex()
        self.max_renderers = max_renderers
        
        # Initialize pool
        for _ in range(max_renderers):
            self.free_renderers.push(Renderer())

    def render_chunk(self, chunk):
        # Acquire a renderer
        with self.mtx:
            while self.active_renderers.get() >= self.max_renderers and self.free_renderers.is_empty():
                self.cv.wait(self.mtx)
            
            renderer = self.free_renderers.pop()
            self.active_renderers.increment()
        
        # Spawn thread
        thread = Thread(target=self._execute_render, args=(chunk, renderer))
        thread.daemon = True
        thread.start()

    def _execute_render(self, chunk, renderer):
        try:
            renderer.render_chunk(chunk)
        finally:
            self.free_renderer(renderer)

    def free_renderer(self, renderer):
        with self.mtx:
            self.free_renderers.push(renderer)
            self.active_renderers.decrement()
            if self.active_renderers.get() == 0:
                self.cv.notify_all()

class Chunk:
    def render(self, renderer):
        if renderer is None:
            raise ValueError("Renderer is null")
        renderer.render_chunk(self)
        # Note: In the source, freeRenderer is called here, 
        # but logically it should be called after the thread finishes.
        # The source code shows: rendererManager.freeRenderer(renderer) inside Chunk::render.
        # However, Chunk::render is called BY the thread. 
        # To be safe and correct, we assume the thread wrapper handles the free call 
        # or the source implies the renderer is returned immediately after renderChunk returns.
        # Based on [S1, unofficial], the thread calls Chunk::render, which calls renderer->renderChunk, 
        # then calls freeRenderer.
        renderer_manager.free_renderer(renderer)
```
*[G] The source code snippet shows `rendererManager.freeRenderer(renderer)` inside `Chunk::render`, which is executed by the spawned thread. This implies the renderer is returned to the pool immediately after the chunk rendering function returns, not after the thread object is joined. This is safe for detached threads as long as `Renderer` state is not accessed after return.*

**Parameters:**
- `MAX_RENDERERS`: 20 [S1, unofficial].
- `CHUNK_SIZE`: 8x8x8 blocks [S1, unofficial].
- `LOADED_WORLD_SIZE`: Determined by world segmentation logic [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Deadlock**: If `activeRenderers` is not decremented correctly, the queue will empty and no new threads will spawn [S1, unofficial].
- **Null Renderer**: The source includes a check for `renderer == nullptr` [S1, unofficial].
- **Z-Buffering**: The method relies on Z-buffering to prevent overdraw artifacts when multiple threads render overlapping chunks or when chunks are rendered out of order [S1, unofficial].

## Subtopics

### Mutex-based Read/Write Lock for Chunks
Implements a reader-writer lock using two pthread mutexes (`mutex_read`, `mutex_write`) and a `readers` counter to allow concurrent reads but exclusive writes for chunk data [S2, unofficial]. 
- **Acquire Read Lock (`chunk_open_ro`)**: Lock `mutex_read`, increment `readers`. If `readers` becomes 1, lock `mutex_write`. Unlock `mutex_read` [S2, unofficial].
- **Release Read Lock (`chunk_close_ro`)**: Lock `mutex_read`, decrement `readers`. If `readers` becomes 0, unlock `mutex_write`. Unlock `mutex_read` [S2, unofficial].
- **Acquire Write Lock (`chunk_open_rw`)**: Directly lock `mutex_write` [S2, unofficial].
- **Release Write Lock (`chunk_close_rw`)**: Directly unlock `mutex_write` [S2, unofficial].
- **Parameter**: `c->readers` (Integer counter, initialized to 0) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of chunks, with a constant factor reduced by parallelism. Thread creation overhead is amortized by reusing renderers [S1, unofficial].
- **Memory Complexity**: $O(M)$ where $M$ is `MAX_RENDERERS` (20) [S1, unofficial].
- **Performance**: Z-buffering provided a 1.33x speedup over the Painters Algorithm in early tests (0.027s vs 0.036s per frame) [S1, unofficial].
- **Strengths**: Efficient CPU utilization; avoids thread creation overhead per frame; simple synchronization [S1, unofficial].
- **Limitations**: Fixed thread pool size may not scale to very large worlds without tuning; CPU-bound [S1, unofficial].
- **When to Use**: Single-machine, multi-core CPU rendering of voxel worlds [S1, unofficial].
- **When Not to Use**: When GPU acceleration is available (use [GPU, web and parallel renderers](50-gpu-web-and-parallel-renderers.md) variants) or when cluster scaling is required [S1, unofficial].

## Background
Originated in the VidFurlan/Voxel1993 project as a "First Major Performance Improvement" over the Painters Algorithm, switching to Z-buffering and multi-threaded chunk management [S1, unofficial]. The approach was part of a broader effort to optimize voxel world rendering by combining chunking, Z-buffering, and multi-threading, with future plans to integrate Voxel Octrees [S1, unofficial]. The mutex-based variant appears in the kosshi-net/voxplat project for managing concurrent access to chunk data [S2, unofficial].

## Sources
[S1] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
[S2] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
