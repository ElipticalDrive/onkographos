# Bin-Hash Index Construction

Also called: Bin-Hash, Bin-Hash Index, GPU-based Query-Driven Visualization with Bin-Hash Index, GPU-based query indexing structure, Multi-resolution Bin-Hash Index

## Overview
The Bin-Hash Index Construction is a GPU-based indexing strategy designed to accelerate query processing and rendering for adaptive mesh refinement (AMR) data [S1]. It solves the problem of efficiently querying large, spatiotemporally dynamic datasets by utilizing multi-resolution information to reduce memory bandwidth and computational overhead during the query phase [S1]. Unlike previous query-driven visualization (QDV) methods that focused on regular uniform grids or relied on CPU-based indexing, this method extends QDV to AMR data and executes the entire pipeline—including index building, searching, and rendering—on the GPU [S1]. It sets itself apart by generating both an encoded low-resolution version and a spatially compacted full-resolution version of the data, enabling fast coarse filtering followed by precise full-resolution access only when necessary [S1].

## How it works
The Bin-Hash index construction algorithm takes as input the full-resolution AMR data from a single timestep (after synchronization with a composite template) and generates two outputs: a low-resolution encoded array and a set of spatially compacted hash tables for full-resolution data [S1]. The construction is performed in two distinct stages: Binning and Partitioning/Spatial Compaction [S1].

### Inputs and Outputs
*   **Input:** Full-resolution AMR data records for a selected timestep. Each record consists of a 32-bit base data value (scalar or vector) and its spatial location within the AMR hierarchy [S1].
*   **Output:**
    1.  **Low-resolution data:** An array of 8-bit integers representing bin assignments.
    2.  **Hash/Offset Table Pairs:** 256 pairs of tables (one pair per bin) containing the full-resolution data for records falling into that bin [S1].

### Data Structures
The index consists of the following components [S1]:

1.  **Low-Resolution Array:**
    *   **Layout:** A contiguous array where each element corresponds to a data record.
    *   **Type:** 8-bit unsigned integer (`uint8_t`).
    *   **Values:** Integers in the range [0, 255], representing the bin index.

2.  **Hash and Offset Tables (per bin $i$):**
    *   For each bin $i \in [0, 255]$, there is a pair of tables generated via perfect spatial hashing [S1].
    *   **Hash Table:** Stores the spatial hash keys for records in bin $i$.
    *   **Offset Table:** Stores the offsets or pointers to the actual full-resolution data values for records in bin $i$.
    *   **Note:** The exact internal layout of the hash and offset tables (e.g., open addressing vs. chaining, specific hash function) is not detailed in the source, but the method relies on "perfect spatial hashing" [S1].

### Parameters
*   **Number of Bins:** 256 [S1].
*   **Bin Encoding:** 8-bit (since $2^8 = 256$) [S1].
*   **Full-Resolution Data Type:** 32-bit (base data) [S1].
*   **Storage Overhead:** Approximately 1.5 to 2.0 times the size of the original AMR data [S1].

### Algorithm Steps

**Stage 1: Binning**
1.  **Define Bin Boundaries:** Determine bin boundaries $(b_0, b_1, \dots, b_{256})$ for the variable $A$ being indexed. The boundaries are chosen such that each bin contains approximately the same number of records [S1].
2.  **Assign Bin Numbers:** For each data record with value $v$, determine the bin index $k$ such that $b_k \le v < b_{k+1}$ [S1].
3.  **Create Low-Resolution Data:** Replace each 32-bit full-resolution data value with its associated 8-bit bin number $k$ (where $0 \le k \le 255$) [S1].

**Stage 2: Partitioning and Spatial Compaction**
1.  **Partition Records:** Group all data records according to their assigned bin numbers from Stage 1 [S1].
2.  **Apply Perfect Spatial Hashing:** For each bin $i$:
    *   Take the subset of full-resolution data records associated with bin $i$.
    *   Apply perfect spatial hashing to these records [S1].
    *   Store the results as two small tables: a hash table and an offset table [S1].
