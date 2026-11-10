# Fixed-Sized Buckets for Isosurface Extraction

Also called: Fixed-Sized Buckets, Isosurface Extraction Using Fixed-Sized Buckets, SpanQuad, SpanQuad method, Waters-Co-Joy Method

## Overview
Fixed-Sized Buckets is an output-optimal algorithm for accelerated isosurface extraction from volumetric data sets, designed to identify active cells (cells intersected by the isosurface) with work proportional to the output size $K$ rather than the input size $N$ [S1]. It builds on [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) by replacing complex tree structures with a logical 2D array data structure that requires only standard sorting routines for construction [S1]. Unlike [Interval Table for Isosurface Extraction](4357-interval-table-for-isosurface-extraction.md) which may require quantized values, this method works equally well for floating-point data and offers a simple implementation with a small memory footprint [S1].

## How it works
**Inputs:** An array $R$ of $N$ cell records, where each record contains a `CellHandle` (pointer or index to the cell geometry), a `min` value (minimum intensity in the cell), and a `max` value (maximum intensity in the cell). A user-defined constant $B$ (bucket size). An isovalue $q$.
**Outputs:** A list of active cells (those where the isosurface intersects the cell).

**Data Structures:**
The method maintains a logical 2D array stored in a 1D array `m_cells` of size $N$.
- **m_cells**: An array of `Record` structs.
  - `struct Record`:
    - `CellHandle cell`: Handle to the cell.
    - `Type min`: Minimum intensity value.
    - `Type max`: Maximum intensity value.
- **m_min_dict**: An array of size `num_buckets`. Stores the `min` value of the last cell in each bucket.
- **m_last_bkt**: Index of the last bucket.
- **m_last_bkt_len**: Number of cells in the last bucket.

**Construction:**
1.  **Sort by Min:** Sort all $N$ records in $R$ by their `min` value in ascending order. Store the result in `m_cells`.
2.  **Calculate Buckets:** Compute `num_buckets = N / B + 1`. Set `m_last_bkt = num_buckets - 1`. Set `m_last_bkt_len = N - m_last_bkt * B`.
3.  **Build Min-Dictionary:** Allocate `m_min_dict` of size `num_buckets`.
    - For each bucket $i$ from $0$ to `m_last_bkt - 1`:
      - Set `m_min_dict[i] = m_cells[(i+1)*B - 1].min`.
    - Set `m_min_dict[m_last_bkt] = m_cells[N-1].min`.
4.  **Sort by Max:** Resort the cells within each bucket by their `max` value in descending order.
    - For each full bucket $i$ from $0$ to `m_last_bkt - 1`:
      - Sort the $B$ cells starting at `m_cells[i*B]` by `max` descending.
    - Sort the `m_last_bkt_len` cells starting at `m_cells[m_last_bkt*B]` by `max` descending.

**Full Extraction (Single Isovalue $q$):**
1.  **Identify Active Buckets:** Traverse buckets $c_0, c_1, \dots$ using `m_min_dict`.
    - Buckets $c_0$ to $c_{a-1}$ are "normal active buckets" where `m_min_dict[c] >= q`. These trivially satisfy the min-criterion for all cells in the bucket.
    - Bucket $c_a$ is the "last active bucket" where `m_min_dict[c_a] < q`.
    - Buckets $c_{a+1}$ and beyond are inactive because their minimum values exceed $q$ (assuming sorted order of bucket starts).
2.  **Process Normal Active Buckets ($c_0$ to $c_{a-1}$):**
    - For each bucket $c$ in this range:
      - Iterate through the cells in the bucket (sorted by `max` descending).
      - For each cell, if `cell.max >= q`, add `cell.cell` to the output.
      - Stop processing the bucket when `cell.max < q` or the end of the bucket is reached.
      - Because the bucket is sorted by `max` descending, active cells form a contiguous block at the start. Remaining cells are skipped.
3.  **Process Last Active Bucket ($c_a$):**
    - Let `lim` be `m_last_bkt_len` if $c_a$ is the last bucket, otherwise $B$.
    - Iterate through the cells in bucket $c_a$ (sorted by `max` descending).
    - For each cell, check both criteria:
      - If `cell.max >= q` AND `cell.min >= q`, add `cell.cell` to the output.
    - Stop when `cell.max < q` or the end of the bucket is reached.
    - Note: Active cells in $c_a$ may not be contiguous because the bucket is sorted by `max` but the min-criterion splits the bucket.

