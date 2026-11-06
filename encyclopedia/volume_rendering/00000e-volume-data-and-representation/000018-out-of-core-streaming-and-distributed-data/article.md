# Out-of-core, streaming and distributed data

Also called: Out-of-core rendering, Volume streaming, Distributed volume rendering

## Overview
This family addresses the rendering of volumetric datasets that exceed the memory capacity of a single processing unit or the bandwidth limits of the system bus [S1, S2, S4, unofficial]. It solves the scalability bottleneck inherent in single-node rendering by partitioning data or computation across multiple nodes, or by streaming data from disk to GPU in chunks [S1, S2, S4, unofficial]. What sets this family apart from [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) is the explicit handling of data movement and partitioning strategies, whereas dense grids focus on in-memory organization. It differs from [Compression](4-compression.md) by focusing on temporal/spatial streaming rather than static storage reduction, and from [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md) by prioritizing data delivery over visual approximation, though they often combine.

## Shared principles
All methods in this family share the following core mechanisms:
1.  **Decomposition:** The input volume or mesh is partitioned into subsets. This can be spatial (sub-volumes) or topological (mesh cells) [S1, S2, S4, unofficial].
2.  **Distribution:** Subsets are assigned to distinct processing units (MPI ranks, cluster nodes) [S1, S2, S4, unofficial].
3.  **Local Rendering:** Each unit renders its assigned subset independently, typically using ray casting or projection [S1, S2, S4, unofficial].
4.  **Compositing/Assembly:** Results from all units are combined to form the final image. This occurs either via pixel-based accumulation over the network (e.g., SDVR) or image-based assembly by a root process (e.g., MPI 1D/2D) [S1, S2, S4, unofficial].

## Subtopics

### [Sectioning](1636-sectioning.md)
Partitions a volume into segments to constrain on-chip FIFO buffer usage, solving limited hardware memory for intermediate ray-casting values by off-loading to external memory. Unlike [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md) which uses hardware buffers, or [Chunk-based spatial partitioning](3862-chunk-based-spatial-partitioning.md) which divides space for load balancing, Sectioning manages recirculation within a single pass for exact precision.

### [VolumeLoader](2045-volumeloader.md)
A generic ingestion method handling loading, resampling, and coordinate alignment for medical formats. It solves heterogeneous file conversion to device-ready tensors while managing memory via automatic downscaling. It differs from specialized loaders by providing a unified interface with overflow guards rather than focusing on specific headers.

### [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md)
An irregular volume rendering algorithm streaming data to solve visibility sorting for unstructured grids by combining object-space and image-space processing. It offers more accurate sorting than centroid sort methods like HAPT while avoiding the high memory overhead of storing the entire volume on the GPU.

### [Bandwidth-based Streaming Interval Calculation](3128-bandwidth-based-streaming-interval-calculation.md)
A method to calculate the minimum time interval between data fetches and simulation slice skipping based on system bandwidth limits to ensure real-time streaming from HDD to GPU.

### [Chunk-based spatial partitioning](3862-chunk-based-spatial-partitioning.md)
Divides space into chunks for load balancing and efficient data access.

### [Distributed computing and load balancing](5624-distributed-computing-and-load-balancing.md)
Techniques for rendering datasets exceeding single-machine capacity by distributing workload across multiple nodes. It solves scalability bottlenecks by partitioning data or computation. What sets this family apart from [Out-of-core, streaming and distributed data](8-out-of-core-streaming-and-distributed-data.md) is the explicit focus on simultaneous multi-node processing and load balancing, whereas out-of-core methods primarily address single-node memory limits via disk swapping [G].

### [MapReduce-based Volume Rendering](5963-mapreduce-based-volume-rendering.md)
A multi-GPU parallel implementation using the MapReduce model, designed to handle large datasets by streaming data to GPUs and eliminating the need for custom I/O algorithms.

### [Out-of-core brick-based volume rendering](6288-out-of-core-brick-based-volume-rendering.md)
A system streaming multi-GB datasets over HTTP with a fixed VRAM budget by decoupling logical 3D grid coordinates from physical GPU memory using a brick pyramid and indirection texture.

### Out-of-Core Array (OOCArray)
- **Definition:** A templated class that manages large arrays by buffering data in memory and paging it to a temporary file, enabling out-of-core storage for datasets exceeding RAM [S1, unofficial].
- **Step:** Resize: Extends the backing file by seeking to `sizeof(Data) * sz` using `_fseeki64` (Windows) or `fseek` (Unix) [S1, unofficial].
- **Layout:** Template parameters: `Data` (element type), `BufferSize` (default 64). Fields: `FILE* _fp` (file pointer), `char _fileName[512]` (temp file path), `size_t _currentBufferIndex` (active buffer page), `Data _buffer[BufferSize]` (in-memory page) [S1, unofficial].
- **Pitfall:** Not thread-safe. Uses `mktemp`/`_mktemp` which may have race conditions or portability issues. File path separator handling differs between Windows (`\`) and Unix (`/`) [S1, unofficial].

### Subvolume Updates
- **Definition:** A technique allowing transfer of any subvolume (from single voxel to full volume) between host memory and Voxel Memory, enabling dynamic updates and partial loading [S2, unofficial].
- **Step:** Update subvolumes in-between frames to achieve 4D animation effects or load sections of a larger supervolume piece-wise. Select any subvolume of the loaded volume for rendering to reduce rendering times on an area of interest [S2, unofficial].
- **Parameter:** Subvolumes can have as little as one voxel on each edge [S2, unofficial].

### Indexed Leaf for Stream Processing
- **Definition:** A wrapper for a Leaf node that includes its index in the stream, used for building the SVO [S3, unofficial].
- **Layout:** `IndexedLeaf` contains: `leaf` (Leaf), `index` (unsigned long long int) [S3, unofficial].

### Generic Data Producer Model
- **Definition:** A producer/consumer architecture where a generic data producer provides a request interface to obtain volumetric data based on position, size, and resolution, outputting to a specified memory zone [S4, unofficial].
- **Parameter:** Scene Coordinates: Position and size are defined between 0.0 and 1.0 on each axis. Output Memory Zone: Can be a standard memory zone or an OpenGL buffer object for direct GPU transfer [S4, unofficial].

## Background
The need for out-of-core and streaming techniques arises from the exponential growth of volumetric datasets in medical imaging and scientific visualization, which often exceed the memory capacity of available hardware [S2, S4, unofficial]. Early systems like VolumePro introduced hardware-assisted streaming and subvolume updates to handle large datasets on limited hardware (e.g., 128 MB Voxel Memory) by transferring subvolumes between host and device memory [S2, unofficial]. Subsequent research expanded these concepts to distributed computing environments, utilizing MPI and MapReduce models to scale rendering across clusters [S1, S4, unofficial]. The Generic Data Producer Model further abstracted data access, allowing for flexible integration with various rendering pipelines and GPU memory management [S4, unofficial].

## Sources
[S1] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S2] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S3] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S4] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
