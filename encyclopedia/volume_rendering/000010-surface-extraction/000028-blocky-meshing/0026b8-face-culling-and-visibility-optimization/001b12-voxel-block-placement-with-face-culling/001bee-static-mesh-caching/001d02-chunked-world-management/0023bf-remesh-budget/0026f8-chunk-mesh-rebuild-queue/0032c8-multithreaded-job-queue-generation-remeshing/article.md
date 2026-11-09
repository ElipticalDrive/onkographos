# Multithreaded Job Queue (Generation & Remeshing)

Also called: Asynchronous Job Processing, Job Queue, LIFO Job Queue with Priority Handling, TaskManager for Multithreading, Worker Loop, World::WorkerLoop

## Overview
This method distributes terrain generation and meshing tasks across multiple CPU threads using a LIFO job queue with priority handling [S1, unofficial]. It solves the problem of blocking the main thread during chunk processing by decoupling job submission from execution [S1, unofficial]. It is distinguished from [Asynchronous Chunk Loading](4216-asynchronous-chunk-loading.md) by its explicit priority-based ordering (remesh before generation) and its tight integration with the immediate meshing pipeline rather than just data loading [S1, unofficial].

## How it works
**Inputs:**
- A job queue (`m_JobQueue`) storing `WorkerJob` structs.
- A result queue (`m_ResultQueue`) storing `WorkerResult` structs.
- A generation function `gen: Int3 -> Chunk` [S2, unofficial].
- A maximum remesh limit `m_MaxRemeshPerFrame` [S1, unofficial].

**Outputs:**
- Completed `WorkerResult` objects containing generated chunks or optimized meshes.
- Side effects: Neighbor chunks marked as `NeedRemesh` after generation [S1, unofficial].

**Data Structures:**

1.  **WorkerJob**:
    -   `JobType type`: Enum, either `GenerateJobType` or `MeshJobType` [S1, unofficial].
    -   `ChunkPosition pos`: The 3D coordinates of the chunk [S1, unofficial].
    -   For `MeshJobType` only:
        -   `std::shared_ptr<Chunk> center`: The chunk being remeshed [S1, unofficial].
        -   `std::shared_ptr<Chunk> left`: Neighbor chunk at `pos.x - 1` [S1, unofficial].
        -   `std::shared_ptr<Chunk> right`: Neighbor chunk at `pos.x + 1` [S1, unofficial].
        -   `std::shared_ptr<Chunk> front`: Neighbor chunk at `pos.z - 1` [S1, unofficial].
        -   `std::shared_ptr<Chunk> back`: Neighbor chunk at `pos.z + 1` [S1, unofficial].

2.  **WorkerResult**:
    -   Contains the output data moved from the generator or mesh object (e.g., `m_Blocks`, `m_HeightTable`, `m_OpaqueSSBO`, vertices, indices) [S1, unofficial].

**Parameters:**
-   `workerCount`: Number of worker threads. Typically `std::thread::hardware_concurrency()`, with a fallback to 4 if unavailable [S1, unofficial]. Alternatively `Environment.ProcessorCount` [S2, unofficial].
-   `m_MaxRemeshPerFrame`: Integer limit on remesh jobs pushed per frame to control CPU load [S1, unofficial].

**Algorithm:**

1.  **Initialization**:
    -   Determine `workerCount` [S1, unofficial].
    -   Spawn `workerCount` threads. Each thread executes the `World::WorkerLoop` function [S1, unofficial].

2.  **Job Submission (Main Thread)**:
    -   When a chunk needs processing, determine if it is a remesh or generation job.
    -   **Remesh Jobs**: Create `WorkerJob` with `type = MeshJobType`. Retrieve neighbor chunk pointers (`left`, `right`, `front`, `back`) via lambda or lookup [S1, unofficial]. Push to `m_JobQueue`.
    -   **Generation Jobs**: Create `WorkerJob` with `type = GenerateJobType` and `pos`. Do not include neighbor pointers [S1, unofficial]. Push to `m_JobQueue`.
    -   **Priority Ordering**: Push all remesh jobs first, then generation jobs. Because the queue is LIFO (Last-In, First-Out), generation jobs are at the top and processed first [S1, unofficial].
    -   **Frame Limit**: Limit the number of remesh jobs pushed per frame to `m_MaxRemeshPerFrame` [S1, unofficial].

