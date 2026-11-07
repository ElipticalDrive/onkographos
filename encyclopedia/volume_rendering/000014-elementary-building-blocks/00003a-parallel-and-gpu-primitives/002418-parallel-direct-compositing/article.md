# Parallel Direct Compositing

Also called: Direct compositing method

## Overview
Parallel Direct Compositing is a distributed memory volume rendering technique where image compositing is performed directly across processors following the subdivision of the data domain [S1, unofficial]. Unlike tiled approaches that require complex synchronization or reduction trees, this method relies on a host processor to collect and combine subimages, accepting a serial bottleneck in display rates in exchange for simplified communication logic and decreasing relative communication costs as processor count increases [S1, unofficial].

## How it works
**Inputs:**
- A 3D volume dataset [S1, unofficial].
- A distributed memory environment with $p$ processors [S1, unofficial].
- A bidirectional torus network topology [S1, unofficial].

**Outputs:**
- A final 2D composite image of resolution $n$ pixels [S1, unofficial].

**Data Structures:**
- **Subimages:** Each processor $i$ maintains a local buffer containing the rendered portion of the final image corresponding to its assigned data block [S1, unofficial]. The layout is a standard 2D pixel array [S1, unofficial].
- **Global State:** Not explicitly maintained during compositing; the host processor manages the aggregation logic [S1, unofficial].

**Steps:**
1. **Domain Subdivision:** The 3D data domain is subdivided into blocks [S1, unofficial]. Each block is assigned to a specific processor in the distributed system [S1, unofficial].
2. **Local Rendering:** Each processor independently renders its assigned data block into a local subimage [S1, unofficial]. This phase requires no inter-processor communication [S1, unofficial].
3. **Subimage Transmission:** Processors transmit their local subimages to a designated host processor [S1, unofficial]. The transmission occurs over the network (e.g., bidirectional torus) [S1, unofficial].
4. **Aggregation:** The host processor invokes a library function to collect all incoming subimages and combine them into the final image [S1, unofficial].
5. **Display:** The final image is displayed on a remote workstation or stored [S1, unofficial].

**Formulas:**
The communication cost $C_{comm}$ for the compositing phase on a bidirectional torus is modeled as [S1, unofficial]:

$$ C_{comm} = C_1 \cdot p \cdot n \cdot t_{trans} + C_2 \cdot t_{overhead} $$

Where:
- $p$: Number of processors [S1, unofficial].
- $n$: Number of pixels in the image [S1, unofficial].
- $t_{trans}$: Per-pixel transfer cost [S1, unofficial].
- $t_{overhead}$: Per-message overhead [S1, unofficial].
- $C_1, C_2$: Positive constants smaller than 1, representing efficiency factors of the network and protocol [S1, unofficial].