3.  **Repeat for All Bins:** Perform Step 2 for all 256 bins, resulting in 256 pairs of hash and offset tables [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bin_hash_index_construction(full_resolution_data, num_bins=256):
    """
    Constructs the Bin-Hash index for a single timestep of AMR data.
    
    Args:
        full_resolution_data: List of tuples (record_id, value_32bit, spatial_coords)
        num_bins: Number of bins (fixed at 256 per source)
        
    Returns:
        low_res_array: Array of 8-bit bin indices
        hash_offset_tables: List of 256 tuples (hash_table, offset_table)
    """
    
    # Stage 1: Binning
    
    # 1. Determine bin boundaries to ensure approximately equal record counts
    # Note: The source does not specify the exact algorithm for boundary calculation
    # (e.g., histogram-based, quantile-based), only that bins contain approx. same records.
    bin_boundaries = compute_equal_count_boundaries(full_resolution_data, num_bins)
    
    # 2. Assign bin numbers and create low-resolution array
    low_res_array = []
    for record in full_resolution_data:
        value = record.value_32bit
        # Find bin index k such that bin_boundaries[k] <= value < bin_boundaries[k+1]
        bin_index = find_bin_index(value, bin_boundaries)
        low_res_array.append(bin_index)
        
    # Stage 2: Partitioning and Spatial Compaction
    
    # 1. Partition records by bin
    bins = [[] for _ in range(num_bins)]
    for i, record in enumerate(full_resolution_data):
        bin_idx = low_res_array[i]
        bins[bin_idx].append(record)
        
    # 2. Apply perfect spatial hashing for each bin
    hash_offset_tables = []
    for i in range(num_bins):
        records_in_bin = bins[i]
        
        # Apply perfect spatial hashing to generate hash and offset tables
        # The source cites "perfect spatial hashing [17]" but does not provide
        # the specific algorithm or formula. This step is a placeholder for
        # the specific hashing implementation required.
        hash_table, offset_table = perfect_spatial_hashing(records_in_bin)
        
        hash_offset_tables.append((hash_table, offset_table))
        
    return low_res_array, hash_offset_tables

def compute_equal_count_boundaries(data, num_bins):
    """
    Computes boundaries such that each bin contains approximately the same number of records.
    Implementation detail: Not given in the sources. [G]
    Typically involves sorting values and splitting into quantiles.
    """
    values = sorted([d.value_32bit for d in data])
    n = len(values)
    boundaries = []
    for i in range(1, num_bins):
        # Approximate median split for equal counts
        idx = int(i * n / num_bins)
        boundaries.append(values[idx])
    return [min(values)] + boundaries + [max(values) + 1]

def find_bin_index(value, boundaries):
    """
    Finds the bin index for a value given boundaries.
    """
    for i in range(len(boundaries) - 1):
        if boundaries[i] <= value < boundaries[i+1]:
            return i
    return len(boundaries) - 1

def perfect_spatial_hashing(records):
    """
    Placeholder for the perfect spatial hashing algorithm.
    The source states this technique stores full-resolution data as two small tables:
    a hash table and an offset table.
    Implementation detail: Not given in the sources. [G]
    """
    # This function would implement the specific perfect hashing scheme
    # referenced in [S1] (citation [17]).
    return [], []
```

### Edge Cases and Pitfalls
*   **Empty Bins:** If a bin contains no records, the corresponding hash and offset tables will be empty. The search process must handle this case gracefully [S1].
*   **Precision:** The low-resolution data uses 8-bit integers, which may lead to quantization errors if the bin boundaries are not chosen carefully. The source specifies that bins should contain approximately the same number of records to mitigate this [S1].
*   **Memory Overhead:** The total storage overhead is 1.5 to 2.0 times the original data size. This must be accounted for in GPU memory allocation [S1].

## Operating characteristics
*   **Time Complexity:** The construction time depends on the binning and hashing steps. The source does not provide explicit Big-O notation for construction. The search process is accelerated by the GPU-based index [S1].
*   **Memory Complexity:** Approximately 1.5 to 2.0 times the size of the original AMR data [S1].
*   **Performance:** The method accelerates query processing and rendering by utilizing multi-resolution information and executing the entire pipeline on the GPU [S1]. It is designed for interactive and resource-efficient exploration of large multidimensional data [S1].
*   **Strengths:**
    *   Accelerates query performance on AMR data [S1].
    *   Efficiently utilizes GPU memory [S1].
    *   Supports scalar, vector, and arbitrary timestep/variable queries [S1].
*   **Limitations:**
    *   Requires synchronization of timesteps to a common AMR template [S1].
    *   The specific details of the perfect spatial hashing algorithm are not provided in the source [S1].
*   **When to Use:** When visualizing time-varying AMR data on a GPU and requiring fast query-driven visualization [S1].
*   **When Not to Use:** When CPU-based processing is preferred or when data is not adaptively refined (regular uniform grids may use different QDV methods) [S1].

## Background
The Bin-Hash index was introduced in the context of Query-Driven Visualization (QDV) of time-varying Adaptive Mesh Refinement (AMR) data [S1]. Previous QDV work focused on regular uniform grids and often relied on CPU-based indexing [S1]. This method extends QDV to AMR data, which has spatially dynamic grid structures, and is the first GPU-based QDV approach that utilizes a GPU-based indexing strategy [S1]. It was implemented on NVIDIA CUDA for GPU-based query engines, with the CPU acting as a host to stream minimal data for full-resolution queries [S1]. The method builds on the concept of multi-resolution information to accelerate query processing [S1].

## Sources
[S1] Query-Driven Visualization of Time-Varying Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc897115/?q=amr volume rendering, fetched 2026-10-28
