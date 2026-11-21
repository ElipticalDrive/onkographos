# Distributed Memory Caching System with Load Balancing

Also called: Block Caching Challenge Array, Logical Global Address Space with Caching

## Overview
This method manages the storage, communication, and caching of volume blocks across $N$ distributed nodes to minimize communication overhead during ray casting volume rendering [S1]. It differs from automatic caching systems by providing absolute control over block assignment, ensuring blocks are assigned only to nodes that already hold a copy (when possible) to eliminate tile transfer costs [S1]. Unlike methods using iterated binary swaps for composition, it assigns fixed screen regions to nodes and transfers overlapping tiles once to the responsible node [S1].

## How it works
**Inputs and Outputs**
*   **Inputs**: A volume dataset partitioned into $B$ blocks; a distributed system with $N$ nodes; a replication factor $R$; the current viewpoint; and historical rendering times for blocks [S1].
*   **Outputs**: The final composited image displayed at a central location [S1].

**Data Structures**
*   **Master Process State**: Maintains a global map of which blocks are currently stored in the cache slots of each of the $N$ nodes. It also maintains a list of blocks sorted by estimated rendering cost [S1].
*   **Node Cache**: Each node maintains $v$ cache slots, where $v = (R \times B) / N$. The total collective memory holds $R \times B$ slots [S1].
*   **Block List**: A list of all $B$ blocks, sorted in decreasing order of estimated computational cost [S1].

**Parameters**
*   $B$: Total number of blocks in the dataset [S1].
*   $N$: Total number of nodes in the distributed system [S1].
*   $R$: Replication factor. A replication factor of $R$ requires each node to maintain $v$ cache slots, for a total of $R \times B$ slots collectively [S1].
*   Load Balance Threshold ($L_{thresh}$): The maximum allowed ratio between the maximum and average rendering times of any node. Typical values tested: 0.65, 0.80, 0.90, 0.95 [S1].
*   Block Size: Optimal L2 cache performance on the RN00 hardware was observed between 2 MB and 4 MB [S1].
*   Tile Size: Average tile size for typical views was 150 KB (100 pixels, 4 floating point values per pixel) [S1].

**Algorithm Steps**

