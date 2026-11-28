# Spherical Harmonic Clustering

Also called: ClusterSHs, K-Means Clustering for SH, SH Clustering

## Overview
Spherical Harmonic Clustering reduces the memory footprint of Gaussian Splatting assets by clustering spherical harmonic (SH) coefficients using K-Means, trading precision for storage efficiency [S1, unofficial].

## How it works
**Inputs:** A source splat data structure (`InputSplatData`) containing SH coefficients [S1, unofficial].
**Outputs:** A compressed SH table (`GaussianSplatAsset.SHTableItemFloat16`) where each entry represents a cluster center [S1, unofficial].

**Data Structures:**
1. **`shData`**: A temporary `float` array of size `splatData.Length * kShDim` [S1, unofficial]. It stores the raw SH coefficients from the input [S1, unofficial].
2. **`shMeans`**: A `float` array of size `shCount * kShDim` [S1, unofficial]. It stores the resulting cluster centers (means) after K-Means convergence [S1, unofficial]. `shCount` is the number of clusters [S1, unofficial].
3. **`shIndices`**: An integer array of size `splatData.Length` [S1, unofficial]. It maps each original splat to its assigned cluster index [S1, unofficial].
4. **`shs`**: The final output array of type `GaussianSplatAsset.SHTableItemFloat16` with size `shCount` [S1, unofficial].
5. **`ConvertSHClustersJob`**: A compute job structure with `m_Input` as a `NativeArray<float>` reinterpreted as `float3`, and `m_Output` as a `NativeArray<GaussianSplatAsset.SHTableItemFloat16>` [S1, unofficial].

**Parameters:**
* `kShDim`: Constant dimension of SH data per splat, equal to $15 \times 3 = 45$ floats [S1, unofficial].
* `kBatchSize`: Constant batch size for K-Means calculation, equal to 2048 [S1, unofficial].
* `passesOverData`: Number of passes over the data for K-Means. Values depend on the target cluster count format: Cluster64k is 0.3, Cluster32k is 0.4, Cluster16k is 0.5, Cluster8k is 0.8, and Cluster4k is 1.2 [S1, unofficial].

**Algorithm Steps:**
1. **Check Necessity:** Calculate `shCount` using `GaussianSplatAsset.GetSHCount(format, splatData.Length)` [S1, unofficial]. If `shCount >= splatData.Length`, return early as no clustering is needed [S1, unofficial].
2. **Gather SH Data:** Allocate `shData` of size `splatData.Length * kShDim` [S1, unofficial]. Copy SH coefficients from `splatData` into `shData` using `GatherSHs`, which copies 45 floats starting at offset 9 from each splat's data block [S1, unofficial].
3. **K-Means Clustering:** Allocate `shMeans` of size `shCount * kShDim` and `shIndices` of size `splatData.Length` [S1, unofficial]. Call `KMeansClustering.Calculate` with parameters `kShDim`, `shData`, `kBatchSize`, `passesOverData`, a progress callback, `shMeans`, and `shIndices` [S1, unofficial]. Dispose of `shData` after this step [S1, unofficial].
4. **Quantize and Pack:** Allocate `shs` of size `shCount` [S1, unofficial]. Create a `ConvertSHClustersJob` with `m_Input` set to `shMeans` reinterpreted as `float3` and `m_Output` set to `shs` [S1, unofficial]. Schedule and complete the job with a batch size of 256 [S1, unofficial]. The job maps each input float to a half-precision 3-component vector (`half3`) [S1, unofficial]. Dispose of `shMeans` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_harmonic_clustering(splat_data, format, progress_callback):
    # Step 1: Check necessity
    sh_count = GaussianSplatAsset.GetSHCount(format, len(splat_data))
    if sh_count >= len(splat_data):
        return None  # No clustering needed

    k_sh_dim = 45  # 15 * 3
    k_batch_size = 2048
    
    # Step 2: Gather SH Data
    sh_data = [0.0] * (len(splat_data) * k_sh_dim)
    for i in range(len(splat_data)):
        src_offset = 9
        for j in range(k_sh_dim):
            sh_data[i * k_sh_dim + j] = splat_data[i][src_offset + j]

    # Step 3: K-Means Clustering
    sh_means = [0.0] * (sh_count * k_sh_dim)
    sh_indices = [0] * len(splat_data)
    
    passes_map = {
        "Cluster64k": 0.3,
        "Cluster32k": 0.4,
        "Cluster16k": 0.5,
        "Cluster8k": 0.8,
        "Cluster4k": 1.2
    }
    passes = passes_map.get(format, 1.0)
    
    # Standard K-Means clustering routine [G]
    # k_means_calculate(k_sh_dim, sh_data, k_batch_size, passes, progress_callback, sh_means, sh_indices)

    sh_data = None  # Dispose shData

    # Step 4: Quantize and Pack
    shs = [None] * sh_count
    
    # ConvertSHClustersJob execution
    for i in range(sh_count):
        addr = i * 15
        res = SHTableItemFloat16()
        for c in range(5):
            base_idx = addr + c * 3
            # Standard IEEE 754 half-precision conversion [G]
            r = float_to_half(sh_means[base_idx])
            g = float_to_half(sh_means[base_idx + 1])
            b = float_to_half(sh_means[base_idx + 2])
            res.sh[c] = (r, g, b)
        shs[i] = res
            
    sh_means = None  # Dispose shMeans
    
    return shs
