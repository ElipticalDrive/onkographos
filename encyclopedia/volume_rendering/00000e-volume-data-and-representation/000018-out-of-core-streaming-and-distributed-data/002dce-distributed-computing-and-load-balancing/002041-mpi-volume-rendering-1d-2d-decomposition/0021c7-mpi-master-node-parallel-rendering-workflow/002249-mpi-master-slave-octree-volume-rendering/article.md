# MPI Master-Slave Octree Volume Rendering

Also called: Octree-based Data Distribution, Parallel Pipeline, Parallel Pipeline with Octree-based Partitioning, Parallel Volume Rendering with MPI and Octree Traversal

## Overview
This method implements a parallel volume rendering architecture where input processors read data from a parallel file system, preprocess it, and distribute data blocks to rendering processors via a pipeline [S1]. It is distinguished from sibling methods like [MPI Parallel Slice-Based Volume Rendering](3827-mpi-parallel-slice-based-volume-rendering.md) by using a spatial octree encoding of the raw data to partition hexahedral elements, allowing for efficient traversal of empty space and one-time distribution of the octree subtree structure [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Raw volume data files (stored on a parallel file system), transfer function data (`tf1d`), volume metadata (dimensions `bmin`, `bmax`, `mmu_ratio`, file path) [S1][S2, unofficial].
*   **Outputs:** Rendered image frames, assembled from partial images contributed by slave processes [S1][S2, unofficial].

**Data Structures:**
*   **Octree Subtree:** A data structure representing a portion of the spatial domain. It is associated with a block of hexahedral elements and delivered to the assigned rendering processor only once at the beginning, as the structure remains constant across all time steps [S1].
*   **`vifo` (Volume Info):** Contains `Vector3i dims` (dimensions), `Vector3i blk_size` (block size), `Vector3f bmin` (bounding min), `Vector3f bmax` (bounding max), `float mmu_ratio`, and `string volume_file` [S2, unofficial].
*   **Image Buffer (`ibuf`):** A 2D array `ibuf[slave_size][ibuf_size]` where `ibuf_size = ceil(height / slave_size) * width * 3`. This stores partial images from each slave [S2, unofficial].
*   **Transfer Function Buffer (`tf1d_buf`):** A buffer of size `TF1D_BUF_SIZE` (1024 * 10 bytes) used to broadcast transfer function data [S2, unofficial].

**Steps:**
1.  **Preprocessing (Input Processors):**
    *   Generate a spatial octree encoding of the raw data in a one-time preprocessing step [S1].
    *   Use the octree and a workload estimation method to distribute blocks of hexahedral elements among rendering processors [S1].
    *   Each block is associated with a subtree of the global octree [S1].
    *   Perform preprocessing calculations: quantization (32-bit to 8-bit), central differencing for gradient vectors (lighting), and one-side differencing for temporal domain enhancement [S1]. These tasks are conducted at input processors to avoid data replication and leverage parallel pipelining [S1].
2.  **Master Initialization (Master Process):**
    *   Parse command line arguments into a string `msg` containing volume dimensions (`bmin`, `bmax`), `mmu_ratio`, and volume file path [S2, unofficial].
    *   Allocate `slave_argv` with this message [S2, unofficial].
    *   Spawn `slave_size` instances of the slave renderer (`./VolumeRendererServer_slave`) using `MPI_Comm_spawn` [S2, unofficial].
3.  **Parameter Distribution:**
    *   Send the transfer function data (`tf1d`) to all slaves via `MPI_Bcast` or a dedicated send function `send_tf1d` [S2, unofficial].
4.  **Data Distribution (Input to Renderers):**
    *   Deliver the octree subtree associated with each block to the assigned rendering processor only once at the start [S1].
    *   Use non-blocking send and receive operations for block distribution to overlap communication with computation [S1].
5.  **Rendering (Slave Processes):**
    *   Traverse the Octree (OTree) structure to find empty/non-empty blocks [S2, unofficial].
    *   Compute ray segments intersecting non-empty blocks [S2, unofficial].
    *   Render those segments and produce a partial image [S2, unofficial].
6.  **Image Assembly (Master Process):**
    *   Allocate `slave_request` and `slave_status` arrays for asynchronous MPI receives [S2, unofficial].
    *   Receive partial images from slaves into `ibuf` [S2, unofficial].
    *   Assemble the full image from `ibuf` [S2, unofficial].
    *   Encode the image to JPEG and transmit over TCP [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import mpi4py.MPI as MPI
import numpy as np
import math

# Constants from [S2, unofficial]
TF1D_BUF_SIZE = 1024 * 10
MSG_BUF_SIZE = 1024

def master_process():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    
    if rank == 0:
        # 1. Parse arguments into msg string [S2, unofficial]
        # msg contains bmin, bmax, mmu_ratio, volume_file
        msg = parse_command_line_args()
        
        # 2. Spawn slaves [S2, unofficial]
        slave_argv = [msg]
        slave_size = get_slave_count()
        slave_comm, slaves = MPI.COMM_SELF.Spawn(
            "./VolumeRendererServer_slave", 
            slave_argv, 
            slave_size
        )
        
        # 3. Send transfer function [S2, unofficial]
        tf1d_buf = np.zeros(TF1D_BUF_SIZE, dtype=np.uint8)
        load_transfer_function(tf1d_buf)
        slave_comm.Bcast(tf1d_buf, root=0)
        
        # 4. Allocate image buffer [S2, unofficial]
        height, width = get_image_dimensions()
        ibuf_size = math.ceil(height / slave_size) * width * 3
        ibuf = np.zeros((slave_size, ibuf_size), dtype=np.uint8)
        slave_request = [None] * slave_size
        slave_status = [MPI.Status()] * slave_size
        
        # 5. Receive partial images [S2, unofficial]
        for i in range(slave_size):
            slave_comm.Irecv(ibuf[i], source=i, tag=i, request=slave_request[i])
            
        MPI.Waitall(slave_request, slave_status)
        
        # 6. Assemble and encode [S2, unofficial]
        full_image = assemble_image(ibuf, height, width)
        encode_to_jpeg(full_image)
        transmit_over_tcp(full_image)
        
        slave_comm.Free()

def slave_process():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    
    # 1. Receive metadata and transfer function [S2, unofficial]
    msg = receive_message()
    tf1d_buf = np.zeros(TF1D_BUF_SIZE, dtype=np.uint8)
    comm.Bcast(tf1d_buf, root=0)
    
    # 2. Parse volume info [S2, unofficial]
    vifo = parse_vifo(msg)
    
    # 3. Receive octree subtree and data block [S1]
    # Delivered once at the beginning
    octree_subtree = receive_octree_subtree()
    data_block = receive_data_block()
    
    # 4. Render [S2, unofficial]
    # Traverse OTree to find empty/non-empty blocks
    # Compute ray segments intersecting non-empty blocks
    partial_image = render_segments(data_block, octree_subtree, tf1d_buf, vifo)
    
    # 5. Send partial image back [S2, unofficial]
    comm.Send(partial_image, dest=0, tag=rank)

def parse_command_line_args():
    # Implementation details not provided in sources [G]
    return "bmin:0,0,0 bmax:1,1,1 mmu:1.0 file:data.vol"

def get_slave_count():
    # Implementation details not provided in sources [G]
    return 4

def get_image_dimensions():
    # Implementation details not provided in sources [G]
    return 512, 512

def load_transfer_function(buf):
    # Implementation details not provided in sources [G]
    pass

def parse_vifo(msg):
    # Implementation details not provided in sources [G]
    return {"dims": (512,512,512), "bmin": (0,0,0), "bmax": (1,1,1)}

def receive_octree_subtree():
    # Implementation details not provided in sources [G]
    return None

def receive_data_block():
    # Implementation details not provided in sources [G]
    return None

def render_segments(data, octree, tf, vifo):
    # Implementation details not provided in sources [G]
    return np.zeros(1000, dtype=np.uint8)

def assemble_image(ibuf, height, width):
    # Implementation details not provided in sources [G]
    return np.zeros((height, width, 3), dtype=np.uint8)

def encode_to_jpeg(image):
    # Implementation details not provided in sources [G]
    pass

def transmit_over_tcp(image):
    # Implementation details not provided in sources [G]
    pass
```

**Parameters:**
*   `TF1D_BUF_SIZE`: 1024 * 10 bytes [S2, unofficial].
*   `MSG_BUF_SIZE`: 1024 bytes [S2, unofficial].
*   `ibuf_size`: Calculated as `ceil(height / slave_size) * width * 3` [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Data Replication:** Preprocessing must be conducted at input processors rather than rendering processors to avoid data replication, as input processors have access to all needed data [S1].
*   **Static Mesh:** The method assumes the mesh structure never changes throughout the simulation, allowing the octree to be generated once [S1].

## Subtopics

### [MPI Collective Noncontiguous Read for Octree Data](10836-mpi-collective-noncontiguous-read-for-octree-data.md)
A file reading strategy using MPI-IO collective operations to read noncontiguous hexahedral cell data reconstructed from node data based on octree blocks.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Avoids data replication by preprocessing at input processors; efficient traversal of empty space via octree; one-time distribution of octree subtree reduces communication overhead for time-varying data [S1].
*   **Limitations:** Requires a parallel file system; assumes static mesh structure for octree generation [S1].
*   **When to use:** Large time-varying volume data where the spatial structure is static [S1].
*   **When not to use:** Dynamic mesh structures that change every time step [S1].

## Background
*   **Origin:** The method is described in "I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data" by Hongfeng Yu, Kwan-Liu Ma & Joel Welling [S1].
*   **History:** The architecture is essentially a parallel pipeline that becomes most efficient as soon as all pipeline stages are filled [S1].
*   **Usage:** Used for parallel visualization of large time-varying volume data [S1].

## Sources
[S1] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
