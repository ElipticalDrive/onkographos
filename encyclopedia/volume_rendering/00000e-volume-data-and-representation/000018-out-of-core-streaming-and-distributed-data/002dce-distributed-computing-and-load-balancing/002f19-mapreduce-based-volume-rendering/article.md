# MapReduce-based Volume Rendering

Also called: GPU MapReduce library, GPU MapReduce volume renderer, MapReduce-based parallel volume rendering, Multi-GPU MapReduce Volume Rendering, Multi-GPU Volume Rendering using MapReduce

## Overview
MapReduce-based Volume Rendering is a multi-GPU parallel volume rendering implementation built using the MapReduce programming model, designed to handle large volumetric datasets by streaming data to GPUs and eliminating the need for custom I/O algorithms [S1, unofficial]. It splits rendering into two embarrassingly parallel steps: partial ray casting against volume bricks (Map) and compositing unsorted ray fragments into final pixels (Reduce), with Partition and Sort stages managing data distribution [S1, unofficial]. It sets itself apart from siblings like [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md) by using a key-value pair streaming model rather than explicit MPI message passing for compositing, and from [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md) by decoupling the spatial decomposition from the compositing logic via a global pixel-index key space [G].

## How it works
The method operates on a cluster of GPUs, treating the volume as a collection of bricks (chunks) and the final image as a set of pixels [S1, unofficial]. The workflow consists of four stages: Map, Partition, Sort, and Reduce [S1, unofficial].

### Inputs and Outputs
*   **Input**: A large volumetric dataset stored on disk, pre-bricked into small pieces (Chunks) [S1, unofficial]. Each Chunk contains floating-point samples (4-byte) [S1, unofficial].
*   **Output**: A final composited 2D image (e.g., 512x512 pixels) [S1, unofficial].
*   **Intermediate Data**: Key-value pairs where the Key is a 4-byte integer representing a pixel index, and the Value is a Ray Fragment (color, opacity, depth) [S1, unofficial].

### Data Structures
*   **Chunk**: A brick of volume data [S1, unofficial]. The size is small enough to fit in GPU memory (e.g., $64^3$) [S1, unofficial].
*   **Key**: A 4-byte integer representing the pixel index: $K = y \times W + x$, where $W$ is the final image width and $(x, y)$ is the pixel coordinate [S1, unofficial].
*   **Value (Ray Fragment)**: A homogeneous structure containing color (RGB), opacity ($\alpha$), and depth ($d$) [S1, unofficial]. Computed in GPU local memory [S1, unofficial].
*   **Placeholder**: A dummy key-value pair emitted by GPU threads that do not intersect the volume or contribute to the image, to be discarded later [S1, unofficial].

### Numbered Steps

1.  **Initialization**:
    *   The system initializes static GPU data, such as the view matrix, on each GPU Mapper [S1, unofficial].
    *   Mappers are assigned specific Chunks of volume data [S1, unofficial].

2.  **Map Stage (Ray Casting)**:
    *   For each Chunk, the Mapper requests GPU memory for the volume data [S1, unofficial].
    *   The volume data is copied **synchronously** to the GPU [S1, unofficial]. This is required because CUDA 3-D textures do not support asynchronous memory copies [S1, unofficial].
    *   The Mapper executes a CUDA ray-casting kernel [S1, unofficial].
        *   The kernel is launched with a 2D grid of 2D blocks [S1, unofficial].
        *   Block size is $16 \times 16$ threads [S1, unofficial].
        *   Grid dimensions match the sub-image size (the projection of the Chunk onto the final image) with padding [S1, unofficial].
        *   Each thread generates a ray [S1, unofficial].
        *   Rays are intersected against the Chunk's bounding box [S1, unofficial]. Non-intersecting rays are discarded (emitting a placeholder) [S1, unofficial].
        *   Intersecting rays perform non-adaptive trilinear sampling at fixed increments through the 3D texture [S1, unofficial].
        *   Early ray termination is used [S1, unofficial].
        *   Front-to-back compositing is performed locally within the Chunk using a texture-based 1D transfer function to obtain a final color and opacity for the ray segment [S1, unofficial].
        *   The resulting Ray Fragment is emitted as a key-value pair $(K, V)$, where $K$ is the pixel index [S1, unofficial].

