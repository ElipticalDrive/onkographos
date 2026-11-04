# MPI Master Node Parallel Rendering Workflow

Also called: MPI Image Assembly, Master-Slave Volume Rendering

## Overview
This method implements a master-slave parallel volume rendering workflow where a single master process coordinates multiple slave processes to render a 3D volume dataset. The master distributes volume metadata and transfer functions, while slaves perform independent ray casting or slice-based compositing on assigned spatial regions. The master assembles the resulting partial image buffers into a final image, typically encoding it to JPEG for transmission or storage [S1, unofficial]. It differs from pure MPI-only approaches by introducing a TCP-based communication layer for image data transfer, allowing slaves to operate as independent server processes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Command-line arguments specifying image dimensions (`width`, `height`), number of slave processes (`slave_size`), thread counts, volume file path, and rendering parameters.
- **Outputs:** A final assembled image file (e.g., JPEG) and timing statistics.

**Data Structures:**
- `vifo`: A structure containing volume metadata:
  - `dims` (Vector3i): Volume dimensions.
  - `blk_size` (Vector3i): Brick/block sizes.
  - `bmin`, `bmax` (Vector3f): Bounding box coordinates.
  - `mmu_ratio` (float): Memory management unit ratio or scaling factor.
  - `volume_file` (string): Path to the volume data file.
- `tf1d_buf`: A buffer of size `TF1D_BUF_SIZE` (10240 bytes) storing the 1D transfer function data [S1, unofficial].
- `ibuf`: A 2D array `char ibuf[slave_size][ibuf_size]` storing partial image buffers from each slave.
  - `ibuf_size` is calculated as $\lceil \text{height} / \text{slave\_size} \rceil \times \text{width} \times 3$ bytes [S1, unofficial].
- `slave_request` and `slave_status`: Arrays of size `slave_size` for MPI asynchronous receive requests and status.
- `msg_buf`: A buffer of size `MSG_BUF_SIZE` (1024 bytes) for broadcasting control messages [S1, unofficial].

**Steps:**
1. **Initialization:**
   - Parse command-line arguments to extract `width`, `height`, `slave_size`, `thread_nums`, and `vifo_path`.
   - Load volume metadata into a `vifo` structure.
   - Construct a message string containing dimensions, brick sizes, bounding box, `mmu_ratio`, and file path.
   - Spawn `slave_size` slave processes using `MPI_Comm_spawn` with the constructed message as arguments [S1, unofficial].
   - Send the transfer function data to all slaves using `send_tf1d` (broadcasting `tf1d_buf` over `slave_comm`) [S1, unofficial].

2. **Communication Setup:**
   - Set up a TCP acceptor on `PORT` (9523) for receiving the final assembled image if required by the specific implementation variant, though the primary assembly happens in memory before disk write [S1, unofficial].
   - Calculate `lines_per_slave` as $\lceil \text{height} / \text{slave\_size} \rceil$.
   - Allocate `ibuf` and request/status arrays.

3. **Rendering Loop (for `image_nums` iterations):**
   - Accept TCP connections from slaves (if using TCP for image transfer) or rely on MPI for image data [S1, unofficial]. *Note: The source code shows MPI_Irecv for image data, but also mentions TCP acceptor and JPEG encoding over socket. The primary workflow described in the notes uses MPI for image assembly.*
   - Broadcast a control message (`msg_buf`) to all slaves via `MPI_Send` (or `MPI_Bcast` if implemented as such) [S1, unofficial].
   - Record start time `t1`.
   - Issue non-blocking receives for each slave: `MPI_Irecv` into `ibuf[slave_id]` for `ibuf_size` bytes, tag `slave_id`, from rank `slave_id` on `slave_comm`.
   - Wait for all receives to complete using `MPI_Waitall`.
   - Record end time `t2`.
   - Calculate and print rendering time.

4. **Image Assembly:**
   - Iterate through each row `r` from 0 to `height - 1`.
   - Determine the slave responsible for this row: `slave_no = r % slave_size`.
   - Determine the row index within that slave's buffer: `slave_row = r / slave_size`.
   - Copy `width * 3` bytes from `ibuf[slave_no] + slave_row * width * 3` to the destination buffer `dst_base`.
   - Advance `dst_base` by `width * 3`.
   - *Pitfall:* This round-robin distribution assumes `height` is divisible by `slave_size` for perfect uniformity, but the integer division handles uneven distribution by assigning extra rows to earlier slaves [S1, unofficial].

5. **Output:**
   - Encode the assembled image buffer (`ibuf_dst`) to JPEG format.
   - Write the JPEG size and buffer to a socket (if TCP transmission is used) or save to disk [S1, unofficial].
   - Close connections and finalize MPI.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import time
import mpi4py.MPI as MPI
import numpy as np

# Constants from source
TF1D_BUF_SIZE = 10240
MSG_BUF_SIZE = 1024
PORT = 9523

def send_tf1d(tf1d_buf, comm):
    """Broadcast transfer function data to all slaves."""
    comm.Bcast(tf1d_buf, root=0)

