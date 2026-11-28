# Shared-Memory Parallel Raycasting (CPU)

Also called: Multi-Core CPU Implementation, POSIX Threads Raycasting

## Overview
Shared-memory parallel raycasting on CPUs distributes the computational load of ray casting and integration across multiple threads within a single process using a shared-memory programming model [S1, unofficial]. It solves the performance bottleneck of sequential volume rendering by decomposing the output image into spatially disjoint work blocks, allowing multiple cores to render pixels concurrently [S1, unofficial]. It is distinguished from GPU-based parallel raycasting by its use of explicit thread management (e.g., POSIX threads) and dynamic work queues to handle load imbalance, rather than relying on hardware-level SIMT execution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar/vector volume dataset, camera parameters (view matrix, projection matrix), and a user-specified number of rendering threads [S1, unofficial].
*   **Output:** A 2D final image buffer containing the composited color and opacity for each pixel [S1, unofficial].

**Data Structures:**
1.  **Shared-Memory Data Structure:** A global memory region visible to all threads that stores precomputed values common to all rendering tasks, such as the inverse of the model-view matrix [S1, unofficial].
2.  **Work Queue:** A thread-safe queue containing work assignments (blocks of pixels) to be processed by slave threads [S1, unofficial].
3.  **Final Image Buffer:** A shared memory buffer where threads write the final pixel color and opacity [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** The application launches a user-specified number of rendering threads. Each thread acts as a "slave" and enters an event loop [S1, unofficial].
2.  **Precomputation:** The master thread computes values common to all slaves (e.g., inverse model-view matrix) and stores them in the shared-memory data structure [S1, unofficial].
3.  **Work Assignment Generation:** The master thread computes a list of work assignments. Each assignment is a spatially disjoint region of the final image (a work block) [S1, unofficial].
4.  **Synchronization:** A pair of shared-memory barriers synchronizes communication between the master and slave threads in the event loop [S1, unofficial].
5.  **Execution Loop (Slave Threads):**
    a.  The master thread signals slave threads to begin rendering [S1, unofficial].
    b.  Each slave thread requests a work block from the work queue [S1, unofficial].
    c.  The thread performs raycasting over all pixels in the assigned block, performing color and opacity integration along each ray [S1, unofficial].
    d.  The thread writes the results to the final image buffer [S1, unofficial].
    e.  The thread returns to step (b) to request the next work block until the queue is empty [S1, unofficial].
6.  **Completion:** The master thread waits for all slave threads to finish their current blocks (potentially using a barrier if "block synchronization" is enabled) and then proceeds to the next frame or task [S1, unofficial].

**Formulas:**
*   **Ray Casting:** For each pixel $(u, v)$ in a work block, a ray is generated. The volume is sampled along the ray. The color $C$ and opacity $\alpha$ are integrated using standard volume rendering integration (Levoy’s formulation) [S1, unofficial]:
    $$ C = \sum_{i=0}^{n-1} c_i \alpha_i \prod_{j=0}^{i-1} (1 - \alpha_j) $$
    $$ \alpha = 1 - \prod_{i=0}^{n-1} (1 - \alpha_i) $$
    Where $c_i$ and $\alpha_i$ are the color and opacity at sample $i$, and $n$ is the number of samples along the ray [S1, unofficial].

**Parameters:**
*   **Work Block Tessellation:** The user specifies an arbitrary tessellation of the output image forming the basis for work assignments, such as blocks of size $4 \times 4$, $8 \times 8$, $512 \times 1$, or $1 \times 512$ pixels [S1, unofficial].
*   **Thread Count:** User-specified number of rendering threads, typically matching the number of available CPU cores [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Load Imbalance:** Static work assignment (round-robin) can suffer from load imbalance if the volume data or camera view causes some rays to require significantly more computation than others, making dynamic assignment preferable [S1, unofficial].
*   **Memory Contention:** Concurrent writes to the final image buffer or shared data structures can cause cache thrashing or race conditions if not managed carefully [S1, unofficial].
*   **Block Synchronization:** Forcing all threads to process blocks in lock-step via shared barriers can induce better memory utilization by having multiple worker threads access nearby regions of the source data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import threading
import queue

# Global shared state
shared_data = {}
work_queue = queue.Queue()
final_image_buffer = None
stop_event = threading.Event()

def compute_shared_data():
    """Master thread computes common values."""
    shared_data['inv_mv_matrix'] = [[1.0]*4 for _ in range(4)]

def generate_ray(u, v, inv_mv_matrix):
    """Generates ray origin and direction from camera through pixel (u, v)."""
    return [0.0, 0.0, 0.0], [0.0, 0.0, -1.0]

def compute_ray_length(origin, direction, volume_bounds):
    """Computes distance from origin to volume exit or far plane."""
    return 100.0

def trilinear_sample(volume_data, pos):
    """Returns density/value at position using trilinear interpolation."""
    return 0.5

def apply_transfer_function(value):
    """Maps density to color [r,g,b] and opacity alpha."""
    return [0.5, 0.5, 0.5], 0.1

def raycast_pixel(u, v, inv_mv_matrix, volume_data, volume_bounds, step_size):
    """Cast a single ray and integrate color/opacity using Levoy's formulation."""
    ray_origin, ray_direction = generate_ray(u, v, inv_mv_matrix)
    ray_length = compute_ray_length(ray_origin, ray_direction, volume_bounds)
    
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    t = 0.0
    
    while t < ray_length:
        pos = [ray_origin[i] + ray_direction[i] * t for i in range(3)]
        sample = trilinear_sample(volume_data, pos)
        c, a = apply_transfer_function(sample)
        
        # Front-to-back compositing
        weight = a * (1.0 - alpha)
        color = [color[i] + c[i] * weight for i in range(3)]
        alpha = alpha + a * (1.0 - alpha)
        
        if alpha >= 1.0:
            break
        t += step_size
        
    return color, alpha

def slave_thread_event_loop(thread_id):
    """Slave thread event loop."""
    while not stop_event.is_set():
        try:
            block = work_queue.get(timeout=1.0)
        except queue.Empty:
            continue
            
        x_min, y_min, x_max, y_max = block
        
        for y in range(y_min, y_max):
            for x in range(x_min, x_max):
                color, alpha = raycast_pixel(x, y, shared_data['inv_mv_matrix'], volume_data, volume_bounds, step_size)
                final_image_buffer[y][x] = (color, alpha)
                
        work_queue.task_done()

def master_thread_render(image_width, image_height, num_threads, block_size, volume_data, volume_bounds, step_size):
    """Master thread manages rendering."""
    global final_image_buffer
    final_image_buffer = [[None for _ in range(image_width)] for _ in range(image_height)]
    
    compute_shared_data()
    
    blocks = []
    for y in range(0, image_height, block_size):
        for x in range(0, image_width, block_size):
            blocks.append((x, y, min(x + block_size, image_width), min(y + block_size, image_height)))
            
    for block in blocks:
        work_queue.put(block)
        
    threads = []
    for i in range(num_threads):
        t = threading.Thread(target=slave_thread_event_loop, args=(i,))
        t.start()
        threads.append(t)
        
    work_queue.join()
    stop_event.set()
    for t in threads:
        t.join()
```

## Subtopics

### Hybrid Parallelism for CPU Concurrency Testing
A hybrid parallelism approach that combines image-tile decomposition with domain decomposition to distribute the data volume across groups of threads, allowing testing at varying concurrency levels while maintaining image and data volume size [S1, unofficial]. It uses multiple groups of threads at a given concurrency level, with each group allocating its own memory buffer disjoint from other groups [S1, unofficial]. Thread affinity is enforced using the `-Mconcur=numa` flag in the PGI compiler to prevent threads from migrating across cores [S1, unofficial]. Data decomposition configurations vary by platform:
*   **Intel/Nehalem:** 1 thread per buffer, 8 disjoint buffers, buffer size $256 \times 256 \times 256$ [S1, unofficial].
*   **AMD/MagnyCours:** 2 threads per buffer, 4 disjoint buffers, buffer size $256 \times 256 \times 512$ [S1, unofficial].
*   **Intel/Nehalem (4 threads/buffer):** 2 buffers, size $256 \times 512 \times 512$ [S1, unofficial].
*   **Intel/Nehalem (8 threads/buffer):** 1 buffer, size $512 \times 512 \times 512$ [S1, unofficial].
*   **AMD/MagnyCours (6 threads/buffer):** 4 buffers, size $256 \times 256 \times 512$ [S1, unofficial].
*   **AMD/MagnyCours (24 threads/buffer):** 1 buffer, size $512 \times 512 \times 512$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Dynamic work assignment performs better than static assignment due to reduced load imbalance [S1, unofficial]. Performance scales with the number of cores but is limited by memory bandwidth and cache contention [S1, unofficial].
*   **Strengths:** Flexible work distribution, effective load balancing with dynamic queues, straightforward implementation on multi-core CPUs [S1, unofficial].
*   **Limitations:** Thread management and synchronization overhead; static assignment suffers from load imbalance [S1, unofficial].
*   **When to Use:** Multi-core CPU rendering where dynamic load balancing is required [S1, unofficial].
*   **When Not to Use:** Extreme many-core scalability scenarios, which are better suited for GPU raycasting [S1, unofficial].

## Background
The method is based on Levoy’s formulation of volume rendering [S1, unofficial]. It was implemented using POSIX threads as the shared-memory programming model [S1, unofficial]. The approach was explored in the context of multi-core and many-core shared-memory parallel raycasting volume rendering optimization and tuning [S1, unofficial]. The hybrid parallelism approach was previously implemented for large-scale distributed-memory tests [S1, unofficial].

## Sources
[S1] Multi-core and Many-core Shared-memory Parallel Raycasting Volume Rendering Optimization and Tuning, https://digital.library.unt.edu/ark:/67531/metadc829701/?q=empty space skipping volume rendering, fetched 2026-10-28 (unofficial)
