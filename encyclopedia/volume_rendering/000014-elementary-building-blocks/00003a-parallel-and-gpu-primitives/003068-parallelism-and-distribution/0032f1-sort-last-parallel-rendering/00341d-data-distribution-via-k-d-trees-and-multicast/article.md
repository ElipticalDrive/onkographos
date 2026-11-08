# Data Distribution via k-d Trees and Multicast

Also called: Top-down data distribution layout, k-d Tree Based Data Distribution

## Overview
This method is a data distribution strategy for sort-last parallel volume rendering that minimizes network redundancy by assigning volume data based on k-d tree leaf nodes, ensuring each worker node is responsible for exactly one leaf [S1, unofficial]. It differs from its siblings by using a top-down distribution from a master node that holds comprehensive knowledge of the dataset's distribution, rather than relying on static block-splitting or distributed pipeline stages [S1, unofficial]. Unlike [Block-Splitting (BS) method](9229-block-splitting-bs-method.md) which may imply static spatial partitioning without visibility sorting, this approach integrates k-d trees as a visibility sorting structure to guide data assignment [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A volume dataset, a set of worker nodes, a master node, and optionally a parallel file system.
*   **Outputs:** Distributed volume data on worker nodes, ready for local rendering and subsequent compositing.

**Data Structures:**
*   **k-d Tree:** A binary tree structure used for visibility sorting. Each leaf node corresponds to a specific region of the volume data.
*   **Axis-Aligned Bounding Box (AABB):** Associated with each k-d tree leaf node, defining the spatial extent of the data assigned to a worker.
*   **Multicast Group:** A network group identifier used for UDP-based data transmission.

