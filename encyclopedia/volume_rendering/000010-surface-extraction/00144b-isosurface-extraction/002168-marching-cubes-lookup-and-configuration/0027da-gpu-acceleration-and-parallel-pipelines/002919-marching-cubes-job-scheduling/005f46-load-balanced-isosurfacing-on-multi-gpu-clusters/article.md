# Load-Balanced Isosurfacing on Multi-GPU Clusters

Also called: Cost-Heuristic Block Distribution, Preprocessing-Based Load Balancing

## Overview
This method efficiently distributes isosurfacing workloads across multiple GPUs in a cluster by using preprocessing to estimate the computational cost of volume blocks, maximizing load uniformity over a user-defined range of isovalues [S1]. Unlike naive round-robin assignment which yields unpredictable speedups due to data-dependent work variance, this technique uses a linear cost heuristic derived from triangle counts and hardware profiling to assign blocks to GPUs, ensuring balanced work distribution [S1]. It builds on [Marching Cubes Job Scheduling](4446-marching-cubes-job-scheduling.md) by providing the specific cost estimation and assignment algorithms required for multi-GPU clusters.

## How it works

### Inputs and Outputs
**Inputs:**
*   Volume data $V$ of resolution $R_x \times R_y \times R_z$, treated as an array of cuboid blocks of cells [S1].
*   User-defined salient isovalue range $[I_{min}, I_{max}]$ [S1].
*   Target cluster hardware configuration (GPU types, memory) [S1].

**Outputs:**
*   Block-to-GPU assignment map [S1].
*   Isosurface triangle vertex buffers for each block [S1].

### Data Structures

**Block Metadata (Preprocessing Output):**
*   Stored per block.
*   Contains $M$ entries. Each entry is a pair $(I_k, T_k)$, where $I_k$ is a probe isovalue and $T_k$ is the triangle count for that isovalue in the block [S1].

**Profiling Model:**
*   Linear function parameters $(C, L)$ for each GPU type [S1].
*   $C$: Constant overhead factor (ms).
*   $L$: Linear factor (ns per triangle).

**Isosurface Crossing Table:**
*   Byte-per-cell table at full block resolution [S1].
*   Stores indices of active cells (those crossing the isovalue) within each X row [S1].
*   One table stored per GPU [S1].

**Minimum-Maximum Table:**
*   Stores min/max values for contiguous spans of cells (length 10-20) within each X row [S1].
*   Used to skip spans where the isovalue is not present [S1].

### Algorithm Steps

#### Phase 1: Preprocessing
1.  **Divide Volume:** Partition the volume into roughly uniformly sized cuboid blocks. Typical block size is $128^3$ cells [S1].
2.  **Sort Data:** For each block, sort the data values in ascending order [S1].
3.  **Select Probe Isovalues:** To find $M$ probe isovalues for $N$ sample values, select the value at every $N/(M-1)$'th position in the sorted list [S1].
4.  **Count Triangles:** For each probe isovalue $I_k$:
    *   Iterate through all cells in the block on the CPU.
    *   Classify each cell into a marching cubes case.
    *   Lookup the triangle count for the case from a standard table.
    *   Sum the counts to get $T_k$ [S1].
5.  **Store Metadata:** Write the mapping $(I_k, T_k)$ for all $k \in [0, M-1]$ to a file for each block [S1].

#### Phase 2: Profiling
1.  **Generate Synthetic Volume:** Create a test volume of size similar to a block (e.g., $128^3$) using superimposed sinusoidal waves with random frequencies, directions, and amplitudes [S1].
2.  **Measure Times:** Compute isosurfaces for isovalues ranging from the minimum to the maximum value in the synthetic field. Record the isosurfacing time $t$ for each triangle count $T$ [S1].
3.  **Fit Linear Model:** Use linear least squares fitting to determine the parameters $C$ and $L$ for the equation:
    $$ \text{Time} = C + L \times T $$
    where Time is in milliseconds, $T$ is the number of triangles, $C$ is the constant overhead, and $L$ is the cost per triangle [S1].
    *   *Note:* Perform this independently for each GPU type in the cluster [S1].

#### Phase 3: Assignment
1.  **Compute Mean Triangle Count:** For each block, calculate the mean triangle count $\bar{T}$ within the user-defined salient isovalue range $[I_{min}, I_{max}]$.
    *   Use linear interpolation between the preprocessed probe isovalues to estimate $T$ for any isovalue in the range.
    *   Average these estimates over the range [S1].