1.  **Initialization**: The Master Process initializes the cache map for all $N$ nodes as empty. The Master sends the initial viewpoint to all nodes [S1].
2.  **Initial Assignment (Frame 1)**: Assign the $B$ blocks to the $N$ nodes for projection in a round-robin fashion. This avoids concentrating blocks assigned to any node in any specific region of the dataset, which would cause load imbalance due to correlated costs of adjacent blocks [S1]. Each node loads its assigned blocks from disk into its local cache. If $R > 1$, the Master instructs nodes to fill their remaining cache slots with additional blocks (fairly assigned) that they are *not* assigned to project for the first frame. This pre-fills the cache with blocks likely to be needed for composition or future frames [S1].
3.  **Projection**: Each node projects the blocks currently held in its cache. For each block, the node generates a tile (a 2D subimage) corresponding to the block's contribution to the final image [S1].
4.  **Tile Transfer**: The screen is partitioned into fixed regions, one assigned to each node for the composition step. Each node identifies which tiles it generated overlap with the screen regions assigned to *other* nodes. Nodes send these overlapping tiles to the respective nodes responsible for those screen regions. Note: If a node holds a block that it needs for composition (because it was assigned to that node for projection or pre-cached), no tile transfer is needed for that block [S1].
5.  **Composition**: Each node composites the tiles received from other nodes with its own locally generated tiles for its assigned screen region. Composition is performed in depth order relative to the view direction [S1].
6.  **Result Submission**: Each node sends its final composited partial image to a central location for display [S1].
7.  **Cost Estimation and Load Balancing (End of Frame)**: The Master receives the actual rendering times for each block rendered in the current frame. These times are used as estimates for the computational cost of rendering each block in the next frame. The Master sorts all $B$ blocks into a list in decreasing order of estimated cost. The Master calculates the **Load Threshold**: The maximum amount of estimated work that can be assigned to any node. Load balance is measured as the ratio between the maximum and average times for a node to render its assigned blocks. The algorithm ensures this ratio does not exceed the Load Balance Threshold ($L_{thresh}$). The Master reassigns blocks to nodes for the next frame using a greedy algorithm based on the sorted list. For each block (starting with the most expensive):
    1.  **Priority 1 (Load Balance)**: Assign the block to a node such that the estimated load balance remains above the threshold. No further blocks are assigned to a node that has already passed this threshold (unless all nodes have) [S1].
    2.  **Priority 2 (Block Communication)**: If possible, avoid transferring the block itself. Assign the block to a node that already holds a copy in its cache [S1].
    3.  **Priority 3 (Tile Communication)**: If possible, assign the block to the same node that will require its tile for composition (i.e., the node responsible for the screen region overlapping the block). This avoids transferring the tile [S1].
    4.  **Fallback Assignment**: If a block must be transferred (neither Priority 2 nor 3 can be satisfied), find another node that holds a copy of the block. If multiple nodes hold a copy, use a fair algorithm to choose among them so that the average number of blocks served by each node remains equal [S1].
    The Master updates the cache map and sends instructions to nodes to load/unload blocks as necessary [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distributed_caching_rendering(B, N, R, L_thresh, blocks, viewpoints):
    """
    B: Total number of blocks
    N: Total number of nodes
    R: Replication factor
    L_thresh: Load balance threshold (max ratio of max/avg node time)
    blocks: List of block objects, each with id, size, and estimated_cost
    viewpoints: List of viewpoints for each frame
    """
    # Initialize cache state: node_id -> set of block_ids
    node_caches = {node_id: set() for node_id in range(N)}
    
    # Initial assignment for Frame 0
    # Round-robin assignment to avoid spatial clustering
    for i, block in enumerate(blocks):
        node_id = i % N
        node_caches[node_id].add(block.id)
        
    # Fill caches if R > 1
    if R > 1:
        slots_per_node = (R * B) // N
        for node_id in range(N):
            needed = slots_per_node - len(node_caches[node_id])
            if needed > 0:
                # Fairly assign additional blocks not currently in cache
                count = 0
                for block in blocks:
                    if count >= needed:
                        break
                    if block.id not in node_caches[node_id]:
                        node_caches[node_id].add(block.id)
                        count += 1

    for frame_idx, viewpoint in enumerate(viewpoints):
        # 1. Send viewpoint to all nodes
        for node_id in range(N):
            send(viewpoint, node_id)
            
        # 2. Projection Phase
        tiles_generated = {} # node_id -> list of (block_id, tile_data)
        render_times = {} # block_id -> time_taken
        
        for node_id in range(N):
            tiles = []
            times = {}
            for block_id in node_caches[node_id]:
                tile_data = generate_tile(block_id) # Standard ray casting step
                tiles.append((block_id, tile_data))
                times[block_id] = get_elapsed_time()
            tiles_generated[node_id] = tiles
            render_times.update(times)
            
        # 3. Tile Transfer Phase
        # Screen is partitioned into N regions, region_id -> node_id
        # Assume blocks map to regions deterministically: region = block.id % N
        tile_transfers = {} # (source_node, dest_node) -> list of tiles
        for source_node, tiles in tiles_generated.items():
            for block_id, tile in tiles:
                dest_node = block_id % N # Fixed screen partition
                if dest_node != source_node:
                    if (source_node, dest_node) not in tile_transfers:
                        tile_transfers[(source_node, dest_node)] = []
                    tile_transfers[(source_node, dest_node)].append(tile)
                    
        # Send tiles
        for (src, dst), tiles in tile_transfers.items():
            for tile in tiles:
                send(tile, dst)
                
        # 4. Composition Phase
        final_images = {}
        for node_id in range(N):
            own_tiles = tiles_generated[node_id]
            received_tiles = receive_from_network(node_id)
            final_images[node_id] = alpha_composite(own_tiles + received_tiles)
            
        # 5. Result Submission
        send(final_images, master_node)
        
        # 6. Load Balancing for Next Frame
        if frame_idx < len(viewpoints) - 1:
            # Update estimated costs
            for block_id, time in render_times.items():
                blocks[block_id].estimated_cost = time
                
            # Sort blocks by cost descending
            sorted_blocks = sorted(blocks, key=lambda b: b.estimated_cost, reverse=True)
            
            # Calculate average load for threshold check
            total_cost = sum(b.estimated_cost for b in sorted_blocks)
            avg_load = total_cost / N
            max_allowed_load = L_thresh * avg_load
            
            # Reassign blocks
            new_assignments = {node_id: set() for node_id in range(N)}
            node_load = {node_id: 0.0 for node_id in range(N)}
            
            for block in sorted_blocks:
                best_node = None
                best_score = -1
                eligible_nodes = []
                
                # Priority 1: Check load balance threshold
                for node_id in range(N):
                    if node_load[node_id] + block.estimated_cost <= max_allowed_load:
                        eligible_nodes.append(node_id)
                        
                if not eligible_nodes:
                    # All nodes passed threshold, assign to least loaded
                    eligible_nodes = [min(node_load, key=node_load.get)]
                    
                # Priority 2 & 3: Among eligible nodes, choose best based on cache/tile locality
                for node_id in eligible_nodes:
                    score = 0
                    # Priority 2: Avoid block transfer (node already has block)
                    if block.id in node_caches[node_id]:
                        score += 1000
                    # Priority 3: Avoid tile transfer (node needs tile for composition)
                    if node_id == block.id % N:
                        score += 100
                        
                    if score > best_score:
                        best_score = score
                        best_node = node_id
                        
                # Fallback: If best_node doesn't have the block, find eligible node that does
                if block.id not in node_caches[best_node]:
                    candidates = [nid for nid in eligible_nodes if block.id in node_caches[nid]]
                    if candidates:
                        # Fair selection: pick node with fewest currently assigned blocks
                        best_node = min(candidates, key=lambda nid: len(node_caches[nid]))
                        
                # Assign block
                new_assignments[best_node].add(block.id)
                node_load[best_node] += block.estimated_cost
                
            # Update caches
            for node_id in range(N):
                node_caches[node_id] = new_assignments[node_id]

# Standard operations assumed available or implemented as stubs
def send(data, target): pass
def receive_from_network(node_id): return []
def generate_tile(block_id): return b""
def get_elapsed_time(): return 0.0
def alpha_composite(tiles): return b""
```

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Each node stores $v = (R \times B) / N$ blocks. Total memory is $R \times B$ blocks [S1].
*   **Performance**: Experiments on the RN00 showed that replication factor $R$ significantly impacts performance. Higher $R$ reduces tile and block transfers but increases memory usage. Typical tile size was 150 KB, while blocks were 2-4 MB [S1].
*   **Strengths**: Absolute control over caching allows minimizing communication by assigning blocks to nodes that already hold them or need their tiles. It exploits data coherence effectively [S1].
*   **Limitations**: The central master process is a serial bottleneck as $N$ increases. The method assumes blocks are of similar size and that L2 cache effects are significant [S1].
*   **When to use**: Suitable for distributed systems with moderate $N$ where communication overhead is high and data coherence is strong. Not suitable for very large $N$ due to the master bottleneck [S1].

## Background
This method was described in the paper "Exploiting deep parallel memory hierarchies for ray casting volume rendering" [S1]. It was designed to handle storage, communication, and caching of volume blocks across distributed nodes. The work highlights the trade-offs between block replication, load balancing, and communication overhead in distributed volume rendering [S1].

## Sources
[S1] Exploiting deep parallel memory hierarchies for ray casting volume rendering, https://doi.org/10.1145/266638.266645, fetched 2026-10-26