**Steps:**
1.  **Tree Construction and Assignment:** A k-d tree is constructed for the volume dataset. Each worker node is assigned responsibility for exactly one leaf node of this tree [S1, unofficial].
2.  **AABB Distribution:** The master node distributes the AABB of each leaf node to the corresponding worker node [S1, unofficial].
3.  **Dataset Location Communication:** The master node communicates the location of the volume dataset on the parallel file system to the worker nodes [S1, unofficial].
4.  **Data Loading (Option A - Parallel File System):** Worker nodes load their own copy of the volume dataset directly from the parallel file system. This imposes network load on the connection between worker nodes and the file server but allows the master node to be used solely for alpha compositing without storing the dataset permanently [S1, unofficial].
5.  **Data Loading (Option B - Multicast):**
    *   The master node sends the whole dataset using IP Multicast [S1, unofficial].
    *   Worker nodes register to the same Multicast group [S1, unofficial].
    *   Each worker node retrieves UDP packages from the Multicast stream [S1, unofficial].
    *   A worker node stores a package permanently only if the data contained in the package is within the AABB of its assigned leaf node [S1, unofficial].
    *   Each worker node allocates enough main memory to accommodate its share of the volume dataset [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Master Node Logic
def master_distribute(dataset, worker_nodes, k_d_tree):
    # Step 1: Assign each worker to a leaf node
    # k_d_tree.leaves is a list of leaf nodes
    # worker_nodes is a list of worker node identifiers
    assert len(k_d_tree.leaves) == len(worker_nodes)
    
    assignments = {}
    for i, leaf in enumerate(k_d_tree.leaves):
        worker = worker_nodes[i]
        assignments[worker] = leaf
        
        # Step 2: Distribute AABB
        aabb = leaf.get_aabb()
        send_aabb_to_worker(worker, aabb)
        
        # Step 3: Communicate dataset location (if using Option A)
        # This is only needed if workers load from parallel file system
        # send_filesystem_location_to_worker(worker, dataset_path)

    # Step 5: Multicast Distribution (Option B)
    # Note: In a real implementation, this would be a streaming operation
    multicast_group = "239.1.1.1"
    multicast_port = 5000
    
    # Start multicast stream
    start_multicast_stream(multicast_group, multicast_port)
    
    # Send dataset chunks
    for chunk in dataset.get_chunks():
        send_multicast_packet(multicast_group, multicast_port, chunk)
        
    # Handle reliability fallback if needed
    # This is handled by worker nodes checking for missing data

# Worker Node Logic
def worker_receive(worker_id, assigned_leaf, dataset_path):
    aabb = assigned_leaf.get_aabb()
    memory_buffer = allocate_memory_for_share(aabb)
    
    # Option A: Load from Parallel File System
    # load_data_from_filesystem(dataset_path, aabb, memory_buffer)
    
    # Option B: Receive via Multicast
    multicast_group = "239.1.1.1"
    multicast_port = 5000
    
    # Register for multicast group
    join_multicast_group(multicast_group)
    
    while True:
        packet = receive_multicast_packet(multicast_group, multicast_port)
        
        # Check if packet data is within the worker's AABB
        if is_within_aabb(packet.data, aabb):
            store_permanently(packet.data, memory_buffer)
        else:
            discard(packet.data)
            
        # Check for completion or reliability signals
        if check_completion_signal():
            break
            
    # Fallback handling for reliability
    if check_missing_data():
        # Fall back to TCP if Multicast failed
        request_data_via_tcp(worker_id, assigned_leaf)
```

**Parameters:**
*   **Multicast Group Address:** Not given in the sources.
*   **Multicast Port:** Not given in the sources.
*   **k-d Tree Splitting Criteria:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Unreliability of UDP:** IP Multicast uses UDP, which is unreliable. The system must ensure datagram delivery by communicating additional meta information. If Multicast fails, the system falls back to sending the dataset to individual worker nodes using TCP [S1, unofficial]. However, because worker nodes are usually in the same local network as the master node, this fallback happens infrequently [S1, unofficial].
*   **Master Node Knowledge:** The master node must maintain comprehensive knowledge of how the dataset is distributed among worker nodes [S1, unofficial].
*   **Memory Allocation:** Worker nodes must allocate enough main memory to accommodate their share of the volume dataset [S1, unofficial].

## Subtopics

### KD-Tree Based Data Partitioning with Clipping
A data partitioning strategy for distributed memory parallel rendering that ensures proper view ordering for all processes by creating a KD-tree and clipping straddling cells to partition boundaries [S2]. Unlike regular grids where KD-tree planes do not intersect cells, unstructured grids require cells to be copied and clipped because KD-tree planes inevitably intersect general unstructured cells [S2]. The process involves creating a KD-tree that partitions cells evenly, identifying cells that straddle multiple KD-tree regions, copying straddling cells to all regions they intersect, and clipping the copied cells by the bounds of each region to ensure every region completely contains all cells within it [S2]. Hardware clipping planes are avoided because they do not support algorithms like projected tetrahedra well, causing artifacts along cutting planes in transparent regions [S2]. This method ensures load balancing as data is split evenly amongst processes cell-wise [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Worker nodes must allocate enough main memory to accommodate their share of the volume dataset [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Reduces network redundancy by assigning volume data based on k-d tree leaf nodes [S1, unofficial]. Allows the master node to be used for alpha compositing only if workers load from a parallel file system [S1, unofficial]. Ensures reasonable load balancing in the KD-tree clipping variant [S2].
*   **Limitations:** Requires the master node to maintain comprehensive knowledge of the dataset distribution [S1, unofficial]. IP Multicast is unreliable, requiring fallback mechanisms [S1, unofficial].
*   **When to Use:** Distributed volume rendering on HPC clusters where reducing network redundancy is critical [S1, unofficial].
*   **When Not to Use:** When the master node cannot maintain comprehensive knowledge of the distribution [S1, unofficial].

## Background
This method was implemented in a described distributed volume rendering architecture to manage large datasets across HPC clusters [S1, unofficial]. It is part of a software architecture that supports various usage scenarios, ranging from remote data exploration using thin clients to virtual reality applications [S1, unofficial]. The k-d tree based partitioning with clipping variant was developed for parallel unstructured volume rendering in ParaView [S2].

## Sources
[S1] A SOFTWARE ARCHITECTURE FOR DISTRIBUTED VOLUME RENDERING ON ... - Actapress, https://actapress.com/Abstract.aspx?paperId=454770, fetched 2026-10-27 (unofficial)
[S2] PDF Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
