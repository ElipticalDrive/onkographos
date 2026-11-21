# ShardedVolume

Also called: Volume Shards

## Overview
ShardedVolume is a data structure that represents a 3D volume partitioned into multiple independent shards (sub-arrays) to facilitate parallel processing and memory management across devices or memory spaces [S1, unofficial]. It solves the problem of managing large volumetric datasets that exceed single-device memory or require distributed computation by treating each shard as an autonomous unit for arithmetic operations and data access [S1, unofficial]. It differs from [MPI-based Volume Partitioning](1093-mpi-based-volume-partitioning.md) by focusing on in-process or device-local shard management rather than explicit message-passing interface protocols, and from [Object-space decomposition with local clustering](4156-object-space-decomposition-with-local-clustering.md) by using a fixed spatial grid partition rather than adaptive clustering based on object density.

## How it works
**Inputs and Outputs:**
The structure takes a 3D volume domain and partitions it into $N$ shards. It outputs a unified interface that supports arithmetic operations (AXPY, dot product) and slice extraction (Axial, Coronal, Sagittal) across the global volume, abstracting the underlying shard storage.

**Data Structures:**
The `ShardedVolume` object contains two primary fields:
1.  `shards`: A tuple of arrays, where each element corresponds to one shard's data. The data type and shape depend on the shard's local dimensions.
2.  `volume_shards`: A tuple of metadata objects. Each metadata object defines the global position of its corresponding shard and contains the fields:
    *   `z_start`: The starting index of the shard along the Z-axis in the global volume.
    *   `z_stop`: The stopping index of the shard along the Z-axis in the global volume.

**Numbered Steps:**

1.  **Initialization**: The volume is divided into shards. Each shard is assigned a global Z-range $[z_{start}, z_{stop})$. The `shards` tuple stores the local data, and `volume_shards` stores the metadata.

2.  **Axial Slice Extraction**: To extract an axial slice at global index $i$:
    *   Iterate through `volume_shards` to find the shard $j$ such that $z_{start}^{(j)} \le i < z_{stop}^{(j)}$.
    *   If no shard contains the index, raise a `ValueError`.
    *   Calculate the local index: $l = i - z_{start}^{(j)}$.
    *   Return `shards[j][l]`.

3.  **Coronal Slice Extraction**: To extract a coronal slice at global index $i$:
    *   Clamp $i$ to $[0, \text{shape}[1]-1]$, where $\text{shape}[1]$ is the global Y-dimension.
    *   Initialize an empty result array.
    *   For each shard $j$ in `shards`:
        *   Extract the slice `shards[j][:, i, :]`.
        *   Concatenate this slice to the result array along axis 0.
    *   Return the concatenated result.

4.  **Sagittal Slice Extraction**: To extract a sagittal slice at global index $i$:
    *   Clamp $i$ to $[0, \text{shape}[2]-1]$, where $\text{shape}[2]$ is the global X-dimension.
    *   Initialize an empty result array.
    *   For each shard $j$ in `shards`:
        *   Extract the slice `shards[j][:, :, i]`.
        *   Concatenate this slice to the result array along axis 0.
    *   Return the concatenated result.

5.  **AXPY Operation**: To perform $dst = dst + \alpha \cdot src$:
    *   Verify compatibility: `left.global_shape == right.global_shape`, `len(left.shards) == len(right.shards)`, and identical `volume_shards` partitions. Raise `ValueError` if mismatched.
    *   For each shard pair $(self.shards[j], x.shards[j])$:
        *   Update the destination shard in place: `dst[...] += float(alpha) * src`.

6.  **Dot Product**: To compute $\sum (self \cdot x)$:
    *   Initialize `total = 0.0`.
    *   For each shard pair $(self.shards[j], x.shards[j])$:
        *   Compute the local dot product: $p = \sum (self.shards[j].astype(float64) \cdot x.shards[j].astype(float64))$.
        *   Move scalar $p$ to host memory.
        *   Add $p$ to `total`.
    *   Return `total`.