**Incremental Extraction (Isovalue $q' > q$):**
To support incremental updates, maintain markers for each bucket tracking the last visited cell during the previous extraction.
1.  **Identify New Active Buckets:** Determine the new set of active buckets $c_0, \dots, c_{a'}$.
2.  **Deactivate Old Cells:**
    - For buckets $c_0$ to $c_{a-1}$ (which remain active but may have lost cells):
      - Cells that were active in the previous pass but fail `cell.max >= q'` must be deactivated.
      - Use markers to skip cells known to be inactive.
3.  **Handle Interleaved Cells in $c_a$:**
    - Bucket $c_a$ remains active but may contain interleaved active/inactive cells.
    - Check both min and max criteria for cells starting from the previous marker.
4.  **Activate New Cells:**
    - For newly active buckets $c_{a+1}$ to $c_{a'}$:
      - Perform standard extraction (checking both criteria for the last bucket, or just max for normal buckets if applicable, though typically the new last bucket requires full check).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct(R, N, B):
    """
    R: List of Record objects with fields .min, .max, .cell
    N: Number of records
    B: Bucket size
    Returns: SpanQuad structure
    """
    # Step 1: Sort all cells by min ascending
    R.sort(key=lambda r: r.min)
    
    # Step 2: Calculate bucket parameters
    num_buckets = N // B + 1
    m_last_bkt = num_buckets - 1
    m_last_bkt_len = N - m_last_bkt * B
    
    # Step 3: Build min-dictionary
    m_min_dict = [0] * num_buckets
    for i in range(m_last_bkt):
        # Last cell in bucket i is at index (i+1)*B - 1
        m_min_dict[i] = R[(i + 1) * B - 1].min
    m_min_dict[m_last_bkt] = R[N - 1].min
    
    # Step 4: Sort each bucket by max descending
    for i in range(m_last_bkt):
        start_idx = i * B
        end_idx = start_idx + B
        # Sort slice by max descending
        R[start_idx:end_idx].sort(key=lambda r: r.max, reverse=True)
        
    start_idx = m_last_bkt * B
    end_idx = start_idx + m_last_bkt_len
    R[start_idx:end_idx].sort(key=lambda r: r.max, reverse=True)
    
    return {
        'cells': R,
        'min_dict': m_min_dict,
        'last_bkt': m_last_bkt,
        'last_bkt_len': m_last_bkt_len,
        'B': B
    }

def search(span_quad, q):
    """
    span_quad: Structure returned by construct
    q: Isovalue
    Returns: List of active cell handles
    """
    cells = span_quad['cells']
    min_dict = span_quad['min_dict']
    last_bkt = span_quad['last_bkt']
    last_bkt_len = span_quad['last_bkt_len']
    B = span_quad['B']
    
    active_cells = []
    cb = 0
    
    # Normal active buckets: min_dict[cb] >= q
    while cb != last_bkt and q >= min_dict[cb]:
        rec = 0
        start_idx = cb * B
        # Iterate through bucket, stop if max < q or end of bucket
        for i in range(start_idx, start_idx + B):
            if cells[i].max < q:
                break
            active_cells.append(cells[i].cell)
            rec += 1
        cb += 1
        
    # Last active bucket
    lim = last_bkt_len if cb == last_bkt else B
    start_idx = cb * B
    for i in range(start_idx, start_idx + lim):
        if cells[i].max < q:
            break
        # Check both min and max criteria for the last active bucket
        if q >= cells[i].min:
            active_cells.append(cells[i].cell)
            
    return active_cells
```

## Subtopics

### [Incremental Isosurface Extraction](1303-incremental-isosurface-extraction.md)
Incremental isosurface extraction is an optimization for isosurface extraction that reuses the state from a previous query (with isovalue $q$) to accelerate a new query (with isovalue $q'$), avoiding full re-scanning of the data structure by tracking active/inactive cell boundaries via markers. It builds on [Fixed-Sized Buckets for Isosurface Extraction](1252-fixed-sized-buckets-for-isosurface-extraction.md) by maintaining boundary markers that allow the algorithm to skip over cells whose active status is known to be unchanged, rather than re-evaluating every cell in the active range.

### [Triangular Matrix Data Redistribution](11596-triangular-matrix-data-redistribution.md)
Triangular Matrix Data Redistribution is a deterministic greedy algorithm used to approximate the optimal data distribution for load balancing in parallel isosurface extraction. It addresses the NP-hard problem of minimizing workload differences among processors by categorizing volume data blocks into a triangular matrix of buckets based on their functional value ranges. Unlike [Fixed-Sized Buckets for Isosurface Extraction](1252-fixed-sized-buckets-for-isosurface-extraction.md), which sorts cells by intensity ranges into fixed-sized buckets, this method explicitly constructs a triangular matrix in the range space to handle blocks with varying spans, ensuring that blocks with similar range spans are distributed evenly across processors.

## Operating characteristics
- **Time Complexity:** Construction is $O(N \log N)$, dominated by the first sorting pass. The second pass is $O((N/B) \cdot B \log B) = O(N \log B)$, which is subsumed if $B$ is constant [S1]. Initial sorting takes $O(N \log N)$ time. Worst-case performance is $O(N)$ [S1]. Average-case runtime complexity is $O(K)$, where $K$ is the number of active cells, making it output optimal [S1].
- **Memory Complexity:** The method incurs a $3 + 1/B$ factor increase in memory consumption relative to the input data, or $3 + 2/B$ if incremental search is desired [S1]. For large $B$, this factor is effectively 3 [S1]. This includes the storage for the cell records, the min-dictionary, and potentially markers for incremental extraction [S1].
- **Strengths:** Simple implementation requiring only standard sorting routines. Works with floating-point data. Small memory footprint. Output optimal on average. Supports incremental extraction [S1].
- **Limitations:** Worst-case performance is $O(N)$, which occurs when the isosurface intersects many cells in the last active bucket or when the data distribution is unfavorable. The last active bucket requires checking both min and max criteria for all cells, which can be costly if $B$ is large and the isosurface splits the bucket finely [S1].
- **When to use:** Suitable for interactive visualization where isovalue changes frequently (benefiting from incremental extraction) and where implementation simplicity is valued. Effective for large datasets where output size is small relative to input size [S1].
- **When not to use:** When worst-case performance guarantees are critical, or when memory is extremely constrained and $B$ cannot be chosen to minimize the overhead factor [S1].

## Background
The method was introduced by Kenneth W. Waters, Christopher S. Co, and Kenneth I. Joy at the University of California, Davis [S1]. It is presented in the paper "Isosurface Extraction Using Fixed-Sized Buckets" published in the Proceedings of EuroVis 2005 [S1]. The method is implemented in the SpanQuad class and has been demonstrated with Engine and Kyle unstructured flow simulation data sets [S1]. The authors note that the method is amenable to out-of-core implementation using out-of-core sorting algorithms [S1].

## Sources
[S1] Isosurface Extraction Using Fixed-Sized Buckets, https://doi.org/10.2312/vissym/eurovis05/207-214, fetched 2026-10-26