3.  **Partition Stage**:
    *   Key-value pairs from all Mappers are collected [S1, unofficial].
    *   The Key $K$ is used to determine the target Reducer process [S1, unofficial].
    *   Distribution is **per-pixel round-robin** [S1, unofficial]. The target Reducer index $R$ is calculated as:
        $$ R = K \pmod P $$
        where $P$ is the number of Reducer processes (typically equal to the number of GPUs or nodes) [S1, unofficial].
    *   This method is empirically the highest-performing distribution strategy [S1, unofficial].
    *   Values are streamed asynchronously to the appropriate Reducer processes, avoiding disk storage of intermediate keys [S1, unofficial].

4.  **Sort Stage**:
    *   Each Reducer receives a stream of key-value pairs for its assigned subset of pixels [S1, unofficial].
    *   The pairs are sorted by Key [S1, unofficial].
    *   A specialized **counting sort** is used, which runs in $\Theta(n)$ time, where $n$ is the number of key-value pairs [S1, unofficial].
    *   The sort is performed on the CPU or GPU depending on data volume [S1, unofficial]. The library knows the minimum and maximum keys for each node, allowing for efficient binning [S1, unofficial].
    *   Sorting ensures that all ray fragments for a given pixel are contiguous and sorted by depth (ascending) [S1, unofficial].