7.  **Concatenation to NumPy**: To convert the entire volume to a contiguous NumPy array:
    *   Iterate over `self.shards`.
    *   Convert each shard to a NumPy array via `_to_numpy`.
    *   Concatenate all arrays along axis 0 using `np.concatenate`.
    *   Ensure contiguous memory with `np.ascontiguousarray`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class ShardedVolume:
    def __init__(self, shards, volume_shards):
        """
        shards: tuple of arrays (local data for each shard)
        volume_shards: tuple of objects with z_start and z_stop attributes
        """
        self.shards = shards
        self.volume_shards = volume_shards
        # Compute global shape from shards if not provided
        # Note: Global shape calculation depends on how shards are arranged.
        # Assuming Z-axis partitioning for simplicity based on z_start/z_stop.
        # Y and X dimensions are assumed constant across shards for this implementation.
        self.global_shape = (
            volume_shards[-1].z_stop, 
            shards[0].shape[1], 
            shards[0].shape[2]
        )

    def get_axial_slice(self, index):
        """Extract axial slice at global Z index."""
        for j, meta in enumerate(self.volume_shards):
            if meta.z_start <= index < meta.z_stop:
                local_index = index - meta.z_start
                return self.shards[j][local_index]
        raise ValueError(f"No shard contains index {index}")

    def get_coronal_slice(self, index):
        """Extract coronal slice at global Y index."""
        max_y = self.global_shape[1] - 1
        index = max(0, min(index, max_y))
        result = None
        for shard in self.shards:
            slice_data = shard[:, index, :]
            if result is None:
                result = slice_data
            else:
                result = np.concatenate((result, slice_data), axis=0)
        return result

    def get_sagittal_slice(self, index):
        """Extract sagittal slice at global X index."""
        max_x = self.global_shape[2] - 1
        index = max(0, min(index, max_x))
        result = None
        for shard in self.shards:
            slice_data = shard[:, :, index]
            if result is None:
                result = slice_data
            else:
                result = np.concatenate((result, slice_data), axis=0)
        return result

    def axpy(self, alpha, x, dst):
        """dst = dst + alpha * x. Assumes compatible sharding."""
        if not self._check_compatibility(x):
            raise ValueError("Incompatible sharding for axpy")
        for j in range(len(self.shards)):
            self.shards[j][...] += float(alpha) * x.shards[j]

    def dot(self, x):
        """Compute dot product of self and x."""
        if not self._check_compatibility(x):
            raise ValueError("Incompatible sharding for dot")
        total = 0.0
        for j in range(len(self.shards)):
            # Cast to float64 for precision
            left = self.shards[j].astype(np.float64, copy=False)
            right = x.shards[j].astype(np.float64, copy=False)
            local_dot = np.sum(left * right)
            total += local_dot
        return total

    def to_numpy(self):
        """Convert to contiguous NumPy array."""
        arrays = [self._to_numpy(shard) for shard in self.shards]
        concatenated = np.concatenate(arrays, axis=0)
        return np.ascontiguousarray(concatenated)

    def _to_numpy(self, shard):
        """Helper to convert shard to numpy array."""
        # Implementation depends on underlying array type (e.g., cupy, torch)
        # Assuming numpy-like interface for this example
        if hasattr(shard, 'get'):
            return shard.get()
        return np.asarray(shard)

    def _check_compatibility(self, other):
        """Check if two ShardedVolumes are compatible for arithmetic."""
        if self.global_shape != other.global_shape:
            return False
        if len(self.shards) != len(other.shards):
            return False
        # Check identical volume_shards partitions
        for s_meta, o_meta in zip(self.volume_shards, other.volume_shards):
            if s_meta.z_start != o_meta.z_start or s_meta.z_stop != o_meta.z_stop:
                return False
        return True
```

**Parameters:**
*   `alpha`: Scalar multiplier in AXPY operation. Typical value: 1.0.
*   `index`: Slice index. Clamped to valid range for Coronal/Sagittal.

**Edge Cases and Pitfalls:**
*   **Compatibility Mismatch**: `axpy` and `dot` require `left.global_shape == right.global_shape`, `len(left.shards) == len(right.shards)`, and identical `volume_shards` partitions. A mismatch raises a `ValueError` [S1, unofficial].
*   **Index Out of Bounds**: Axial slice extraction raises `ValueError` if the index is not contained in any shard [S1, unofficial]. Coronal and Sagittal slices clamp the index to the valid range [S1, unofficial].
*   **Memory Contiguity**: Concatenation to NumPy requires ensuring contiguous memory with `np.ascontiguousarray` [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory**: Linear in the size of the volume data, plus overhead for metadata. The sharding allows for out-of-core processing if shards are loaded on demand, but the structure itself holds all shards in memory if not explicitly managed by an external loader.
*   **Axial Slice**: $O(1)$ shard lookup, $O(W \cdot H)$ data access.
*   **Coronal/Sagittal Slice**: $O(N \cdot H \cdot W)$ or $O(N \cdot W \cdot H)$ where $N$ is the number of shards, due to concatenation across shards.
*   **AXPY**: $O(V)$ where $V$ is the total volume size, distributed across shards.
*   **Dot Product**: $O(V)$ with communication overhead for summing scalar results from each shard.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Scales with the number of shards and devices. Parallel arithmetic operations can be performed on each shard independently.

**Strengths and Limitations:**
*   **Strengths**: Enables parallel processing and memory management across devices. Provides a unified interface for arithmetic and slicing.
*   **Limitations**: Coronal and Sagittal slices require concatenation across shards, which may be inefficient. Strict compatibility requirements for arithmetic operations.

**When to Use:**
Use when dealing with large volumes that need to be processed in parallel or distributed across multiple devices, and when axial slices are the primary access pattern.

**When Not to Use:**
Avoid if frequent coronal or sagittal slices are required and performance is critical, due to the concatenation overhead. Avoid if the volume is small enough to fit in a single device's memory.

## Background
ShardedVolume is part of the TomoAnchor-V3 project, an open-source tool for tomographic reconstruction [S1, unofficial]. It was developed to handle large volumetric datasets in medical imaging and computed tomography (CT) reconstruction, where memory constraints and computational demands require efficient data partitioning. The implementation is found in the `tomoanchor_gpu_pdhg.py` and `tomoanchor_gpu_v3.py` modules, which utilize sharded volumes for GPU-accelerated reconstruction algorithms [S1, unofficial].

## Sources
[S1] alidarbandi/TomoAnchor-V3, https://github.com/alidarbandi/TomoAnchor-V3, fetched 2026-10-27 (unofficial)
