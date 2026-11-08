# Streaming LPM

Also called: Slice-based streaming, Sub-volume streaming

## Overview
Streaming LPM is a memory-management technique for Light Propagation Maps (LPM) that overcomes GPU memory limitations by processing the volume in small, streamed sub-volumes rather than the entire grid at once [S1]. It sets itself apart from standard LPM by decoupling the memory footprint from the total volume size, allowing high-resolution volumes (e.g., $256^3$) to be rendered on hardware with limited VRAM, whereas standard LPM requires storing all radiance buffers for the entire volume simultaneously [S1].

## How it works
**Inputs and Outputs:**
- **Input:** The full volume domain $V$, defined by maximum coordinates $(X_{max}, Y_{max}, Z_{max})$, the sub-volume size parameter $N_{sub}$, and the LPM propagation parameters (directions, bounces).
- **Output:** The final scattered radiance field for the volume.

**Data Structures:**
The volume is partitioned into sub-volumes $B_{ijk}$ and slices $S_l$.
- **Sub-volume $B_{ijk}$:** Defined as the set of voxels $C_{xyz}$ where:
  $$ B_{ijk} = \{ C_{xyz} \mid x \in [i \cdot N_{sub}, (i+1) \cdot N_{sub}], y \in [j \cdot N_{sub}, (j+1) \cdot N_{sub}], z \in [k \cdot N_{sub}, (k+1) \cdot N_{sub}] \} $$
  where $C_{xyz}$ is a voxel at spatial position $(x,y,z)$ [S1].
- **Slice $S_l$:** A group of sub-volumes aligned with the LPM face orientation. For an LPM face perpendicular to the Z-axis, a slice is defined as:
  $$ S_l = \{ B_{ijk} \mid i \in [0, I_{max}], j \in [0, J_{max}], k = l \} $$
  where $l$ is the slice index along the Z-axis [S1].

**Parameters:**
- $N_{sub}$: The linear dimension of a sub-volume. It must be chosen such that the memory required for one slice fits within GPU limits [S1].
- CUDA Streams: Two streams are utilized:
  1. **Transfer Stream:** Manages CPU-GPU data transfer (downloading previous slice results, uploading next slice inputs).
  2. **Computation Stream:** Executes the LPM propagation algorithm for the current slice [S1].
- Pinned Memory: Host memory allocated as pinned memory to accelerate CPU-GPU transfers [S1].

**Algorithm Steps:**
1. **Partitioning:** Split the volume $V$ into sub-volumes $B_{ijk}$ using $N_{sub}$. Group these sub-volumes into slices $S_l$ based on the LPM face orientation (e.g., Z-slices for Z-perpendicular faces) [S1].
2. **Stream Initialization:** Initialize two CUDA streams: one for computation and one for data transfer. Allocate pinned memory buffers on the CPU [S1].
3. **Iteration Loop:** Iterate through each slice $S_l$ from $l=0$ to $Z_{max}/N_{sub} - 1$. The total number of steps is $Z_{max} / N_{sub}$ [S1].
4. **Data Transfer (Overlap):**
   - The **Transfer Stream** downloads the radiance data ($I, U$) computed for the previous slice $S_{l-1}$ from GPU to CPU pinned memory.
   - Simultaneously, the **Transfer Stream** uploads the necessary input data (boundary conditions or incoming radiance) for the next slice $S_{l+1}$ from CPU pinned memory to GPU [S1].
   - *Note:* For the first slice, initial data is uploaded. For the last slice, no subsequent upload is needed [S1].
5. **Computation:**
   - The **Computation Stream** executes the LPM propagation algorithm (Algorithm 2 from [S1]) on the current slice $S_l$.
   - This involves iterating over all voxels in the slice and all propagation directions to update radiance buffers $I$ and $U$ using the transmittance $T$ [S1].
   - Propagation and collection for slice $S_l$ must complete before $S_{l+1}$ is processed [S1].