5.  **Reduce Stage (Compositing)**:
    *   Each Reducer iterates through the sorted key-value pairs [S1, unofficial].
    *   For each unique Key (pixel), all associated Ray Fragments are composited using front-to-back compositing against the background color [S1, unofficial].
    *   The final color and opacity for the pixel are computed [S1, unofficial].
    *   Empirical results indicate that CPU compositing is quicker than GPU compositing for this specific formulation, primarily due to the overhead of sorting on the GPU and the data-parallel nature of the final pixel blending [S1, unofficial].
    *   The final pixel values are transferred from GPU to CPU (if computed on GPU) or directly available on CPU [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 16
IMAGE_WIDTH = 512
NUM_REDUCERS = 8  # Example: 8 GPUs

# Data Structures
class RayFragment:
    def __init__(self, color, opacity, depth):
        self.color = color
        self.opacity = opacity
        self.depth = depth

class Chunk:
    volume_data: list  # 3D texture data (flattened or nested)
    bounding_box_min: tuple
    bounding_box_max: tuple
    step_size: float

# Helper: Generate ray for pixel (x, y) [G]
def generate_ray(x, y, view_matrix, image_width):
    # Standard inverse projection/view transform [G]
    u = (2.0 * x / image_width) - 1.0
    v = 1.0 - (2.0 * y / image_width)
    origin = (0.0, 0.0, 0.0)
    direction = (u, v, 1.0)
    return {"origin": origin, "direction": direction}

# Helper: Intersect ray with axis-aligned bounding box [G]
def intersect_box(ray, bbox):
    t_min = (bbox[0] - ray["origin"][0]) / ray["direction"][0]
    t_max = (bbox[3] - ray["origin"][0]) / ray["direction"][0]
    if t_min > t_max: t_min, t_max = t_max, t_min
    # Simplified 3D AABB check [G]
    return True  # Assume intersection for pseudocode flow

# Helper: Sample 3D texture with trilinear interpolation [G]
def sample_3d_texture(volume_data, t):
    # Standard trilinear lookup [G]
    return 0.5  # Placeholder sample value

# Helper: Apply 1D transfer function [G]
def transfer_function(sample):
    # Standard opacity/color mapping [G]
    return (1.0, 0.8)

# Helper: Counting sort by key, then by depth [G]
def counting_sort_by_key_and_depth(fragments):
    # Bucket sort for known integer key range [G]
    return sorted(fragments, key=lambda f: (f[0], f[1].depth))

# Map Stage: Executed on GPU
def map_chunk(chunk: Chunk, view_matrix, image_width: int):
    fragments = []
    grid_width = image_width
    grid_height = image_width
    
    for y in range(grid_height):
        for x in range(grid_width):
            ray = generate_ray(x, y, view_matrix, image_width)
            
            if not intersect_box(ray, (chunk.bounding_box_min, chunk.bounding_box_max)):
                key = y * image_width + x
                fragments.append((key, RayFragment((0.0, 0.0, 0.0), 0.0, 0.0)))
                continue
            
            color = (0.0, 0.0, 0.0)
            alpha = 0.0
            depth = 0.0
            t = 0.0
            
            while t < 1.0 and alpha < 1.0:
                sample = sample_3d_texture(chunk.volume_data, t)
                if sample > 0.0:
                    c, a = transfer_function(sample)
                    color = (
                        color[0] + c[0] * a * (1.0 - alpha),
                        color[1] + c[1] * a * (1.0 - alpha),
                        color[2] + c[2] * a * (1.0 - alpha)
                    )
                    alpha = alpha + a * (1.0 - alpha)
                    depth = t
                t += chunk.step_size
            
            key = y * image_width + x
            fragments.append((key, RayFragment(color, alpha, depth)))
            
    return fragments

# Partition Stage: Executed on CPU/Coordinator
def partition(key: int, num_reducers: int) -> int:
    return key % num_reducers

# Sort Stage: Executed on CPU/GPU
def sort_fragments(fragments: list) -> list:
    return counting_sort_by_key_and_depth(fragments)

# Reduce Stage: Executed on CPU (Empirically faster)
def reduce_fragments(sorted_fragments: list) -> dict:
    final_image = {}
    current_key = -1
    current_fragments = []
    
    for key, fragment in sorted_fragments:
        if key != current_key:
            if current_key != -1:
                final_image[current_key] = composite_pixel(current_fragments)
            current_key = key
            current_fragments = [fragment]
        else:
            current_fragments.append(fragment)
            
    if current_key != -1:
        final_image[current_key] = composite_pixel(current_fragments)
        
    return final_image

def composite_pixel(fragments: list) -> RayFragment:
    final_color = (0.0, 0.0, 0.0)
    final_alpha = 0.0
    
    for frag in fragments:
        c, a, d = frag.color, frag.opacity, frag.depth
        final_color = (
            final_color[0] + c[0] * a * (1.0 - final_alpha),
            final_color[1] + c[1] * a * (1.0 - final_alpha),
            final_color[2] + c[2] * a * (1.0 - final_alpha)
        )
        final_alpha = final_alpha + a * (1.0 - final_alpha)
        
    return RayFragment(final_color, final_alpha, fragments[-1].depth)
```

### Parameters and Typical Values
*   **Volume Resolution**: $1024^3$ floating-point samples [S1, unofficial].
*   **GPU Count**: 8 GPUs for benchmarked performance [S1, unofficial].
*   **Image Resolution**: 512x512 pixels [S1, unofficial].
*   **Block Size**: $16 \times 16$ threads per block in CUDA kernel [S1, unofficial].
*   **Key Type**: 4-byte integer [S1, unofficial].
*   **Sample Type**: 4-byte floating-point [S1, unofficial].

### Edge Cases and Pitfalls
*   **Synchronous Memory Copies**: CUDA 3-D textures require synchronous memory copies for volume data loading [S1, unofficial]. Asynchronous copies cannot be used [S1, unofficial].
*   **GPU Memory Allocation Overhead**: GPU memory [de]allocation functions are omitted from staging functions due to high overhead and implicit synchronization [S1, unofficial].
*   **Placeholder Emission**: Every GPU thread must emit a key-value pair [S1, unofficial]. Useless pairs (non-intersecting rays) emit a placeholder that is discarded later, adding to the data volume [S1, unofficial].
*   **No Partial Reduce**: Partial reduce/combine stages were omitted because they did not increase performance for this specific volume renderer formulation [S1, unofficial].
*   **Key Existence Assumption**: If a key $X$ exists, it is assumed that all keys $0 \le X$ have a high probability of existing, which simplifies partitioning and binning [S1, unofficial].

## Operating characteristics
*   **Time Complexity**:
    *   Map Stage: $O(B \times X)$, where $B$ is the number of bricks and $X$ is the number of pixels (worst-case, every ray casts through every brick) [S1, unofficial]. Lower bound is $O(X)$ [S1, unofficial].
    *   Sort Stage: $\Theta(n)$, where $n$ is the number of key-value pairs [S1, unofficial].
    *   Reduce Stage: $O(n)$ [S1, unofficial].
*   **Memory Complexity**:
    *   Requires sufficient GPU memory to hold one Chunk (brick) at a time [S1, unofficial].
    *   Intermediate key-value pairs are streamed, not stored on disk, reducing I/O bottlenecks [S1, unofficial].
*   **Performance**:
    *   Rendering a $1024^3$ floating-point volume takes under one second using 8 GPUs [S1, unofficial].
    *   Loading a $64^3$ brick from disk takes ~20 ms [S1, unofficial].
    *   Transferring a $64^3$ brick to GPU takes <0.2 ms (<1% overhead) [S1, unofficial].
    *   GPU-to-CPU transfer of finished ray fragments takes <2 ms [S1, unofficial].
*   **Scaling**:
    *   Scales well with the number of GPUs and data size [S1, unofficial].
    *   Doubling the number of GPUs approximately doubles performance [S1, unofficial].
*   **Strengths**:
    *   Eliminates the need for custom I/O algorithms [S1, unofficial].
    *   High throughput due to GPU texture sampling [S1, unofficial].
    *   Simple programming model (MapReduce) [S1, unofficial].
    *   Supports both in-core and out-of-core visualization [S1, unofficial].
*   **Limitations**:
    *   Lacks fault tolerance, advanced scheduling, and distributed file system support compared to general-purpose MapReduce libraries [S1, unofficial].
    *   CPU compositing is empirically faster than GPU compositing for this formulation, limiting GPU utilization in the Reduce phase [S1, unofficial].
    *   Synchronous memory copies are required for volume data loading because CUDA 3-D textures do not support asynchronous copies [S1, unofficial].
*   **When to Use**:
    *   Interactive visualization of large-scale volumetric data [S1, unofficial].
    *   When GPU support is required out-of-the-box [S1, unofficial].
    *   When simplicity of implementation is prioritized over fault tolerance [S1, unofficial].
*   **When Not to Use**:
    *   When fault tolerance is critical [S1, unofficial].
    *   When the dataset is small enough to fit in a single GPU memory without bricking [S1, unofficial].
    *   When partial reductions are beneficial (not the case for this renderer) [S1, unofficial].

## Background
The method was introduced by Jeff A. Stuart, Cheng-Kai Chen, Kwan-Liu Ma, and John D. Owens in 2010 at the 19th ACM International Symposium on Large Data Analysis and Rendering (LADAR) [S1, unofficial]. It was developed to address the lack of out-of-the-box GPU support in existing visualization tools like ParaView and VisIt, and to provide a scalable infrastructure for multi-GPU volume rendering [S1, unofficial]. The work was tested on the Accelerator Cluster (AC) at the National Center for Supercomputing Applications (NCSA), using up to 32 GPUs (Tesla C1090) connected via QDR Infiniband [S1, unofficial]. The datasets used included Skull, Supernova, and Plume, with resolutions up to $1024^3$ [S1, unofficial]. The approach builds on the MapReduce programming model, originally popularized by Google, but modifies it to support GPU streaming and eliminate disk-based intermediate storage [S1, unofficial].

## Sources
[S1] Multi-GPU volume rendering using MapReduce, https://www.researchgate.net/publication/202260516_Multi-GPU_volume_rendering_using_MapReduce, fetched 2026-10-31 (unofficial)
