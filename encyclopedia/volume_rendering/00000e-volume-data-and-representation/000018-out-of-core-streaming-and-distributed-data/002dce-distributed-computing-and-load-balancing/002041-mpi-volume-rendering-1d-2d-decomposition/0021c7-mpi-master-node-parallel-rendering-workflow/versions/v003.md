# MPI Master Node Parallel Rendering Workflow

Also called: MPI Image Assembly, Master-Slave Volume Rendering

## Overview
This method implements a master-slave parallel volume rendering architecture where a single master node coordinates the spawning of slave processes, distributes rendering parameters, and assembles the final image from per-slice fragments received over TCP [S1, unofficial]. It is distinguished from sibling methods like [MPI Parallel Slice-Based Volume Rendering](3827-mpi-parallel-slice-based-volume-rendering.md) by explicitly handling domain decomposition and image concatenation within a pre-defined master-slave workflow abstraction, rather than embedding the logic within the rendering loop itself [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Command-line arguments for `width`, `height`, `slave_size`, `thread_nums`, and `vifo_path` (volume data path) [S1, unofficial].
- **Outputs:** A JPEG-encoded image file written to disk [S1, unofficial].

**Data Structures:**
- `ibuf`: A 2D array of type `char` with layout `ibuf[slave_size][ibuf_size]`, where `ibuf_size` is calculated as $\lceil \text{height} / \text{slave\_size} \rceil \times \text{width} \times 3$ [S1, unofficial]. This buffer stores the raw RGB pixel data received from each slave.
- `slave_request`: An array of `MPI_Request` objects of size `slave_size` for non-blocking receives [S1, unofficial].
- `msg_buf`: A character buffer of size `MSG_BUF_SIZE` used for broadcasting control messages [S1, unofficial].

**Steps:**
1. **Initialization and Spawning:**
   - Parse command-line arguments to extract volume dimensions, slave count, and file path [S1, unofficial].
   - Construct a message string containing dimensions, brick sizes, bounding box, `mmu_ratio`, and file path [S1, unofficial].
   - Spawn `slave_size` slave processes using `MPI_Comm_spawn` with the command `./VolumeRendererServer_slave` [S1, unofficial].
   - Send the transfer function data to all slaves via `send_tf1d` [S1, unofficial].

2. **Buffer Allocation and Setup:**
   - Calculate `ibuf_size` using the formula:
     $$ \text{ibuf\_size} = \lceil \frac{\text{height}}{\text{slave\_size}} \rceil \times \text{width} \times 3 $$
   - Allocate `ibuf[slave_size][ibuf_size]` and `slave_request[slave_size]` [S1, unofficial].
   - Set up a TCP acceptor on a designated `PORT` (implementation detail: the source code shows TCP usage in `Message.h` but the master loop uses MPI for control and presumably TCP for image data, though the specific TCP accept logic is not fully detailed in the provided snippet, the assembly step implies receiving data) [S1, unofficial]. *Note: The source code snippet shows `MPI_Irecv` for image data, implying MPI is used for the image transfer in the main loop, while TCP is used in the `Message` class for other communications or slave initialization. The "How it works" note mentions "receiving per-slice image data... transmitting over TCP", but the code uses `MPI_Irecv`. We follow the code's explicit MPI receive pattern for the main assembly loop.*

3. **Rendering Loop:**
   - Iterate `image_nums` times (default 5) [S1, unofficial].
   - Broadcast a control message `msg_buf` to all slaves using `MPI_Send` (or `MPI_Bcast` if intended for all, but code says `MPI_Send` to 0? The code snippet says `MPI_Send(&image_nums, ...)` and `MPI_Send` for `msg_buf`. The note says "Broadcast msg_buf". We assume broadcast semantics for control) [S1, unofficial].
   - Record start time $t_1$ [S1, unofficial].
   - Issue non-blocking receives for each slave $i$ (from $0$ to `slave_size`-1) into `ibuf[i]`:
     $$ \text{MPI\_Irecv}(ibuf[i], \text{ibuf\_size}, \text{MPI\_CHAR}, i, i, \text{slave\_comm}, \&slave\_request[i]) $$
   - Wait for all receives to complete using `MPI_Waitall` [S1, unofficial].
   - Record end time $t_2$ [S1, unofficial].
   - Print rendering time $t_2 - t_1$ [S1, unofficial].

4. **Image Assembly:**
   - Initialize `dst_base` pointer to the start of the final image buffer `ibuf_dst` (size `height * width * 3`) [S1, unofficial].
   - Iterate `row` from $0$ to `height`-1:
     - Determine the slave responsible for this row:
       $$ \text{slave\_no} = \text{row} \pmod{\text{slave\_size}} $$
       $$ \text{slave\_row} = \lfloor \text{row} / \text{slave\_size} \rfloor $$
     - Copy `width * 3` bytes from `ibuf[slave_no]` at offset `slave_row * width * 3` to `dst_base` [S1, unofficial].
     - Advance `dst_base` by `width * 3` bytes [S1, unofficial].

5. **Output:**
   - Encode the assembled `ibuf_dst` to JPEG format [S1, unofficial].
   - Write the JPEG size and buffer to a socket (if using TCP for final output) or write directly to disk as shown in `write_image_disk` [S1, unofficial].
   - Close connections and finalize MPI [S1, unofficial].

**Pitfalls:**
- The image assembly assumes a round-robin row distribution. If `height` is not divisible by `slave_size`, the last rows are distributed unevenly, but the modulo and division logic handles this correctly by assigning remaining rows to the first few slaves in the round-robin sequence [S1, unofficial].
- Precision issues may arise if `ibuf_size` calculation does not account for padding or alignment requirements of the underlying MPI implementation, though `char` type minimizes this risk [G].

## Subtopics

### [MPI Master-Slave Octree Volume Rendering](2744-mpi-master-slave-octree-volume-rendering.md)
Uses spatial octree encoding to partition hexahedral elements, allowing efficient traversal of empty space and one-time distribution of the octree subtree structure.

### [MPI Parallel Slice-Based Volume Rendering](3827-mpi-parallel-slice-based-volume-rendering.md)
Decomposes the volume along the slicing axis (typically Z) among processes, handling domain decomposition and image concatenation within the rendering loop itself, operating on uniform slice blocks rather than hierarchical structures.

## Operating characteristics
- **Time Complexity:** The master node's image assembly is $O(\text{height} \times \text{width})$ for copying pixels. The rendering time is dominated by the slowest slave process.
- **Memory Complexity:** The master node requires $O(\text{slave\_size} \times \lceil \text{height}/\text{slave\_size} \rceil \times \text{width} \times 3)$ bytes for the receive buffers, plus $O(\text{height} \times \text{width} \times 3)$ for the final image. This is roughly $O(\text{height} \times \text{width} \times 3)$, effectively doubling the memory requirement for the image buffer on the master compared to a single-process render [S1, unofficial].
- **Strengths:** Simple master-slave abstraction; explicit control over parameter distribution; handles uneven row distributions gracefully [S1, unofficial].
- **Limitations:** The master node becomes a bottleneck for image assembly and control message broadcasting; TCP/MPI hybrid communication may introduce overhead; not given in the sources for specific hardware performance metrics.

## Background
This method originates from an unofficial GitHub repository `hopexn/VolumeRenderingParallel` [S1, unofficial]. It is designed to coordinate parallel volume rendering by spawning slave processes via MPI, distributing transfer functions, and assembling the final image from per-slice data [S1, unofficial].

## Sources
[S1] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
