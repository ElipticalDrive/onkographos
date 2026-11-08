# MPI Collective Noncontiguous Read for Octree Data
Also called: MPI-IO Collective Read for Octree Blocks, Single Collective and Noncontiguous Read (MPI-IO)

## Overview
This method is a parallel I/O strategy for reading large volume datasets stored as hexahedral cell data reconstructed from node data based on octree blocks. It uses MPI-IO collective operations to allow input processors to fetch roughly equal numbers of cells from disk simultaneously. It sets itself apart from independent reading strategies by relying on `MPI_FILE_READ_ALL` to handle the noncontiguous nature of octree block data, thereby avoiding the need for post-read merging of scattered data chunks at the rendering stage [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset consisting of node data and octree structure data (1D arrays) representing hexahedral cells. The data is distributed across $m$ input processors.
*   **Output:** Each input processor holds a subset of the current time step's cell data, organized according to the octree blocks assigned to it, ready for distribution to rendering processors.

**Data Structures:**
*   **Octree Data:** Represented as 1D arrays containing node information and structural hierarchy.
*   **Cell Data:** Hexahedral cells reconstructed from the node data.
*   **Derived Data Type:** An MPI datatype created to describe the noncontiguous memory layout of the specific octree block data to be read.

**Steps:**
1.  **Grouping:** Group cell data according to the octree data structure and the load balancing strategy. This ensures that the data required for specific rendering tasks is logically grouped.
2.  **Merging Octree Data:** Merge octree data for each rendering processor to avoid duplicating node data across input processors. This step ensures that the mapping between octree blocks and their constituent node data is consistent and non-redundant before the I/O operation [S1].
3.  **Derive Data Type:** Each input processor uses `MPI_TYPE_CREATE_INDEXED_BLOCK` to derive a data type from the octree data. This derived data type describes the specific reading pattern (the noncontiguous sequence of nodes/cells) required for the octree blocks assigned to that processor [S1].
4.  **Set View:** Use `MPI_FILE_SET_VIEW` to set the derived data type as the reading pattern for the current input processor. This configures the MPI-IO file view to match the noncontiguous layout defined in step 3 [S1].
5.  **Collective Read:** Use `MPI_FILE_READ_ALL` to collectively read the data. All input processors execute this call simultaneously, fetching their respective subsets of cell data from disk along with the other input processors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume:
# - file_handle: Opened MPI file handle
# - octree_data: Local array of octree structure data for this processor
# - cell_data: Buffer to store the read cell data
# - m: Total number of input processors

# Step 1 & 2: Grouping and Merging are logical/preparation steps
# handled by the application logic prior to I/O.
# The octree data is merged so that each processor knows exactly
# which nodes/cells belong to its assigned blocks without duplication.

# Step 3: Derive the data type
# Create an indexed block type that describes the noncontiguous
# memory layout of the cell data corresponding to the octree blocks.
# The 'indices' array contains the offsets (in terms of basic datatype)
# of the data elements to be read.
# The 'blocklengths' array contains the number of elements in each block.
# The 'basic_type' is the datatype of the individual data elements (e.g., float).
derived_type = MPI_TYPE_CREATE_INDEXED_BLOCK(
    blocklengths=octree_block_lengths,
    indices=octree_block_offsets,
    basic_type=MPI_FLOAT
)
MPI_TYPE_COMMIT(derived_type)

# Step 4: Set the file view
# Configure the MPI-IO file view to use the derived type.
# This tells MPI how to interpret the file offsets for this processor.
MPI_FILE_SET_VIEW(
    file=file_handle,
    disp=processor_file_offset, # Offset in file for this processor's data
    etype=MPI_FLOAT,           # Elementary type for displacement calculation
    filetype=derived_type,     # The noncontiguous type derived above
    order='native'             # Native byte order
)

# Step 5: Collective Read
# All input processors read their respective data simultaneously.
# MPI_FILE_READ_ALL ensures synchronization and collective I/O optimization.
MPI_FILE_READ_ALL(
    file=file_handle,
    buf=cell_data,             # Buffer to store the read data
    count=total_elements,      # Total number of elements to read
    datatype=MPI_FLOAT,        # Datatype of the buffer elements
    status=MPI_STATUS_IGNORE
)

# Cleanup
MPI_TYPE_FREE(derived_type)
```

**Parameters and Edge Cases:**
*   **$m$ (Input Processors):** The number of processors performing the I/O. The strategy assumes $m$ is chosen to balance the load.
*   **Precision:** Standard floating-point precision is assumed for the cell data.
*   **Pitfalls:** If the octree data is not properly merged, node data duplication may occur, leading to incorrect cell reconstruction or wasted memory. The `MPI_TYPE_CREATE_INDEXED_BLOCK` must accurately reflect the file layout; any mismatch will result in corrupted data reads [S1].

## Subtopics

### MPI Independent Contiguous Read
An alternative file reading strategy where each input processor independently reads a contiguous chunk ($m_1$) of node data, then scans octree data to create a mapping to local node data, forwarding both to rendering processors. Rendering processors merge incoming data. Steps: 1. Each input processor independently reads a contiguous chunk ($m_1$) of node data. 2. Each input processor scans octree data to create a mapping between local node data and octree blocks. 3. Each input processor forwards node data and the map to rendering processors according to load balancing. 4. Each rendering processor merges incoming data to form complete local octree blocks of data. No inter-processor communication is needed for merging. This strategy is superior if the overhead of collective I/O becomes too high [S1].

## Operating characteristics
*   **Time Complexity:** The collective read (`MPI_FILE_READ_ALL`) scales with the number of input processors $m$. The overhead of setting up the derived data type and file view is per-processor but negligible compared to I/O time for large datasets.
*   **Memory Complexity:** Memory usage is proportional to the size of the cell data subset held by each input processor. The merging of octree data reduces redundancy.
*   **Measured Performance:** Tested on LeMieux, an HP/Compaq AlphaServer with 3,000 processors, visualizing 100 million hexahedral elements (400 MB per time step). The method was evaluated with 64 rendering processors and a 512x512 image size [S1].
*   **Strengths:** Efficient for large datasets where collective I/O is supported and beneficial. Avoids the need for complex merging logic at the rendering stage by handling data distribution during the read phase.
*   **Limitations:** Relies on MPI-IO support. May suffer from high overhead if the number of input processors is very large or if the file system does not optimize collective operations well.
*   **When to Use:** When the dataset is large, the file system supports efficient collective I/O, and minimizing post-read data merging is desired.
*   **When Not to Use:** When the overhead of collective I/O is too high, in which case the Independent Contiguous Read variant is preferred [S1].

## Background
This method was developed as part of a parallel volume rendering system for large time-varying volume data. It was presented in the context of visualizing ground motion simulation data. The work was conducted by researchers associated with the Pittsburgh Supercomputing System [S1].

## Sources
[S1] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