```

**Edge Cases and Pitfalls:**
* **Precision Loss:** Converting from `float` to `half` (16-bit) during the packing step introduces quantization error [S1, unofficial].
* **Early Return:** If the target cluster count is not smaller than the input splat count, the method returns early to avoid unnecessary computation [S1, unofficial].
* **Memory Management:** Temporary arrays (`shData`, `shMeans`) must be explicitly disposed of to prevent memory leaks, as indicated by the use of persistent allocators [S1, unofficial].

## Subtopics

### Sub-Vector Quantization (SVQ) for Gaussian Features
Splits vectors into sub-vectors and quantizes them independently. Choose when finer-grained control over error distribution is needed compared to global clustering. [Link](13486-sub-vector-quantization-svq-for-gaussian-features.md)

### Illumination-Aware Data Transformation
Adjusts data based on lighting conditions before processing. Choose when lighting variations significantly impact the effectiveness of standard clustering. [Link](16293-illumination-aware-data-transformation.md)

### Voxel Clustering (Global)
Clusters voxels based on global properties. Choose when spatial locality is less important than global statistical similarity. [Link](16560-voxel-clustering-global.md)

## Operating characteristics
* **Time Complexity:** Dominated by the K-Means clustering step, which runs for `passesOverData` iterations over `splatData.Length` items. The constant factor is influenced by `kShDim` (45) and `kBatchSize` (2048) [S1, unofficial].
* **Memory Complexity:** Requires temporary storage for `shData` ($N \times 45$ floats) and `shMeans` ($K \times 45$ floats), where $N$ is splat count and $K$ is cluster count. The final output array `shs` has size `shCount` and stores `GaussianSplatAsset.SHTableItemFloat16` entries [S1, unofficial].
* **Strengths:** Significantly reduces memory usage for SH coefficients, which can be large for high-order harmonics [S1, unofficial].
* **Limitations:** Loss of precision due to clustering and half-precision quantization. Performance depends on the number of passes, which varies by target cluster count [S1, unofficial].
* **When to Use:** When memory bandwidth or storage is a bottleneck and slight color fidelity loss is acceptable.
* **When Not to Use:** When high-fidelity color reproduction is critical or when the number of splats is small enough that compression benefits are negligible.

## Background
This method is part of the preprocessing pipeline for Gaussian Splatting assets in Unity, specifically within the `worldlabs_unity` implementation. It is designed to optimize the rendering of large-scale Gaussian Splat scenes by reducing the size of the SH data stored in GPU memory [S1, unofficial].

## Sources
[S1] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