2.  **Evaluate Cost Heuristic:** Compute the cost $H$ for each block using the profiling model:
    $$ H = C + L \times \bar{T} $$
    [S1].
3.  **Initial Sort and Round-Robin:**
    *   Sort all blocks by descending cost $H$ [S1].
    *   Assign blocks to GPUs in round-robin fashion [S1].
4.  **Refinement (Variance Minimization):**
    *   Initialize a counter for unsuccessful exchanges to 0 [S1].
    *   Loop until the counter exceeds a limit or variance drops below a threshold:
        1.  Pick a random pair of blocks $(B_i, B_j)$ assigned to different GPUs $(G_a, G_b)$ [S1].
        2.  Calculate the variance of the sum of costs $H$ across all GPUs before and after swapping $B_i$ and $B_j$ [S1].
        3.  If the swap decreases the variance, apply the swap and reset the unsuccessful counter to 0 [S1].
        4.  If the swap does not decrease the variance, increment the unsuccessful counter [S1].
    [S1].

#### Phase 4: Block Isosurfacing (Per GPU)
Each GPU processes its assigned blocks sequentially or in parallel depending on resources. For each block:

**Pass 1: Counting and Offsets**
1.  **Triangle Counting:** Compute the total triangle count for the block and the offset of triangles into the vertex buffer for each X row of cells. Do not compute spatial positions [S1].
    *   *Optimization:* Use the Minimum-Maximum Table to skip spans of cells where the isovalue is not in $[min, max]$ [S1].
2.  **Prefix Sums:** Apply exclusive prefix sums per X row to compute exact indices within the output vertex buffer for output triangles associated with each row [S1].
    *   *Implementation:* Perform many independent serial prefix sums in parallel on the GPU, as the lists are small and of nearly uniform length [S1].
3.  **Crossing Table:** Record indices of active cells within each X row in the Isosurface Crossing Table [S1].

**Pass 2: Creation**
1.  **Thread Assignment:** Assign each GPU thread to one X row of cells [S1].
2.  **Triangle Generation:** Compute isosurface triangles and normals for the assigned row.
    *   *Optimization:* Iterate through the Isosurface Crossing Table to apply marching cubes only to active cells [S1].