def main():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    
    if rank == 0:
        # 1. Initialization
        # Parse args: width, height, slave_size, thread_nums, vifo_path
        width = 512
        height = 512
        slave_size = 4
        vifo_path = "volume.vifo"
        
        # Load volume metadata (simulated)
        vifo = {
            'dims': (512, 512, 512),
            'blk_size': (64, 64, 64),
            'bmin': (0.0, 0.0, 0.0),
            'bmax': (1.0, 1.0, 1.0),
            'mmu_ratio': 1.0,
            'volume_file': vifo_path
        }
        
        # Construct message string
        msg = f"{vifo['dims']} {vifo['blk_size']} {vifo['bmin']} {vifo['bmax']} {vifo['mmu_ratio']} {vifo['volume_file']}"
        
        # Spawn slaves
        # Note: In real MPI, this requires the slave executable to be available
        # slave_comm = comm.Spawn("./VolumeRendererServer_slave", args=[msg], count=slave_size)
        # For this example, we assume slave_comm is established or use a pre-existing group
        # Since MPI_Comm_spawn is complex to simulate without actual processes, 
        # we assume slave_comm is a valid communicator with slave_size processes.
        slave_comm = MPI.COMM_SELF.Dup() # Placeholder for actual slave communicator
        # In reality, slave_comm would be created via MPI_Comm_spawn
        
        # Send Transfer Function
        tf1d_buf = np.zeros(TF1D_BUF_SIZE, dtype=np.uint8)
        send_tf1d(tf1d_buf, slave_comm)
        
        # Allocate buffers
        ibuf_size = math.ceil(height / slave_size) * width * 3
        ibuf = np.zeros((slave_size, ibuf_size), dtype=np.uint8)
        slave_request = [slave_comm.Irecv(ibuf[i], source=i, tag=i) for i in range(slave_size)]
        
        image_nums = 5
        t_sum = 0.0
        
        # 3. Rendering Loop
        for j in range(image_nums):
            # Broadcast control message
            msg_buf = np.zeros(MSG_BUF_SIZE, dtype=np.uint8)
            msg_buf[:len(msg)] = msg.encode('ascii')
            slave_comm.Bcast(msg_buf, root=0)
            
            t1 = MPI.Wtime()
            
            # Issue non-blocking receives
            for i in range(slave_size):
                slave_request[i] = slave_comm.Irecv(ibuf[i], source=i, tag=i)
            
            # Wait for all
            MPI.Waitall(slave_request)
            
            t2 = MPI.Wtime()
            if j >= 2:
                render_time = t2 - t1
                t_sum += render_time
                print(f"Render time for image {j}: {render_time}")
        
        print(f"Average Rendering time cost: {t_sum / (image_nums - 2)}")
        
        # 4. Image Assembly
        dst_base = np.zeros((height, width, 3), dtype=np.uint8)
        for r in range(height):
            slave_no = r % slave_size
            slave_row = r // slave_size
            src_offset = slave_row * width * 3
            dst_offset = r * width * 3
            dst_base[r] = ibuf[slave_no, src_offset:src_offset + width * 3].reshape((width, 3))
            
        # 5. Output
        # Encode to JPEG and write to disk or socket
        # write_image_disk(dst_base, width, height, "test.jpg")
        
        # slave_comm.Free()
        MPI.Finalize()

if __name__ == "__main__":
    main()
```

**Parameters:**
- `slave_size`: Number of slave processes. Typical values depend on available cores/nodes.
- `width`, `height`: Image resolution.
- `image_nums`: Number of frames to render for timing averaging.
- `PORT`: TCP port (9523) for potential image transmission [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Uneven Height Distribution:** If `height` is not divisible by `slave_size`, the last rows are distributed unevenly. The formula `slave_row = r // slave_size` correctly maps rows to slave buffers, but the buffer size `ibuf_size` must be large enough to hold the maximum number of rows any single slave might contribute, which is $\lceil \text{height} / \text{slave\_size} \rceil$ [S1, unofficial].
- **Transfer Function Size:** The transfer function buffer size is fixed at 10240 bytes. If the transfer function exceeds this, it will be truncated or cause overflow [S1, unofficial].
- **MPI Spawn:** `MPI_Comm_spawn` requires the slave executable to be in the PATH or specified with a full path. Error handling for spawn failures is critical [S1, unofficial].

## Subtopics

### MPI Master-Slave Octree Volume Rendering
A variant where slaves traverse an Octree (OTree) structure to find empty/non-empty blocks, computing ray segments only for non-empty blocks to optimize rendering. The master sends volume metadata (bmin, bmax, mmu_ratio, file path) and transfer function data. Slaves perform ray casting on these optimized segments. This differs from standard voxel-based ray casting by leveraging hierarchical data structures for early ray termination and empty space skipping [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by ray casting on slaves and MPI communication. Image assembly is $O(\text{height} \times \text{width})$.
- **Memory Complexity:** $O(\text{slave\_size} \times \text{height} \times \text{width} \times 3)$ for image buffers on the master. Slaves store volume data and transfer functions.
- **Performance:** Timing is measured over multiple frames, averaging out startup costs. The use of non-blocking MPI receives allows for potential overlap, though the example waits for all before assembly.
- **Strengths:** Simple master-slave model, leverages MPI for process management, TCP for flexible image transfer.
- **Limitations:** Master becomes a bottleneck for image assembly and control. Round-robin row distribution may lead to load imbalance if ray casting cost varies significantly per row (e.g., due to opacity variations). Fixed buffer sizes for TF and messages limit flexibility.
- **When to Use:** Suitable for static volumes where load balancing is not critical or where the volume is uniform. Not ideal for highly heterogeneous volumes without additional load balancing strategies.

## Background
This workflow originates from distributed volume rendering systems using MPI for process coordination and TCP for data transfer. It is commonly found in academic and research implementations of parallel volume rendering, particularly those focusing on master-slave architectures for simplicity and ease of implementation [S1, unofficial].

## Sources
[S1] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