**Parameters:**
- Typical values for $C_1$ and $C_2$ are not specified, only that they are positive constants $< 1$ [S1, unofficial].
- Network topology: Bidirectional torus [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Serial Bottleneck:** The display rate is limited by the serial data stream resulting from the host processor collecting subimages [S1, unofficial].
- **Network Latency:** Performance is further degraded by network latency, particularly as the number of processors increases if the per-message overhead $t_{overhead}$ dominates [S1, unofficial].
- **Mitigation:** Image compression techniques can be applied to alleviate bandwidth and latency issues, though direct transfer is often used for large-scale scientific simulations where frame rates are not critical [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_direct_compositing(local_volume, image_w, image_h, rank, total_ranks, is_host):
    # Step 1 & 2: Render local data block to a local subimage (RGBA pixels)
    subimage = [[ [0.0, 0.0, 0.0, 0.0] for _ in range(image_w) ] for _ in range(image_h)]
    for y in range(image_h):
        for x in range(image_w):
            # [G] Standard volume rendering: cast ray, accumulate color/alpha
            subimage[y][x] = render_pixel(local_volume, x, y)

    if is_host:
        # Step 3 & 4: Host collects subimages and combines them
        final_image = [[ [0.0, 0.0, 0.0, 0.0] for _ in range(image_w) ] for _ in range(image_h)]
        # Host inserts its own subimage first
        final_image = assemble_tile(final_image, subimage, rank, total_ranks, image_w, image_h)
        # Receive and assemble subimages from all worker ranks
        for r in range(total_ranks):
            if r == rank:
                continue
            worker_subimage = receive_subimage(r, image_w, image_h)
            final_image = assemble_tile(final_image, worker_subimage, r, total_ranks, image_w, image_h)
        return final_image
    else:
        # Step 3: Worker sends its subimage to the host
        send_subimage(0, subimage)
        return None

def render_pixel(volume, x, y):
    # [G] Standard front-to-back or back-to-front compositing along a ray
    # Returns an RGBA tuple [r, g, b, a]
    return [0.0, 0.0, 0.0, 0.0]

def assemble_tile(target, source, src_rank, total_ranks, w, h):
    # [G] Maps a worker's subimage tile to its correct spatial position in the final image
    # Assumes a simple row-major tile decomposition of the image
    tiles_per_row = max(1, total_ranks // (h // max(1, w // total_ranks)))
    if tiles_per_row == 0: tiles_per_row = 1
    tile_h = h // total_ranks
    tile_w = w // tiles_per_row
    ty = (src_rank // tiles_per_row) * tile_h
    tx = (src_rank % tiles_per_row) * tile_w
    for dy in range(tile_h):
        for dx in range(tile_w):
            sy = ty + dy
            sx = tx + dx
            if sy < h and sx < w:
                target[sy][sx] = source[dy][dx]
    return target

def receive_subimage(sender_rank, w, h):
    # [G] Blocks until a message of size w*h*4 floats arrives from sender_rank
    # Returns a 2D list of RGBA pixels
    return [[ [0.0, 0.0, 0.0, 0.0] for _ in range(w) ] for _ in range(h)]

def send_subimage(receiver_rank, data):
    # [G] Sends the 2D list of RGBA pixels to receiver_rank
    pass
```

## Operating characteristics
**Time and Memory Complexity:**
- The rendering phase is scalable with no communication required [S1, unofficial].
- The compositing communication cost is given by $C_1 \cdot p \cdot n \cdot t_{trans} + C_2 \cdot t_{overhead}$ [S1, unofficial].
- Analysis shows that communication cost decreases as the number of processors $p$ increases, making the method acceptable for large $p$ [S1, unofficial].

**Measured Performance:**
Tests on an Intel Paragon XP/S (NASA Langley Research Center) with a $256 \times 256$ image and $60^3$ data points [S1, unofficial]:

| Nodes | Raw Compositing (s) | Communication Compositing (s) |
| :--- | :--- | :--- |
| 125 | 0.0455 | 0.0612 |
| 216 | 0.0310 | 0.0583 |

Scaling analysis for simulation and resampling on the same hardware (8 to 216 nodes) [S1, unofficial]:
- Simulation compute time: 10.665s (8 nodes) $\rightarrow$ 0.4109s (216 nodes) [S1, unofficial].
- Resampling time: 13.01s (8 nodes) $\rightarrow$ 0.681s (216 nodes) [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Simple communication pattern; communication overhead decreases relative to compute as processors increase; no synchronization barriers during rendering [S1, unofficial].
- **Limitations:** Display rates are limited by the serial collection of subimages on the host; performance degrades with high network latency [S1, unofficial].

**When to Use:**
- Suitable for large-scale scientific simulations on massively parallel computers where interactive frame rates are not the primary constraint, and the simulation does not run multiple time steps per second [S1, unofficial].
- Best used when the network topology supports efficient bulk data transfer (e.g., torus) and a dedicated host processor is available for aggregation [S1, unofficial].

## Background
Parallel Direct Compositing was developed and analyzed in the context of runtime volume visualization for parallel Computational Fluid Dynamics (CFD) [S1, unofficial]. The library was implemented in C/C++ using Intel's native communication library NX on a 72-node Intel Paragon XP/S operated at the NASA Langley Research Center [S1, unofficial]. The analysis was consistent with Neumann's results regarding communication costs [S1, unofficial]. The approach was designed for distributed memory message-passing environments, with suggestions for future portability using MPI or PVM [S1, unofficial].

## Sources
[S1] DTIC ADA302573: Runtime Volume Visualization for Parallel CFD., https://archive.org/details/DTIC_ADA302573, fetched 2026-10-26 (unofficial)
