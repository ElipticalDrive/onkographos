# ShardedVolume


## Overview
ShardedVolume is a data structure representing a 3D volume partitioned into multiple shards (sub-arrays) to allow parallel processing or memory management across devices, with methods for arithmetic operations and data access [S1, unofficial]. It solves the problem of managing large volumetric datasets that exceed single-device memory or require distributed computation by treating each shard as an autonomous unit for arithmetic operations and data access [S1, unofficial].

## How it works
**Inputs and Outputs:**
The structure takes a 3D volume domain partitioned into $N$ shards and outputs a unified interface that supports arithmetic operations (AXPY, dot product) and slice extraction (Axial, Coronal, Sagittal) across the global volume, abstracting the underlying shard storage [S1, unofficial].

**Data Structures:**
The `ShardedVolume` object contains two primary fields:
1. `shards`: A tuple of arrays, where each element corresponds to one shard's data [S1, unofficial].
2. `volume_shards`: A tuple of metadata objects. Each metadata object defines the global position of its corresponding shard and contains the fields `z_start` and `z_stop` [S1, unofficial].

**Numbered Steps:**
1. **Initialization**: The volume is divided into shards. Each shard is assigned a global Z-range $[z_{start}, z_{stop})$ [S1, unofficial]. The `shards` tuple stores the local data, and `volume_shards` stores the metadata [S1, unofficial].
2. **Axial Slice Extraction**: To extract an axial slice at global index $i$: Iterate through `volume_shards` to find the shard $j$ such that $z_{start}^{(j)} \le i < z_{stop}^{(j)}$. If no shard contains the index, raise a `ValueError`. Calculate the local index $l = i - z_{start}^{(j)}$. Return `shards[j][l]` [S1, unofficial].
3. **Coronal Slice Extraction**: To extract a coronal slice at global index $i$: Clamp $i$ to $[0, \text{shape}[1]-1]$, where $\text{shape}[1]$ is the global Y-dimension. For each shard $j$, extract `shards[j][:, i, :]` and concatenate along axis 0 [S1, unofficial].
4. **Sagittal Slice Extraction**: To extract a sagittal slice at global index $i$: Clamp $i$ to $[0, \text{shape}[2]-1]$, where $\text{shape}[2]$ is the global X-dimension. For each shard $j$, extract `shards[j][:, :, i]` and concatenate along axis 0 [S1, unofficial].
5. **AXPY Operation**: To perform $dst = dst + \alpha \cdot src$: Verify compatibility (`left.global_shape == right.global_shape`, `len(left.shards) == len(right.shards)`, and identical `volume_shards` partitions). Raise `ValueError` if mismatched. For each shard pair, update the destination shard in place: `dst[...] += float(alpha) * src` [S1, unofficial].
6. **Dot Product**: To compute $\sum (self \cdot x)$: Initialize `total = 0.0`. For each shard pair, compute `xp.sum(left.astype(xp.float64, copy=False) * right.astype(xp.float64, copy=False))`, move the scalar to host memory, and add it to `total` [S1, unofficial].
7. **Concatenation to NumPy**: To convert the entire volume to a contiguous NumPy array: Iterate over `self.shards`, convert each to a NumPy array via `_to_numpy`, concatenate along axis 0 using `np.concatenate`, and ensure contiguous memory with `np.ascontiguousarray` [S1, unofficial].

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
        # Global shape derived from Z-partitioning and first shard dimensions [G]
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
            # Cast to float64 for precision [G]
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
        """Helper to convert shard to numpy array [G]."""
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
* `alpha`: Scalar multiplier in AXPY operation. Typical value: 1.0 [G].
* `index`: Slice index. Clamped to valid range for Coronal/Sagittal [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Compatibility Mismatch**: `axpy` and `dot` require `left.global_shape == right.global_shape`, `len(left.shards) == len(right.shards)`, and identical `volume_shards` partitions. A mismatch raises a `ValueError` [S1, unofficial].
* **Index Out of Bounds**: Axial slice extraction raises `ValueError` if the index is not contained in any shard [S1, unofficial]. Coronal slices clamp the index to $[0, \text{shape}[1]-1]$ and Sagittal slices clamp to $[0, \text{shape}[2]-1]$ [S1, unofficial].
* **Memory Contiguity**: Concatenation to NumPy requires ensuring contiguous memory with `np.ascontiguousarray` [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
* **Memory**: Linear in the size of the volume data, plus overhead for metadata [G]. The sharding allows for out-of-core processing if shards are loaded on demand, but the structure itself holds all shards in memory if not explicitly managed by an external loader [G].
* **Axial Slice**: $O(1)$ shard lookup, $O(W \cdot H)$ data access [G].
* **Coronal/Sagittal Slice**: $O(N \cdot H \cdot W)$ or $O(N \cdot W \cdot H)$ where $N$ is the number of shards, due to concatenation across shards [G].
* **AXPY**: $O(V)$ where $V$ is the total volume size, distributed across shards [G].
* **Dot Product**: $O(V)$ with communication overhead for summing scalar results from each shard [G].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Scales with the number of shards and devices. Parallel arithmetic operations can be performed on each shard independently [G].

**Strengths and Limitations:**
* **Strengths**: Enables parallel processing and memory management across devices. Provides a unified interface for arithmetic and slicing [G].
* **Limitations**: Coronal and Sagittal slices require concatenation across shards, which may be inefficient. Strict compatibility requirements for arithmetic operations [G].

**When to Use:**
Use when dealing with large volumes that need to be processed in parallel or distributed across multiple devices, and when axial slices are the primary access pattern [G].

**When Not to Use:**
Avoid if frequent coronal or sagittal slices are required and performance is critical, due to the concatenation overhead. Avoid if the volume is small enough to fit in a single device's memory [G].

## Background
ShardedVolume is part of the TomoAnchor-V3 project, an open-source tool for tomographic reconstruction [S1, unofficial]. It was developed to handle large volumetric datasets in medical imaging and computed tomography (CT) reconstruction, where memory constraints and computational demands require efficient data partitioning [S1, unofficial]. The implementation is found in the `tomoanchor_gpu_pdhg.py` and `tomoanchor_gpu_v3.py` modules, which utilize sharded volumes for GPU-accelerated reconstruction algorithms [S1, unofficial].

## Sources
[S1] alidarbandi/TomoAnchor-V3, https://github.com/alidarbandi/TomoAnchor-V3, fetched 2026-10-27 (unofficial)