3.  **Buffer Writing:** Place triangles into the target vertex buffer using the offsets computed in Pass 1 [S1].
4.  **Normals:** Compute normals for each vertex using finite differences on the gradient. Use ghost cells around blocks to ensure consistency [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def preprocess_block(block_data, M):
    """
    Preprocessing phase for a single block.
    block_data: list of scalar values in the block.
    M: number of probe isovalues.
    Returns: list of (isovalue, triangle_count) pairs.
    """
    sorted_data = sorted(block_data)
    N = len(sorted_data)
    probe_isovalues = []
    
    # Select M probe isovalues uniformly from sorted data
    for i in range(M):
        index = int(i * (N / (M - 1)))
        probe_isovalues.append(sorted_data[index])
        
    results = []
    for iso in probe_isovalues:
        # Count triangles for this isovalue
        t_count = count_marching_cubes_triangles(block_data, iso)
        results.append((iso, t_count))
        
    return results

def count_marching_cubes_triangles(data, isovalue):
    """
    Counts triangles for a block of data at a specific isovalue.
    Assumes data is a 3D grid flattened or accessible via indices.
    """
    # Placeholder for actual marching cubes case classification logic
    # which iterates cells, classifies cases, and sums triangle counts.
    # This is a conceptual representation.
    total_triangles = 0
    # Iterate over all cells in the block
    for cell in data:
        case = classify_marching_cubes_case(cell, isovalue)
        total_triangles += TRIANGLE_COUNT_TABLE[case]
    return total_triangles

def profile_gpu(block_size, num_samples=100):
    """
    Profiling phase. Determines C and L for a specific GPU.
    block_size: dimensions of a block (e.g., 128).
    Returns: (C, L) tuple.
    """
    # Generate synthetic volume
    synthetic_vol = generate_synthetic_volume(block_size)
    
    min_val = min(synthetic_vol)
    max_val = max(synthetic_vol)
    
    triangle_counts = []
    times = []
    
    # Sample isovalues from min to max
    for i in range(num_samples):
        iso = min_val + (max_val - min_val) * (i / (num_samples - 1))
        t_count = count_marching_cubes_triangles(synthetic_vol, iso)
        # Measure time to isosurface this block
        t_time = measure_isosurfacing_time(synthetic_vol, iso)
        triangle_counts.append(t_count)
        times.append(t_time)
        
    # Linear least squares fitting: Time = C + L * T
    # Using standard linear regression formulas
    n = len(triangle_counts)
    sum_t = sum(triangle_counts)
    sum_time = sum(times)
    sum_tt = sum(t*t for t in triangle_counts)
    sum_t_time = sum(t*ti for t, ti in zip(triangle_counts, times))
    
    denominator = n * sum_tt - sum_t * sum_t
    if denominator == 0:
        return (0.0, 0.0) # Avoid division by zero
        
    L = (n * sum_t_time - sum_t * sum_time) / denominator
    C = (sum_time - L * sum_t) / n
    
    return (C, L)

def assign_blocks(blocks_metadata, gpu_costs, salient_range, max_unsuccessful=1000):
    """
    Assignment phase.
    blocks_metadata: list of dicts, each containing 'probe_data' (list of (iso, count))
    gpu_costs: dict mapping gpu_id -> (C, L)
    salient_range: tuple (I_min, I_max)
    Returns: list of gpu_ids for each block.
    """
    # Step 1: Compute cost for each block
    block_costs = []
    for i, meta in enumerate(blocks_metadata):
        # Interpolate mean triangle count over salient range
        mean_t = compute_mean_triangle_count(meta['probe_data'], salient_range)
        
        # Use the cost model of the first GPU (or average if multiple types)
        # For simplicity, assume homogeneous GPUs or pick one representative
        C, L = gpu_costs['gpu_0']
        cost = C + L * mean_t
        block_costs.append({'id': i, 'cost': cost})
        
    # Step 2: Sort by descending cost
    block_costs.sort(key=lambda x: x['cost'], reverse=True)
    
    # Step 3: Round-robin assignment
    num_gpus = len(gpu_costs)
    assignments = [None] * len(block_costs)
    for idx, block in enumerate(block_costs):
        gpu_id = idx % num_gpus
        assignments[block['id']] = gpu_id
        
    # Step 4: Refinement
    # Calculate initial variance
    gpu_sums = {g: 0.0 for g in gpu_costs}
    for block in block_costs:
        gpu_id = assignments[block['id']]
        gpu_sums[gpu_id] += block['cost']
        
    variance = compute_variance(gpu_sums.values())
    unsuccessful = 0
    
    while unsuccessful < max_unsuccessful:
        # Pick random pair of blocks on different GPUs
        b1_idx, b2_idx = random.sample(range(len(block_costs)), 2)
        b1 = block_costs[b1_idx]
        b2 = block_costs[b2_idx]
        
        g1 = assignments[b1['id']]
        g2 = assignments[b2['id']]
        
        if g1 == g2:
            unsuccessful += 1
            continue
            
        # Calculate variance after swap
        # Current sums
        s1 = gpu_sums[g1]
        s2 = gpu_sums[g2]
        
        # New sums after swap
        new_s1 = s1 - b1['cost'] + b2['cost']
        new_s2 = s2 - b2['cost'] + b1['cost']
        
        # Create new sums dict for variance calculation
        new_sums = gpu_sums.copy()
        new_sums[g1] = new_s1
        new_sums[g2] = new_s2
        
        new_variance = compute_variance(new_sums.values())
        
        if new_variance < variance:
            # Apply swap
            assignments[b1['id']] = g2
            assignments[b2['id']] = g1
            gpu_sums[g1] = new_s1
            gpu_sums[g2] = new_s2
            variance = new_variance
            unsuccessful = 0
        else:
            unsuccessful += 1
            
    return assignments

def compute_mean_triangle_count(probe_data, salient_range):
    """
    Interpolates triangle counts over the salient range and returns the mean.
    probe_data: list of (iso, count) sorted by iso.
    """
    I_min, I_max = salient_range
    # Linear interpolation function
    def get_t(iso):
        # Find bracketing probes
        for i in range(len(probe_data) - 1):
            i1, t1 = probe_data[i]
            i2, t2 = probe_data[i+1]
            if i1 <= iso <= i2:
                # Linear interpolation
                if i2 == i1: return t1
                return t1 + (t2 - t1) * (iso - i1) / (i2 - i1)
        # If outside range, clamp to nearest
        if iso < probe_data[0][0]: return probe_data[0][1]
        return probe_data[-1][1]
        
    # Sample mean over range
    # For implementation, integrate or sample densely
    num_samples = 100
    total_t = 0
    for i in range(num_samples):
        iso = I_min + (I_max - I_min) * (i / (num_samples - 1))
        total_t += get_t(iso)
        
    return total_t / num_samples

def compute_variance(values):
    n = len(values)
    if n == 0: return 0
    mean = sum(values) / n
    return sum((v - mean)**2 for v in values) / n
```

### Parameters
*   **Block Size:** $128^3$ cells. Chosen as a compromise between overhead (smaller blocks) and load balancing granularity (larger blocks) [S1].
*   **Salient Isovalue Range:** User-defined range $[I_{min}, I_{max}]$. Load balancing is maximized over this set [S1].
*   **Span Length (Optimization):** 10-20 cells for the Minimum-Maximum Table [S1].

### Edge Cases and Pitfalls
*   **Overhead Dominance:** Even blocks with very few triangles may take substantial time to compute due to API and kernel startup overhead. The heuristic must account for this constant factor $C$ [S1].
*   **Heuristic Conditioning:** The heuristic must be well-conditioned to avoid unreasonably large changes in estimates for small changes in input metadata [S1].
*   **Metadata Completeness:** The heuristic must be computable from compact metadata produced by preprocessing, as not all data can be loaded every time the heuristic is evaluated [S1].
*   **Hardware Specificity:** The relationship between overhead and actual work varies by platform. The heuristic must reflect the specific hardware platform being used [S1].
*   **Arbitrary Assignment:** Arbitrary block assignment (no cost heuristic) leads to varied and undefined performance because work distribution uniformity depends on the specific isovalue and dataset [S1].

## Operating characteristics

**Time and Memory Complexity:**
*   **Preprocessing:** $O(B \cdot N \cdot M)$, where $B$ is the number of blocks, $N$ is the number of cells per block, and $M$ is the number of probe isovalues. Sorting takes $O(N \log N)$ per block [S1].
*   **Profiling:** $O(S \cdot T_{gpu})$, where $S$ is the number of synthetic samples and $T_{gpu}$ is the time to isosurface one block [S1].
*   **Assignment:** $O(B \log B)$ for sorting, plus refinement iterations. Refinement is bounded by a limit on unsuccessful exchanges [S1].
*   **Isosurfacing:** $O(N)$ per block for counting and creation, with optimizations reducing constant factors [S1].
*   **Memory:** Metadata is compact ($M$ entries per block). Isosurface Crossing Table requires 1 byte per cell per GPU [S1].

**Measured Performance:**
*   **Platform:** 12 Linux nodes, each with 2 NVIDIA Quadro FX5600s (1.5GiB memory), 2 quad-core AMD Opteron 2350 CPUs at 2GHz, 16GiB RAM, Infiniband [S1].
*   **Constant Factor:** ~1.2ms per $128^3$ block on a single GPU, dominated by API and kernel startup time [S1].
*   **Linear Factor:** ~40ns per triangle on a single GPU [S1].
*   **Throughput with Heuristic:** ~250 million triangles per second on 24 GPUs for isovalue range 2.75-3.25 [S1].
*   **Throughput without Heuristic:** ~175 million triangles per second on 24 GPUs for the same range [S1].
*   **Scalability:** The cost heuristic yields substantial performance improvements over naive round-robin distribution, particularly within the salient isovalue range [S1].

**Strengths and Limitations:**
*   **Strengths:** Maximizes load uniformity over user-defined isovalues; enables improved scalability over naive approaches; handles large volumes that do not fit on a single node [S1].
*   **Limitations:** Requires preprocessing and profiling phases; performance peaks only within the salient isovalue range (though other ranges do not suffer unacceptable penalties); small scale variation in results occurs due to noise in isosurfacing times and cost heuristic accuracy [S1].

## Background

**Origin:**
The method was proposed by Steven Martin, Han-Wei Shen, and Patrick McCormick from The Ohio State University and Los Alamos National Laboratory [S1]. It was published in the Eurographics Symposium on Parallel Graphics and Visualization (EGPGV) in 2010 [S1].

**History and Development:**
The technique addresses the challenge of distributing isosurfacing workloads on increasingly hierarchical parallel architectures (clusters with multiple nodes, each with multiple GPUs) [S1]. It improves upon earlier cost heuristics by Zhang et al. [ZN03], which used active cells and hardcoded coefficients, and Scheidegger et al. [SSD*08], which used isosurface statistics. This method uses sampled triangle counts and linear regression based on profiling the target system, providing a more accurate and less complex cost determination [S1].

**Usage:**
Used in multi-GPU cluster environments for isosurfacing large volume datasets, specifically tested on NVIDIA Quadro FX5600 GPUs [S1].

## Sources

[S1] Load-Balanced Isosurfacing on Multi-GPU Clusters, https://doi.org/10.2312/egpgv/egpgv10/091-100, fetched 2026-10-26