3.  **Worker Loop (Worker Threads)**:
    -   Loop indefinitely.
    -   Pop a job from `m_JobQueue`. This is a blocking call [S1, unofficial].
    -   If the queue is stopped or empty (no job available), break the loop [S1, unofficial].
    -   **Execute Job**:
        -   If `type == GenerateJobType`:
            -   Create a `Generator` instance with the world and `pos` [S1, unofficial].
            -   Call `generator.GenerateFromChunk()` [S1, unofficial].
            -   Move `m_Blocks`, `m_HeightTable`, and `m_TreeLevel` from the generator to `WorkerResult` [S1, unofficial].
            -   Push `WorkerResult` to `m_ResultQueue` [S1, unofficial].
            -   Mark neighbor chunks as `NeedRemesh` [S1, unofficial].
        -   If `type == MeshJobType`:
            -   Create a `Mesh` object [S1, unofficial].
            -   Call `mesh.MeshFromChunk` passing the texture and pointers to `center`, `left`, `right`, `front`, `back` chunks [S1, unofficial].
            -   Move opaque/transparent SSBOs, vertices, and indices from `mesh` to `WorkerResult` [S1, unofficial].
            -   Push `WorkerResult` to `m_ResultQueue` [S1, unofficial].

4.  **Result Consumption (Main Thread)**:
    -   Dequeue completed tasks from `m_ResultQueue` using `manager.TryDequeueCompleted(out Completion r)` [S2, unofficial].
    -   Check `r.Error` for issues [S2, unofficial].
    -   Apply results to the scene (e.g., update chunk data, upload buffers).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
import queue
import math

# Data Structures
class WorkerJob:
    def __init__(self, job_type, pos, center=None, left=None, right=None, front=None, back=None):
        self.type = job_type
        self.pos = pos
        self.center = center
        self.left = left
        self.right = right
        self.front = front
        self.back = back

class WorkerResult:
    def __init__(self, blocks=None, heights=None, tree_level=0, mesh_data=None):
        self.blocks = blocks
        self.heights = heights
        self.tree_level = tree_level
        self.mesh_data = mesh_data

class World:
    def __init__(self):
        self.m_JobQueue = queue.Queue()
        self.m_ResultQueue = queue.Queue()
        self.m_MaxRemeshPerFrame = 10 # Example value
        self.worker_threads = []
        self.worker_count = 0

    def start_workers(self):
        # Determine thread count
        hw_concurrency = threading.active_count() # Placeholder for hardware_concurrency check
        self.worker_count = max(4, hw_concurrency) # Fallback to 4
        
        for _ in range(self.worker_count):
            t = threading.Thread(target=self.worker_loop, daemon=True)
            t.start()
            self.worker_threads.append(t)

    def worker_loop(self):
        while True:
            # Blocking pop
            try:
                job = self.m_JobQueue.get(timeout=1.0)
            except queue.Empty:
                # Check if queue is stopped or empty logic applies
                break
            
            if job is None:
                break

            result = WorkerResult()

            if job.type == "GenerateJobType":
                # Create generator and generate
                generator = Generator(self, job.pos)
                generator.generate_from_chunk()
                
                # Move data
                result.blocks = generator.move_blocks()
                result.heights = generator.move_heights()
                result.tree_level = generator.get_tree_level()
                
                # Mark neighbors for remesh
                self.mark_neighbors_need_remesh(job.pos)

            elif job.type == "MeshJobType":
                # Create mesh and mesh from chunk
                mesh = Mesh()
                mesh.mesh_from_chunk(
                    texture=None, # Placeholder
                    center=job.center,
                    left=job.left,
                    right=job.right,
                    front=job.front,
                    back=job.back
                )
                
                # Move mesh data
                result.mesh_data = mesh.move_data()

            # Push result
            self.m_ResultQueue.put(result)
            self.m_JobQueue.task_done()

    def submit_jobs(self, remesh_jobs, generation_jobs):
        # Push remesh jobs first
        for job in remesh_jobs[:self.m_MaxRemeshPerFrame]:
            self.m_JobQueue.put(job)
        
        # Push generation jobs
        for job in generation_jobs:
            self.m_JobQueue.put(job)

    def mark_neighbors_need_remesh(self, pos):
        # Implementation specific to marking neighbors
        pass

