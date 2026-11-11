# MapReduce-based Volume Rendering

Also called: GPU MapReduce library, GPU MapReduce volume renderer, MapReduce-based parallel volume rendering, Multi-GPU MapReduce Volume Rendering, Multi-GPU Volume Rendering using MapReduce

## Overview

MapReduce-based Volume Rendering is a multi-GPU parallel volume rendering implementation built using the MapReduce programming model, designed to handle large volumetric datasets by streaming data to GPUs and eliminating the need for custom I/O algorithms [S1, unofficial]. It splits rendering into two embarrassingly parallel steps: partial ray casting against volume bricks (Map) and compositing unsorted ray fragments into final pixels (Reduce), with Partition and Sort stages managing data distribution [S1, unofficial]. Unlike traditional in-core renderers or those requiring custom I/O pipelines, this method leverages a streaming interface that allows network communication, CPU/GPU data transfers, and disk access to overlap with computation, making it suitable for both in-core and out-of-core visualizations [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A volumetric dataset stored on disk, bricked into small pieces (Chunks). The dataset typically consists of four-byte floating-point samples [S1, unofficial].
*   **Output:** A final composited image (e.g., 512x512 pixels) [S1, unofficial].

### Data Structures

**Key-Value Pair**
The fundamental unit of data passed between stages.
*   **Key:** A 4-byte integer representing the pixel index. Calculated as:
    $$ \text{key} = y_{\text{pixel}} \times \text{width}_{\text{finalImage}} + x_{\text{pixel}} $$
    If a key $X$ exists, there is a high probability that all keys $0 \le X$ exist, allowing for efficient binning [S1, unofficial].
*   **Value:** A Ray Fragment emitted by the GPU thread.
    *   **Layout:** Homogeneous size, computed in GPU local memory.
    *   **Fields:** Color (RGB), Opacity, Depth.
    *   **Placeholder:** If a thread computes a useless key-value pair (e.g., ray does not intersect volume), it emits a placeholder that is later discarded [S1, unofficial].

**Volume Brick (Chunk)**
A small piece of the volume data.
*   **Size:** Example size is $64^3$ [S1, unofficial].
*   **Storage:** Stored in a CUDA 3-D texture with floating-point samples to enable hardware texture caches and filtering units [S1, unofficial].

### Parameters
*   **Block Size:** 16x16 threads per block in the CUDA kernel [S1, unofficial].
*   **Grid Size:** Matches the sub-image size (with padding) onto which the current chunk projects [S1, unofficial].
*   **Volume Sample Type:** Four-byte floating-point [S1, unofficial].
*   **Key Type:** Four-byte integer [S1, unofficial].

### Algorithm Steps

1.  **Bricking:** The volume data is divided into small pieces, each represented as a Chunk (brick) [S1, unofficial].
2.  **Map Stage (Ray Casting):**
    *   **Initialization:** The GPU Mapper initializes static GPU data, such as the view matrix [S1, unofficial].
    *   **Memory Allocation:** Each Chunk requests GPU memory for its volume data. The library allocates this memory [S1, unofficial].
    *   **Data Transfer:** Volume data is copied synchronously to the GPU. Asynchronous copies cannot be used because CUDA 3-D textures require synchronous copies [S1, unofficial]. GPU memory [de]allocation functions are omitted from staging functions due to high overhead and implicit synchronization [S1, unofficial].
    *   **Kernel Execution:** The ray-casting kernel executes. It is launched with a 2D grid of 2D blocks (16x16) [S1, unofficial].
    *   **Ray Casting Logic:**
        *   Rays are intersected against the brick's bounding box. Non-intersecting rays are discarded [S1, unofficial].
        *   Non-adaptive trilinear sampling is used at fixed increments [S1, unofficial].
        *   Early ray termination is applied [S1, unofficial].
        *   Front-to-back compositing is performed using a texture-based 1D transfer function to obtain the final color and opacity of each ray fragment [S1, unofficial].
    *   **Emission:** Each GPU thread emits a key-value pair (key=pixel index, value=ray fragment) [S1, unofficial].
3.  **Partition Stage:**
    *   Keys are distributed to Reducer processes in a per-pixel round-robin fashion. This method is empirically the highest-performing [S1, unofficial].
    *   The target Reducer is determined by a modulo operation on the key [S1, unofficial].
    *   Values are streamed to processes instead of being stored to disk, overcoming bottlenecks present in many other MapReduce libraries [S1, unofficial].
4.  **Sort Stage:**
    *   A specialized counting sort is used on the CPU or GPU (depending on data volume) [S1, unofficial].
    *   The sort runs in $\theta(n)$ time, where $n$ is the number of key-value pairs [S1, unofficial].
    *   The library utilizes knowledge of the minimum/maximum keys and the max number of keys per node to optimize the sort [S1, unofficial].
    *   The sort compacts like keys and arranges their respective values together [S1, unofficial].
5.  **Reduce Stage (Compositing):**
    *   Reducers execute on the CPU or GPU. Empirically, CPU compositing is quicker due to the required sort of ray fragments [S1, unofficial].
    *   All ray fragments for a given pixel are sorted in ascending depth order [S1, unofficial].
    *   Fragments are composited and blended against the background color using front-to-back compositing [S1, unofficial].
    *   Partial reduce/combine stages are omitted as they did not increase performance for this specific formulation [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_DIM = 16
KEY_SIZE_BYTES = 4
VALUE_SIZE_BYTES = 12 # Assuming 3 floats for color/opacity/depth

# Data Structures
class RayFragment:
    color: float[3]
    opacity: float
    depth: float

class KeyValuePair:
    key: int
    value: RayFragment

# Stage 1: Map (GPU Kernel)
def map_kernel(chunk: VolumeChunk, view_matrix: Matrix, output: List[KeyValuePair]):
    """
    Executes on GPU. Each thread handles one pixel of the sub-image.
    """
    # Get thread indices
    x = thread_id_x()
    y = thread_id_y()
    
    # Calculate global pixel coordinates relative to final image
    # Note: chunk projects onto a sub-image; offsets must be applied
    final_x = chunk.offset_x + x
    final_y = chunk.offset_y + y
    
    # Calculate Key
    key = final_y * final_image_width + final_x
    
    # Generate Ray
    ray = generate_ray(final_x, final_y, view_matrix)
    
    # Intersect with Brick Bounding Box
    if not ray_intersects_box(ray, chunk.bounding_box):
        # Emit placeholder to be discarded later
        output.append(KeyValuePair(key, placeholder_fragment()))
        return

    # Ray Casting Loop
    t = ray.origin
    color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    
    # Traverse chunk at fixed increments
    while t < chunk.bounding_box.max_depth:
        sample = trilinear_sample(chunk.texture_3d, ray.position_at(t))
        
        # Apply Transfer Function
        opacity = transfer_function(sample)
        rgb = transfer_color(sample)
        
        # Front-to-Back Compositing
        alpha = opacity * (1.0 - accumulated_opacity)
        color += rgb * alpha
        accumulated_opacity += alpha
        
        # Early Ray Termination
        if accumulated_opacity > 0.99:
            break
            
        t += fixed_increment
        
    # Emit Result
    output.append(KeyValuePair(key, RayFragment(color, accumulated_opacity, t)))

# Stage 2: Partition
def partition(key_value_pairs: List[KeyValuePair], num_reducers: int) -> Dict[int, List[KeyValuePair]]:
    """
    Distributes key-value pairs to reducers using per-pixel round-robin.
    """
    buckets = {i: [] for i in range(num_reducers)}
    for kv in key_value_pairs:
        reducer_id = kv.key % num_reducers
        buckets[reducer_id].append(kv)
    return buckets

# Stage 3: Sort
def sort(key_value_pairs: List[KeyValuePair]) -> List[KeyValuePair]:
    """
    Specialized counting sort. Runs in theta(n).
    Assumes keys are integers and min/max are known.
    """
    # Implementation details omitted; uses counting sort logic
    # based on known key range [min_key, max_key]
    return sorted(key_value_pairs, key=lambda kv: kv.key)

# Stage 4: Reduce (CPU)
def reduce(sorted_pairs: List[KeyValuePair]) -> Image:
    """
    Composites ray fragments for each pixel.
    """
    image = Image(width=final_image_width, height=final_image_height)
    
    # Group by key (pixel)
    # Since input is sorted, we can iterate linearly
    current_key = -1
    current_fragments = []
    
    for kv in sorted_pairs:
        if kv.key != current_key:
            # Process previous pixel
            if current_key != -1:
                composite_pixel(image, current_key, current_fragments)
            
            # Start new pixel
            current_key = kv.key
            current_fragments = []
        
        current_fragments.append(kv.value)
        
    # Process last pixel
    if current_key != -1:
        composite_pixel(image, current_key, current_fragments)
        
    return image

def composite_pixel(image: Image, key: int, fragments: List[RayFragment]):
    """
    Composites fragments for a single pixel.
    Fragments are assumed to be depth-sorted (ascending).
    """
    final_color = (0.0, 0.0, 0.0) # Background
    accumulated_opacity = 0.0
    
    for frag in fragments:
        alpha = frag.opacity * (1.0 - accumulated_opacity)
        final_color += frag.color * alpha
        accumulated_opacity += alpha
        
    x = key % final_image_width
    y = key // final_image_width
    image.set_pixel(x, y, final_color)
```

### Edge Cases and Pitfalls
*   **Synchronous Memory Copies:** Asynchronous memory copies cannot be used for volume data loading because CUDA 3-D textures require synchronous copies [S1, unofficial].
*   **Placeholder Emission:** Every GPU thread must emit a key-value pair. If the thread computes a useless pair, a placeholder is emitted and later discarded. Failure to do so may break the MapReduce framework's assumptions [S1, unofficial].
*   **GPU vs CPU Compositing:** While GPU compositing is data-parallel, CPU compositing is empirically quicker for this formulation due to the required sort of ray fragments. GPU compositing may become more efficient with hundreds or thousands of GPUs [S1, unofficial].
*   **Partial Reductions:** Partial reduce/combine stages are omitted because they do not increase performance for this specific volume renderer formulation [S1, unofficial].

## Operating characteristics

*   **Time Complexity:**
    *   **Map Stage:** Depends on the number of bricks $B$ and pixels $X$. The lower bound on generated ray fragments is $O(X)$ and the loose upper bound is $O(B \times X)$ [S1, unofficial].
    *   **Sort Stage:** $\theta(n)$, where $n$ is the number of key-value pairs [S1, unofficial].
*   **Memory Complexity:**
    *   Each single map task must fit in the main memory of the GPU. Large volumes must be bricked and mapped in stages [S1, unofficial].
    *   Each single reduce task must completely reside in the main memory of the GPU (if executed on GPU) or CPU [S1, unofficial].
*   **Performance:**
    *   Rendering a $1024^3$ floating-point volume takes under one second using 8 GPUs [S1, unofficial].
    *   Loading a $64^3$ brick from disk takes approximately 20 ms on the cluster [S1, unofficial].
    *   Transferring a $64^3$ brick to the GPU takes less than 0.2 ms (less than 1% overhead) [S1, unofficial].
    *   GPU-to-CPU transfer of finished ray fragments takes less than 2 ms [S1, unofficial].
    *   VRAM is more than 10X faster than CPU DRAM [S1, unofficial].
*   **Scaling:**
    *   The system scales with respect to the size of the volume and the number of GPUs [S1, unofficial].
    *   For a given image of $X$ pixels with $Y$ nodes and $B$ bricks, each of the $Y$ nodes must make $Y-1$ communication requests [S1, unofficial].
*   **Strengths:**
    *   Eliminates the need for custom I/O algorithms.
    *   Easy to program for and scales well.
    *   Provides inherent GPU support out-of-the-box, unlike ParaView and VisIt [S1, unofficial].
*   **Limitations:**
    *   Does not provide fault tolerance, advanced scheduling, or a distributed file system [S1, unofficial].
    *   Requires synchronous memory copies for 3D textures, limiting some optimization opportunities [S1, unofficial].

## Background

*   **Origin:** The method was presented by Jeff A. Stuart, Cheng-Kai Chen, Kwan-Liu Ma, and John Owens at the 19th ACM International Symposium in June 2010 [S1, unofficial].
*   **Hardware:** Tested on the Accelerator Cluster (AC) at NCSA with up to 32 GPUs. Each node has a quad-core CPU, 8 GB SDRAM, and a Tesla C1090 with four logical GPUs each. Connected via QDR Infiniband. Linux 2.6 kernel, CUDA 3.0 toolkit, NVIDIA 195 drivers [S1, unofficial].
*   **Datasets:** Skull, Supernova (resolutions $128^3, 256^3, 512^3, 1024^3$), and Plume ($512 \times 512 \times 2048$) [S1, unofficial].
*   **Comparison:** Unlike CellMR, which streams map data in small pieces and performs partial reductions to eliminate I/O, this volume renderer rarely sees benefits from partial reductions due to its formulation [S1, unofficial].

## Sources

[S1] (PDF) Multi-GPU volume rendering using MapReduce - ResearchGate, https://www.researchgate.net/publication/202260516_Multi-GPU_volume_rendering_using_MapReduce, fetched 2026-10-31 (unofficial)
