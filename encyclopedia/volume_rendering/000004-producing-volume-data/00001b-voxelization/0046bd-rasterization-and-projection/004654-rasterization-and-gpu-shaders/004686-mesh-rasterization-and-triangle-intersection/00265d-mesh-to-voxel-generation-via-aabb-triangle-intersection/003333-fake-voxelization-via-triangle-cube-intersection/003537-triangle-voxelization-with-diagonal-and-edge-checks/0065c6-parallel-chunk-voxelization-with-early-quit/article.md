# Parallel Chunk Voxelization with Early Quit

Also called: Rayon Chunk Voxelization, Threaded Batch Voxelization

## Overview
This method accelerates the conversion of triangle meshes into voxel data by distributing the per-chunk voxelization workload across multiple threads using the Rayon parallel iterator library. It solves the bottleneck of sequential chunk processing by allowing concurrent execution, while implementing an early-quit mechanism to skip chunks that contain no geometry or produce empty voxel batches. It builds on [Triangle Voxelization with Diagonal and Edge Checks](7477-triangle-voxalization-with-diagonal-and-edge-checks.md) for the actual intersection logic within each chunk.

## How it works
**Inputs and Outputs:**
- **Input:** A list of chunk world positions (`chunk_positions`), a mapping from chunk positions to triangle face indices (`chunk_face_map`), a list of mesh vertices (`vertices`), and model parameters (`depth`, `chunk_world_size`, `voxel_size`, `voxels_per_axis`, `mesh_min`).
- **Output:** A stream of processed chunks, where each chunk is a pair of its world position (`IVec3`) and a voxel batch (`Batch<i32>`) containing the voxelized data (patches).

**Data Structures:**
- `chunk_face_map`: A hash map (`FxHashMap<IVec3, Vec<IVec3>>`) mapping each chunk's world position to a list of face indices belonging to that chunk.
- `Batch<i32>`: A structure representing the voxel data for a chunk, containing patches. It has a method `has_patches()` to check if it contains valid data.
- `IVec3`: A 3D integer vector type representing chunk coordinates.
- `AtomicUsize`: Used for thread-safe counters.
- `AtomicBool`: Used for the stop signal.
- `Channel`: A bounded channel (capacity 1024) for sending `(IVec3, Batch<i32>)` pairs from worker threads to the main thread.

**Parameters:**
- `chunk_face_map`: Maps chunk positions to face indices.
- `depth`: LOD depth used to query `voxels_per_axis` and `chunk_world_size` from the model.
- `chunk_world_size`: Size of a chunk in world units.
- `voxel_size`: Size of a single voxel.
- `voxels_per_axis`: Number of voxels along one axis in a chunk.
- `mesh_min`: Minimum bounding box of the mesh.
- `stop_signal`: A boolean flag to signal threads to terminate.

**Numbered Steps:**
1.  Initialize atomic counters: `early_quit_no_faces`, `early_quit_empty_faces`, `early_quit_empty_batch`, `processed_chunks` (all `Arc<AtomicUsize>`), and `stop_signal` (`Arc<AtomicBool>`).
2.  Create a bounded channel `(tx, rx)` with capacity 1024 for `(IVec3, Batch<i32>)` pairs.
3.  Spawn worker threads using Rayon's parallel iterator: `chunk_positions.par_iter().for_each(|chunk_position| { ... })`.
4.  Inside the parallel loop for each `chunk_position`:
    a.  Check `stop_signal_clone.load(Ordering::Relaxed)`. If true, return immediately.
    b.  Look up `faces` in `chunk_face_map` for `chunk_position`. If not found (`None`), increment `early_quit_no_faces` and return.
    c.  If `faces.is_empty()`, increment `early_quit_empty_faces` and return.
    d.  Call `Self::voxelize_chunk(chunk_position, depth, chunk_world_size, voxel_size, voxels_per_axis, mesh_min, faces, &vertices)`. This function performs the actual voxelization using the diagonal and edge checks from the parent method.
    e.  If the resulting `batch` has no patches (`!batch.has_patches()`), increment `early_quit_empty_batch` and return.
    f.  If `batch` has patches, increment `processed_chunks` and send `(*chunk_position, batch)` through `tx`.
5.  In the main thread:
    a.  Drop `tx` to close the channel.
    b.  Iterate over `rx.iter()`.
    c.  For each `(chunk_position, batch)`, call `self.model.get_or_create_chunk(chunk_position).apply_batch(&mut interner, &batch)` to integrate the voxel data into the model.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import rayon
import std::sync::atomic::{AtomicUsize, AtomicBool, Ordering}
import std::sync::Arc
import std::sync::mpsc::channel

