# 2-3 Swap Compositing

Also called: 2-3 Swap

## Overview
2-3 Swap Compositing is a parallel image compositing technique designed to combine independently rendered image tiles from multiple processors in a distributed-memory environment. It solves the limitation of standard Parallel Binary-Swap Compositing, which requires the number of processors $N$ to be a power of two, by allowing any integer number of processors to participate [S1]. Unlike its parent method, it does not enforce a strict binary tree structure, instead grouping processors into pairs or triplets to facilitate data exchange [S1].

## How it works
**Inputs and Outputs:**
- Input: $N$ processors, each holding a partial image tile (or volume slice contribution) that needs to be composited into a final image.
- Output: A single composited image distributed across the processors (or gathered to a root, depending on implementation, though the text implies distributed exchange).

**Data Structures:**
- Processor ID: An integer identifier for each processor, typically $0$ to $N-1$.
- Image Tile: The local buffer containing the partial compositing result for the processor's assigned region.

**Algorithm Steps:**
1. **Initialization**: Each processor $p$ holds its local image tile $I_p$.
2. **Grouping**: The $N$ processors are logically divided into groups of size 2 or 3. The specific grouping strategy must ensure all $N$ processors are included. Since $N$ is not necessarily a power of two, the grouping is not a perfect binary tree.
3. **Communication Rounds**: The algorithm proceeds in multiple rounds. In each round, processors within their assigned groups (size 2 or 3) exchange data.
4. **Data Exchange**: Within each group, processors use a **direct send** approach to exchange their current image tiles. This involves sending the local tile to the other processor(s) in the group and receiving the tile(s) from them.
5. **Compositing**: Upon receiving tiles from peers, each processor composites the received data with its own local tile. The compositing operation is typically front-to-back alpha compositing:
   $$ C_{out} = C_{in} + (1 - \alpha_{in}) C_{peer} $$
   $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \alpha_{peer} $$
   where $C$ is color and $\alpha$ is opacity.
6. **Iteration**: Steps 3-5 are repeated for subsequent rounds until all processors have exchanged data with all other processors, resulting in the fully composited image.

**Parameters:**
- $N$: Total number of processors. Must be an integer $\ge 2$.
- Group Size: 2 or 3. The sequence of group sizes across rounds must effectively cover all necessary exchanges to ensure global compositing.

**Edge Cases and Pitfalls:**
- **Non-Power-of-Two $N$**: The primary advantage is handling arbitrary $N$. However, the grouping logic must be carefully designed to ensure connectivity and termination.
- **Synchronization**: Unlike binary swap, which has synchronization after each round, 2-3 swap also requires synchronization after each round to ensure all sends/receives complete before the next compositing step [S1].
- **Network Congestion**: While binary swap reduces congestion, 2-3 swap uses direct send within small groups, which may have different congestion characteristics depending on the network topology.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_three_swap_compositing(N, my_id, my_tile):
    """
    Performs 2-3 swap compositing for N processors.
    my_id: Integer ID of this processor (0 to N-1).
    my_tile: The initial image tile held by this processor.
    """
    # Note: The exact grouping strategy for arbitrary N is not detailed in the source.
    # The source states processors are divided into groups of twos or threes.
    # A common approach for arbitrary N in such algorithms involves a specific
    # permutation or round-robin assignment. Without explicit details, we assume
    # a generic round-based exchange where peers are determined by a schedule.
    
    # Placeholder for the specific grouping schedule logic which is not provided
    # in the source text. The source mentions "multiple rounds" and "groups of twos or threes".
    # Implementers would need to derive or look up the specific schedule from Yu et al. [18].
    
    current_tile = my_tile
    
    # Determine the number of rounds and peer assignments based on N
    # This part is implementation-specific and not fully defined in the source.
    # We assume a loop structure where peers are identified.
    
    for round in range(num_rounds):
        # Identify peers for this round based on the 2-3 grouping strategy
        peers = get_peers_for_round(my_id, N, round)
        
        # Send current tile to peers and receive their tiles
        sent_tiles = {}
        received_tiles = {}
        
        for peer in peers:
            # Send my current tile to peer
            send(peer, current_tile)
            # Receive tile from peer
            received_tiles[peer] = receive(peer)
            
        # Composite received tiles with current tile
        for peer, peer_tile in received_tiles.items():
            current_tile = composite(current_tile, peer_tile)
            
    return current_tile

def composite(tile1, tile2):
    """
    Performs front-to-back alpha compositing of two tiles.
    """
    # Assuming tile is a structure with color and alpha channels
    # This is a simplified representation. Actual implementation depends on data layout.
    result_color = tile1.color + (1 - tile1.alpha) * tile2.color
    result_alpha = tile1.alpha + (1 - tile1.alpha) * tile2.alpha
    return Tile(result_color, result_alpha)
```

## Subtopics

### [Radix-k Compositing](12235-radix-k-compositing.md)
Radix-k compositing is a parallel image compositing strategy designed for distributed-memory systems that combines the flexibility of direct send with the performance characteristics of binary-swap compositing. It solves the limitation of binary-swap methods, which require the number of processors to be a power of two, by allowing any number of processors $N$ to participate in the compositing phase. Unlike its parent method, 2-3 Swap Compositing, which restricts group sizes to 2 or 3, Radix-k allows variable group sizes per round, provided their product equals $N$.

## Operating characteristics
- **Time Complexity**: The number of rounds depends on the grouping strategy for $N$. For binary swap, it is $\log_2 N$. For 2-3 swap, it is generally close to $\log_2 N$ but varies based on the factorization of $N$ into 2s and 3s.
- **Memory Complexity**: $O(1)$ additional memory per processor for storing the image tile and temporary buffers for communication.
- **Strengths**: Handles arbitrary numbers of processors $N$ without requiring padding to the next power of two. Reduces synchronization overhead compared to all-to-all approaches.
- **Limitations**: The grouping logic is more complex than binary swap. Performance may vary depending on the specific factorization of $N$.
- **When to use**: When the number of processors is not a power of two and a flexible, scalable compositing method is required.

## Background
2-3 Swap Compositing was implemented by Yu et al. [S1] as a modification of binary swap to overcome the limitation that the number of processors had to be a power of two. It is part of the broader field of hybrid distributed-memory parallel volume rendering techniques.

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