class Generator:
    def __init__(self, world, pos):
        self.world = world
        self.pos = pos
        self.m_Blocks = []
        self.m_HeightTable = []
        self.m_TreeLevel = 0

    def generate_from_chunk(self):
        # Noise generation logic
        pass

    def move_blocks(self):
        blocks = self.m_Blocks
        self.m_Blocks = []
        return blocks

    def move_heights(self):
        heights = self.m_HeightTable
        self.m_HeightTable = []
        return heights

    def get_tree_level(self):
        return self.m_TreeLevel

class Mesh:
    def __init__(self):
        self.m_OpaqueSSBO = None
        self.m_TransparentSSBO = None
        self.vertices = []
        self.indices = []

    def mesh_from_chunk(self, texture, center, left, right, front, back):
        # Meshing logic using neighbors
        pass

    def move_data(self):
        return {
            'opaque_ssbo': self.m_OpaqueSSBO,
            'transparent_ssbo': self.m_TransparentSSBO,
            'vertices': self.vertices,
            'indices': self.indices
        }
```

**Edge Cases and Pitfalls:**
-   **Precision**: Not given in the sources.
-   **Boundaries**: Neighbor pointers (`left`, `right`, etc.) must handle boundary checks (e.g., if a chunk is at the world edge, the neighbor might be null or a default empty chunk) [S1, unofficial].
-   **Degenerate Input**: If `gen(pos)` fails, the error handling mechanism (`r.Error` in [S2, unofficial]) should be checked by the consumer [S2, unofficial].
-   **Queue Stopping**: The worker loop must handle the case where the queue is stopped gracefully to avoid infinite blocking [S1, unofficial].

## Subtopics

### [VoxelTerrainJob Pipeline](8732-voxelterrainjob-pipeline.md)
A multi-phase, resumable job system that yields back to the main thread between phases to prevent frame drops, unlike the continuous blocking loop of the standard job queue. Choose this when frame-time stability is critical and jobs can be split into smaller phases.

## Operating characteristics
-   **Time Complexity**: Job execution time depends on the complexity of generation and meshing. Queue operations are O(1) for push/pop.
-   **Memory Complexity**: O(N) where N is the number of pending jobs. Each job holds references to chunks, which can be memory-intensive if chunks are large.
-   **Measured Performance**: Not given in the sources.
-   **Scaling**: Scales with `workerCount`. Limited by CPU core count and memory bandwidth.
-   **Strengths**: Prevents main thread blocking; prioritizes remeshing to keep visible geometry updated; simple LIFO priority ensures generation jobs are processed first when mixed.
-   **Limitations**: LIFO priority means generation jobs are processed before remesh jobs if pushed in that order (which is the intended behavior for freshness, but might starve remeshes if generation is heavy); no built-in phase yielding (unlike the VoxelTerrainJob Pipeline).
-   **When to use**: When real-time terrain modification requires non-blocking updates and remeshing is frequent.
-   **When not to use**: When frame-time consistency is more important than throughput and jobs can be broken into smaller phases.

## Background
The method originates from the VoxelEngine project by adrienlabayle [S1, unofficial] and the Voxel-Mesh-Optimization-Library by nicolas-goyon [S2, unofficial]. It is used in voxel-based game engines to manage the computational cost of terrain generation and meshing.

## Sources
[S1] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S2] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
