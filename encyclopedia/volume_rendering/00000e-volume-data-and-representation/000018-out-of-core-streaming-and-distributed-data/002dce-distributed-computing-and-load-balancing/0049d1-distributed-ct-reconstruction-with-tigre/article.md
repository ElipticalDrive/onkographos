# Distributed CT Reconstruction with TIGRE

Also called: Distributed Ax/Atb, Distributed Full-Data Operator, Distributed TIGRE, DistributedFullDataTigrePool, TomoAnchor-V3

## Overview
Distributed CT Reconstruction with TIGRE is a multiprocessing framework that parallelizes the forward projection ($Ax$) and backprojection ($A^Tb$) operators of the TIGRE library across multiple GPUs using shared memory and process pools [S1, unofficial]. It solves the computational bottleneck of large-scale tomographic reconstruction by partitioning projection angles across worker processes, allowing independent GPU execution while maintaining a consistent global data view through shared memory buffers [S1, unofficial]. Unlike spatial decomposition methods that partition the volume, this method partitions the measurement space (angles), making it suitable for iterative reconstruction algorithms that require repeated forward and backprojection operations [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs**:
    *   `volume`: 3D numpy array of shape `(nz, ny, nx)` and dtype `float32` [S1, unofficial].
    *   `projections`: 3D numpy array of shape `(num_angles, det_rows, det_cols)` and dtype `float32` [S1, unofficial].
    *   `angles`: 1D numpy array of projection angles in radians [S1, unofficial].
    *   `geometry_params`: Parameters defining the CT geometry (source-detector distance, detector pixels, voxel size, etc.) [S1, unofficial].
    *   `gpu_ids`: List of GPU indices to use [S1, unofficial].
*   **Outputs**:
    *   `projections`: Result of forward projection ($Ax$) [S1, unofficial].
    *   `volume`: Result of backprojection ($A^Tb$) [S1, unofficial].
    *   `residual`: Scalar value for residual squared norm [S1, unofficial].

### Data Structures

**`_WorkerShard`**
A dataclass defining the assignment of angles to a specific worker/GPU [S1, unofficial].
*   `worker_id` (int): Unique identifier for the worker process.
*   `gpu_id` (int): CUDA device ID assigned to this worker.
*   `start` (int): Start index (inclusive) in the `angles` array.
*   `stop` (int): Stop index (exclusive) in the `angles` array.

**`_SharedArraySpec`**
Specification for shared memory buffers [S1, unofficial].
*   `name` (str): Unique name for the shared memory segment.
*   `shape` (tuple): Shape of the numpy array.
*   `dtype` (np.dtype): Data type (e.g., `np.float32`).

**Shared Memory Buffers**
*   `_volume_shared`: Single shared array of shape `volume_shape` (`nz, ny, nx`), dtype `float32`. Used to store the global volume accessible by all workers [S1, unofficial].
*   `_projection_buffers`: Dictionary mapping `worker_id` to a shared array. Shape: `(shard.count, det_rows, det_cols)`. Each worker writes its partial projection results here [S1, unofficial].
*   `_partial_volume_buffers`: Dictionary mapping `worker_id` to a shared array. Shape: `volume_shape` (`nz, ny, nx`). Each worker accumulates its partial backprojection result here [S1, unofficial].

### Numbered Steps

1.  **Initialization**:
    *   Validate inputs: Ensure `len(gpu_ids) > 1` and `len(angles) > 0` [S1, unofficial].
    *   Partition `angles` into shards using `_partition_angle_shards`. Each shard is assigned to a unique GPU from `gpu_ids` [S1, unofficial].
    *   Create shared memory buffers (`_volume_shared`, `_projection_buffers`, `_partial_volume_buffers`) using `multiprocessing.shared_memory.SharedMemory` with `create=True` [S1, unofficial].
    *   Spawn worker processes using `mp.get_context("spawn")` to ensure clean state isolation [S1, unofficial].
    *   For each shard, create a worker process with arguments: shard metadata, geometry params, subset of angles, shared array specs, and command/result queues [S1, unofficial].
    *   Start all worker processes [S1, unofficial].

2.  **Forward Projection ($Ax$)**:
    *   Copy input `volume` to `_volume_shared` using `np.copyto` [S1, unofficial].
    *   Dispatch 'forward' command to all workers via their command queues [S1, unofficial].
    *   Wait for all workers to respond via result queues with timeout `_QUEUE_POLL_SECONDS` [S1, unofficial].
    *   Gather results: For each shard, copy `self._projection_buffers[shard.worker_id].array()` to `output[shard.start : shard.stop]` [S1, unofficial].

3.  **Backprojection ($A^Tb$)**:
    *   Validate input `projections` shape matches `self.projection_shape` [S1, unofficial].
    *   For each shard, copy `projection_array[shard.start : shard.stop]` to `self._projection_buffers[shard.worker_id].array()` [S1, unofficial].
    *   Dispatch 'backproject' command to all workers [S1, unofficial].
    *   Wait for all workers to respond [S1, unofficial].
    *   Reduce results: Initialize `volume = zeros(volume_shape, dtype=float32)`. For each shard, add `self._partial_volume_buffers[shard.worker_id].array()` to `volume` [S1, unofficial].

4.  **Normal Operator ($A^T A$)**:
    *   Dispatch 'normal' command to workers [S1, unofficial].
    *   Workers perform $Ax$ then $A^T(Ax)$ locally [S1, unofficial].
    *   Reduce results: Initialize `result = zeros(volume_shape, dtype=float32)`. For each shard, add `self._partial_volume_buffers[shard.worker_id].array()` to `result` [S1, unofficial].

5.  **Residual Squared**:
    *   Copy `projections` to `_projection_buffers` as in Backprojection [S1, unofficial].
    *   Dispatch 'residual_squared' command [S1, unofficial].
    *   Workers compute local residual squared norm [S1, unofficial].
    *   Reduce results: Sum `response.get('value', 0.0)` from all workers to get `total_value` [S1, unofficial].

### Formulas

*   **Forward Projection**: $y = Ax$ [S1, unofficial].
*   **Backprojection**: $x = A^T b$ [S1, unofficial].
*   **Normal Operator**: $z = A^T A x$ [S1, unofficial].
*   **Residual Squared**: $R = \sum_{i} ( (Ax)_i - b_i )^2$ [S1, unofficial].

### Parameters

*   `_QUEUE_POLL_SECONDS`: Timeout in seconds for `result_queue.get()` in the master wait loop [S1, unofficial].
*   `_PROCESS_JOIN_TIMEOUT_SECONDS`: Timeout in seconds for `process.join()` during shutdown [S1, unofficial].
*   `backprojection_type`: String parameter passed to `tigre.Atb`, set to `'matched'` for backproject and normal operations [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import multiprocessing as mp
from multiprocessing import shared_memory
import tigre
from tigre.utilities import gpu

class DistributedTigrePool:
    def __init__(self, gpu_ids, angles, geometry_params, volume_shape, det_pixels):
        self.gpu_ids = gpu_ids
        self.angles = angles
        self.geometry_params = geometry_params
        self.volume_shape = volume_shape
        self.det_pixels = det_pixels
        self.projection_shape = (len(angles), det_pixels[0], det_pixels[1])
        
        # Partition angles
        self.shards = self._partition_angle_shards(angles, gpu_ids)
        
        # Create shared memory buffers
        self._volume_shared = self._create_shared_array("vol", volume_shape, np.float32)
        self._projection_buffers = {}
        self._partial_volume_buffers = {}
        
        for shard in self.shards:
            proj_shape = (shard.stop - shard.start, det_pixels[0], det_pixels[1])
            self._projection_buffers[shard.worker_id] = self._create_shared_array(
                f"proj_{shard.worker_id}", proj_shape, np.float32
            )
            self._partial_volume_buffers[shard.worker_id] = self._create_shared_array(
                f"pvol_{shard.worker_id}", volume_shape, np.float32
            )
            
        # Spawn workers
        self.workers = []
        self.command_queues = {}
        self.result_queues = {}
        
        for shard in self.shards:
            cmd_q, res_q = mp.Queue(), mp.Queue()
            self.command_queues[shard.worker_id] = cmd_q
            self.result_queues[shard.worker_id] = res_q
            
            p = mp.Process(
                target=_distributed_tigre_worker,
                args=(shard, geometry_params, self._volume_shared, 
                      self._projection_buffers, self._partial_volume_buffers,
                      cmd_q, res_q)
            )
            p.start()
            self.workers.append(p)

    def _create_shared_array(self, name, shape, dtype):
        shm = shared_memory.SharedMemory(create=True, size=np.prod(shape) * dtype.itemsize)
        return np.ndarray(shape, dtype=dtype, buffer=shm.buf)

    def _partition_angle_shards(self, angles, gpu_ids):
        shards = []
        num_angles = len(angles)
        num_gpus = len(gpu_ids)
        angles_per_gpu = num_angles // num_gpus
        remainder = num_angles % num_gpus
        
        start_idx = 0
        for i, gpu_id in enumerate(gpu_ids):
            count = angles_per_gpu + (1 if i < remainder else 0)
            shards.append(_WorkerShard(
                worker_id=i,
                gpu_id=gpu_id,
                start=start_idx,
                stop=start_idx + count
            ))
            start_idx += count
        return shards

    def forward(self, volume):
        # Copy volume to shared memory
        np.copyto(self._volume_shared, np.ascontiguousarray(volume, dtype=np.float32), casting='no')
        
        # Dispatch commands
        for shard in self.shards:
            self.command_queues[shard.worker_id].put({'op': 'forward'})
            
        # Wait and gather
        responses = self._wait_for_responses()
        output = np.zeros(self.projection_shape, dtype=np.float32)
        for shard in self.shards:
            np.copyto(output[shard.start:shard.stop], 
                      self._projection_buffers[shard.worker_id].array(), 
                      casting='no')
        return output

    def backproject(self, projections):
        # Copy projections to shared buffers
        for shard in self.shards:
            np.copyto(self._projection_buffers[shard.worker_id].array(),
                      np.ascontiguousarray(projections[shard.start:shard.stop], dtype=np.float32),
                      casting='no')
                      
        # Dispatch commands
        for shard in self.shards:
            self.command_queues[shard.worker_id].put({'op': 'backproject'})
            
        # Wait and reduce
        self._wait_for_responses()
        volume = np.zeros(self.volume_shape, dtype=np.float32)
        for shard in self.shards:
            volume += self._partial_volume_buffers[shard.worker_id].array()
        return volume

    def _wait_for_responses(self):
        responses = {}
        for shard in self.shards:
            try:
                res = self.result_queues[shard.worker_id].get(timeout=_QUEUE_POLL_SECONDS)
                if not isinstance(res, dict) or res.get('status') != 'ok':
                    raise RuntimeError(f"Invalid response from worker {shard.worker_id}")
                responses[shard.worker_id] = res
            except mp.queues.Empty:
                if not self.workers[shard.worker_id].is_alive():
                    raise RuntimeError(f"Worker {shard.worker_id} died unexpectedly")
        return responses

def _distributed_tigre_worker(shard, geo_params, vol_shm, proj_buffers, pvol_buffers, cmd_q, res_q):
    # Open shared memory
    vol_shm_arr = np.ndarray(shard.volume_shape, dtype=np.float32, buffer=vol_shm_shm.buf) # Note: vol_shm is SharedMemory object
    # ... (Open other shared memories similarly) ...
    
    # Setup GPU
    gpuids = gpu.GpuIds()
    gpuids.devices = [int(shard.gpu_id)]
    
    # Setup Geometry
    geo = build_tigre_geometry(geo_params)
    local_angles = shard.angles # Subset of angles
    
    while True:
        command = cmd_q.get()
        operation = str(command.get('op', '')).strip().lower()
        
        if operation == 'close':
            break
            
        elif operation == 'forward':
            # Ax
            result = tigre.Ax(
                np.ascontiguousarray(vol_shm_arr, dtype=np.float32), 
                geo, 
                local_angles, 
                gpuids=gpuids
            )
            np.copyto(proj_buffers[shard.worker_id].array(), 
                      np.asarray(result, dtype=np.float32), 
                      casting='no')
                      
        elif operation == 'backproject':
            # Atb
            result = tigre.Atb(
                np.ascontiguousarray(proj_buffers[shard.worker_id].array(), dtype=np.float32), 
                geo, 
                local_angles, 
                backprojection_type='matched', 
                gpuids=gpuids
            )
            np.copyto(pvol_buffers[shard.worker_id].array(), 
                      np.asarray(result, dtype=np.float32), 
                      casting='no')
                      
        elif operation == 'normal':
            # A^T A
            local_proj = tigre.Ax(
                np.ascontiguousarray(vol_shm_arr, dtype=np.float32), 
                geo, 
                local_angles, 
                gpuids=gpuids
            )
            result = tigre.Atb(
                np.ascontiguousarray(local_proj, dtype=np.float32), 
                geo, 
                local_angles, 
                backprojection_type='matched', 
                gpuids=gpuids
            )
            np.copyto(pvol_buffers[shard.worker_id].array(), 
                      np.asarray(result, dtype=np.float32), 
                      casting='no')
                      
        elif operation == 'residual_squared':
            local_proj = tigre.Ax(
                np.ascontiguousarray(vol_shm_arr, dtype=np.float32), 
                geo, 
                local_angles, 
                gpuids=gpuids
            )
            residual = np.asarray(local_proj, dtype=np.float32) - np.asarray(proj_buffers[shard.worker_id].array(), dtype=np.float32)
            value = float(np.sum(np.asarray(residual, dtype=np.float64) ** 2))
            res_q.put({'status': 'ok', 'value': value})
            continue
            
        res_q.put({'status': 'ok', 'worker_id': shard.worker_id, 'gpu_id': shard.gpu_id})

class _WorkerShard:
    def __init__(self, worker_id, gpu_id, start, stop):
        self.worker_id = worker_id
        self.gpu_id = gpu_id
        self.start = start
        self.stop = stop
        self.angles = None # To be filled by parent
```

### Edge Cases and Pitfalls

*   **Process Context**: Uses `spawn` context, which does not inherit open file descriptors or GPU contexts from the parent. This ensures clean state but increases startup time [S1, unofficial].
*   **Shared Memory Cleanup**: Shared memory segments must be explicitly closed and unlinked to prevent OS resource leaks [S1, unofficial].
*   **Worker Death**: If a worker process exits unexpectedly, the master detects this via `Empty` exception from `result_queue.get()` and checks `process.is_alive()`. A `RuntimeError` is raised with the dead PIDs [S1, unofficial].
*   **Invalid Response**: If the response from a worker is not a dictionary or has `status != 'ok'`, a `RuntimeError` is raised [S1, unofficial].
*   **Shape Mismatch**: Volume and projection arrays must match `self.volume_shape` and `self.projection_shape` respectively. A `ValueError` is raised if they do not [S1, unofficial].
*   **Non-Contiguous Arrays**: Input arrays are converted to contiguous using `np.ascontiguousarray` before passing to TIGRE or shared memory copy to avoid memory access errors [S1, unofficial].
*   **Cancellation**: Cancellation is checked only during the wait loop. `OperationCancelled` is raised only after the current GPU call completes [S1, unofficial].

## Operating characteristics

*   **Time Complexity**: The forward and backprojection operations are performed in parallel across $N$ GPUs. The theoretical speedup is linear with the number of GPUs, assuming perfect load balancing of angles and negligible communication overhead [S1, unofficial].
*   **Memory Complexity**: Memory usage is dominated by the shared volume buffer (one copy) and the projection buffers (one per GPU). The total memory footprint is $O(V + N \cdot P)$, where $V$ is the volume size and $P$ is the size of the projection shard per GPU [S1, unofficial].
*   **Communication Overhead**: Communication is limited to copying the volume to shared memory (once per operation) and copying partial results from GPU to shared memory (once per worker). No inter-worker communication is required during the TIGRE kernel execution [S1, unofficial].
*   **Strengths**:
    *   Simple load balancing by partitioning angles.
    *   Leverages TIGRE's optimized CUDA kernels.
    *   Shared memory avoids network overhead in single-node multi-GPU setups [S1, unofficial].
*   **Limitations**:
    *   Requires all GPUs to be accessible via shared memory (single-node or specific cluster configurations).
    *   Volume must fit in shared memory on all nodes.
    *   Not suitable for distributed systems across multiple machines without additional network layer [S1, unofficial].
*   **When to Use**: Ideal for single-node multi-GPU systems where the volume fits in memory and the bottleneck is the projection/backprojection speed [S1, unofficial].
*   **When Not to Use**: When the volume is too large to fit in shared memory or when distributed across multiple networked nodes [S1, unofficial].

## Background

*   **Origin**: This method is part of the TomoAnchor-V3 framework, developed by Ali Darvishbandani [S1, unofficial].
*   **History**: It was introduced as a distributed computing framework for tomographic forward and backprojection operators using multiple GPUs via shared memory and multiprocessing [S1, unofficial].
*   **Usage**: Used in computed tomography (CT) reconstruction, particularly for iterative algorithms like MBIR (Model-Based Iterative Reconstruction) that require repeated forward and backprojection operations [S1, unofficial].

## Sources

[S1] alidarbandi/TomoAnchor-V3, https://github.com/alidarbandi/TomoAnchor-V3, fetched 2026-10-27 (unofficial)