def parallel_chunk_voxelization(
    chunk_positions: Vec<IVec3>,
    chunk_face_map: FxHashMap<IVec3, Vec<IVec3>>,
    depth: i32,
    chunk_world_size: f32,
    voxel_size: f32,
    voxels_per_axis: i32,
    mesh_min: Vec3,
    vertices: Vec<Vec3>,
    model: &Model,
    interner: &mut Interner
):
    # Initialize atomic counters
    early_quit_no_faces = Arc::new(AtomicUsize::new(0))
    early_quit_empty_faces = Arc::new(AtomicUsize::new(0))
    early_quit_empty_batch = Arc::new(AtomicUsize::new(0))
    processed_chunks = Arc::new(AtomicUsize::new(0))
    stop_signal = Arc::new(AtomicBool::new(false))

    # Create channel
    capacity = 1024
    (tx, rx) = channel::<(IVec3, Batch<i32>)>(capacity)

    # Clone shared state for threads
    chunk_face_map_clone = chunk_face_map.clone()
    vertices_clone = vertices.clone()
    early_quit_no_faces_clone = early_quit_no_faces.clone()
    early_quit_empty_faces_clone = early_quit_empty_faces.clone()
    early_quit_empty_batch_clone = early_quit_empty_batch.clone()
    processed_chunks_clone = processed_chunks.clone()
    stop_signal_clone = stop_signal.clone()

    # Parallel processing
    rayon::spawn(move || {
        chunk_positions.par_iter().for_each(|chunk_position| {
            # Check stop signal
            if stop_signal_clone.load(Ordering::Relaxed) {
                return;
            }

            # Get faces for chunk
            let faces = match chunk_face_map_clone.get(chunk_position) {
                Some(f) => f,
                None => {
                    early_quit_no_faces_clone.fetch_add(1, Ordering::SeqCst);
                    return;
                }
            };

            # Check if faces are empty
            if faces.is_empty() {
                early_quit_empty_faces_clone.fetch_add(1, Ordering::SeqCst);
                return;
            }

            # Voxelize chunk
            let batch = Self::voxelize_chunk(
                *chunk_position,
                depth,
                chunk_world_size,
                voxel_size,
                voxels_per_axis,
                mesh_min,
                faces,
                &vertices_clone
            );

            # Check if batch has patches
            if !batch.has_patches() {
                early_quit_empty_batch_clone.fetch_add(1, Ordering::SeqCst);
                return;
            }

            # Send batch if valid
            processed_chunks_clone.fetch_add(1, Ordering::SeqCst);
            tx.send((*chunk_position, batch)).unwrap();
        });
    });

    # Main thread collects results
    drop(tx);
    for (chunk_position, batch) in rx.iter() {
        model.get_or_create_chunk(chunk_position).apply_batch(&mut interner, &batch);
    }

    return {
        "early_quit_no_faces": early_quit_no_faces.load(Ordering::SeqCst),
        "early_quit_empty_faces": early_quit_empty_faces.load(Ordering::SeqCst),
        "early_quit_empty_batch": early_quit_empty_batch.load(Ordering::SeqCst),
        "processed_chunks": processed_chunks.load(Ordering::SeqCst)
    }
```

**Edge Cases and Pitfalls:**
- **Precision:** The `voxelize_chunk` function must handle floating-point precision issues when determining triangle-voxel intersections.
- **Boundaries:** Ensure `chunk_face_map` correctly handles chunks at the boundary of the mesh's bounding box.
- **Degenerate Input:** If `chunk_positions` is empty, the parallel loop does nothing, and the main thread receives no results.
- **Channel Blocking:** If the main thread is slow to consume results, the channel may block. The capacity of 1024 helps mitigate this.
- **Stop Signal:** The `stop_signal` must be checked at the beginning of each iteration to allow for graceful termination.

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of chunks, but with a constant factor reduced by parallelization. The actual speedup depends on the number of available CPU cores and the load balance of the chunks.
- **Memory Complexity:** O(N) for storing the chunk positions and face maps. The channel adds a small overhead.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with the number of CPU cores, provided the chunks are evenly distributed and the voxelization work per chunk is significant enough to outweigh the overhead of thread management.
- **Strengths:** Significant speedup for large meshes with many chunks. Early quit reduces unnecessary work for empty or sparse chunks.
- **Limitations:** Overhead of thread creation and synchronization. Not suitable for very small meshes where the overhead outweighs the benefits.
- **When to use:** Large meshes with many chunks, where parallel processing is beneficial.
- **When not to use:** Small meshes, or when memory bandwidth is a bottleneck.

## Background
This method is an implementation detail from the `voxelis` project by WildPixelGames [S1, unofficial]. It leverages the Rayon library for parallel iteration in Rust. The specific implementation details, including the use of atomic counters and channels, are derived from the source code of the `voxelis` repository.

## Sources
[S1] WildPixelGames/voxelis, https://github.com/WildPixelGames/voxelis, fetched 2026-10-26 (unofficial)