6. **Result Storage:** The radiance propagated through slice $S_l$ is stored at its outgoing face. This data serves as the input boundary condition for the next slice $S_{l+1}$ [S1].
7. **Sub-slicing (Fallback):** If a single slice $S_l$ still exceeds GPU memory capacity, it is further split into sub-slices and streamed individually [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def streaming_lpm(volume, N_sub, lpm_params):
    # 1. Partitioning
    # Calculate max indices for sub-volumes
    I_max = ceil(volume.X_max / N_sub)
    J_max = ceil(volume.Y_max / N_sub)
    K_max = ceil(volume.Z_max / N_sub)
    
    # Define slices based on Z-axis orientation
    slices = []
    for k in range(K_max):
        slice_subvolumes = []
        for i in range(I_max):
            for j in range(J_max):
                slice_subvolumes.append((i, j, k))
        slices.append(slice_subvolumes)
        
    # 2. Initialize CUDA Streams and Pinned Memory
    transfer_stream = cuda_stream()
    compute_stream = cuda_stream()
    pinned_memory = allocate_pinned_memory()
    
    # 3. Iterate through slices
    for l in range(len(slices)):
        current_slice = slices[l]
        
        # 4. Data Transfer (Overlap with previous computation)
        # Download previous slice result to pinned memory
        if l > 0:
            transfer_stream.download_gpu_to_pinned(pinned_memory, l-1)
            
        # Upload next slice input from pinned memory
        if l < len(slices) - 1:
            transfer_stream.upload_pinned_to_gpu(pinned_memory, l+1)
            
        # 5. Computation
        # Launch kernel for current slice on compute_stream
        # The kernel processes all voxels in current_slice
        # It uses radiance data from the outgoing face of the previous slice
        # or initial conditions for l=0
        launch_lpm_kernel(compute_stream, current_slice, lpm_params)
        
    # 6. Finalize
    # Ensure all streams complete
    compute_stream.synchronize()
    transfer_stream.synchronize()
```

**Edge Cases and Pitfalls:**
- **Memory Limitation:** A $256^3$ volume with 25 propagation directions per LPM face requires ~3.8 GB memory, exceeding typical GPU memory (e.g., 1.5–2 GB on GTX 580/560M) [S1]. Streaming is mandatory in this case [S1].
- **Precision:** Not given in the sources.
- **Boundaries:** The radiance stored at the outgoing face of one slice must be correctly passed to the incoming face of the next slice. Failure to synchronize streams correctly will cause data races or stale data usage [S1].

## Operating characteristics
- **Time Complexity:** The computational work per voxel remains the same as standard LPM, but overhead is introduced by data transfers. The use of pinned memory and concurrent streams (transfer and computation) overlaps these costs [S1].
- **Memory Complexity:** Scales with the size of a single slice (or sub-slice) rather than the entire volume. This allows rendering of volumes much larger than available VRAM [S1].
- **Measured Performance:** For a $64^3$ volume, 25 directions, and 3 bounces, the GPU implementation took 417 ms, compared to 47 seconds on a CPU [S1]. Speedup increases with volume size due to better GPU resource utilization [S1].
- **Strengths:** Enables high-resolution volume rendering on consumer GPUs; efficient use of GPU parallelism via concurrent streams [S1].
- **Limitations:** Requires careful management of data dependencies between slices; increased implementation complexity compared to standard LPM [S1].

## Background
Streaming LPM was introduced in the paper "Light Propagation Maps on Parallel Graphics Architectures" (2012) by the authors of [S1]. It was developed to address the severe memory limitations of GPUs at the time, which restricted the size of volumes that could be processed using standard LPM algorithms. The technique leverages CUDA's concurrent execution and data transfer capabilities to optimize performance [S1].

## Sources
[S1] Light Propagation Maps on Parallel Graphics Architectures, https://doi.org/10.2312/egpgv/egpgv12/081-088, fetched 2026-10-26
