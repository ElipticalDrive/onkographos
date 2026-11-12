# Lookup Table Addressing for Resampling

Also called: Brick Resampling Lookup Table, Bricking with Lookup Table Addressing, Lookup Table Addressing

## Overview
Lookup Table Addressing for Resampling is a method to compute memory offsets for accessing neighboring samples in a brick-structured volume without using conditional branches or expensive division/modulo operations [S1, S2, unofficial]. It solves the problem of efficiently addressing the 7-neighborhood (for resampling) or 26-neighborhood (for gradient computation) required by volume raycasting, where samples may span across brick boundaries [S1, S2, unofficial]. Unlike the parent method [Brick Volume Addressing with If-Else](3340-brick-volume-addressing-with-if-else.md), which uses `if-else` statements to check boundary conditions, this method uses a precomputed global lookup table indexed by a bitwise-derived case identifier, eliminating branches and ensuring cache-friendly access [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Current sample integer coordinates $(i, j, k)$ within the volume.
- Brick dimensions $B_x, B_y, B_z$ (must be powers of two).
- Logarithms of brick dimensions $N_x, N_y, N_z$ such that $B_x = 2^{N_x}$, $B_y = 2^{N_y}$, $B_z = 2^{N_z}$ [S2, unofficial].

**Outputs:**
- An array of memory offsets for the required neighbors (7 offsets for resampling, 27 offsets for gradient).

### Data Structures
**Lookup Table (LUT):**
- **Layout:** A flat array of 56 integers (for resampling) or 216 integers (for gradient, derived from $3^3 \times 27$ cases? No, the source specifies 56 offsets for resampling. For gradient, the source describes mapping to an index in $[0, 26]$, implying a table of 27 offsets, but the text explicitly states "The lookup table contains $8 \cdot 7 = 56$ offsets" for resampling. For gradient, it says "distinguish 27 cases... map to an index in the range of $[0, 26]$". It does not explicitly state the table size for gradient, but logically it would be 27 offsets. However, the source [S1, unofficial] focuses on the 56-offset table. We will describe the resampling table as the primary artifact, noting the gradient indexing formula).
- **Size:** 56 entries $\times$ 4 bytes/entry = 224 bytes [S1, S2, unofficial].
- **Content:** Each entry is a 32-bit signed integer representing the byte offset from the current sample's base address to a specific neighbor. The offsets are stored in a fixed order corresponding to the 7 neighbors: $(+1,0,0), (0,+1,0), (+1,+1,0), (0,0,+1), (+1,0,+1), (0,+1,+1), (+1,+1,+1)$ [S1, unofficial].
- **Storage:** Global, shared across all bricks [S2, unofficial].

### Numbered Steps (Resampling - 7-neighborhood)
1.  **Extract Brick Offset:** Compute the position of the sample within the current brick using bitwise AND with the brick dimensions minus one:
    $$ i_{local} = i \ \& \ (B_x - 1) $$
    $$ j_{local} = j \ \& \ (B_y - 1) $$
    $$ k_{local} = k \ \& \ (B_z - 1) $$
    [S1, S2, unofficial]

2.  **Increment:** Add 1 to each local coordinate to shift the range from $[0, B-1]$ to $[1, B]$:
    $$ i' = i_{local} + 1 $$
    $$ j' = j_{local} + 1 $$
    $$ k' = k_{local} + 1 $$
    [S1, S2, unofficial]

3.  **Mask to Boundary:** Perform a bitwise AND with the bitwise NOT of the brick dimension minus one. This maps values strictly inside the brick ($1 \dots B-1$) to 0, and values on the boundary ($B$) to $B$:
    $$ i'' = i' \ \& \ \sim(B_x - 1) $$
    $$ j'' = j' \ \& \ \sim(B_y - 1) $$
    $$ k'' = k' \ \& \ \sim(B_z - 1) $$
    [S1, S2, unofficial]
    *Note:* $\sim(B-1)$ is equivalent to `~(B-1)` in two's complement arithmetic. If $B=32$ ($2^5$), $B-1=31$ ($0x1F$), $\sim(B-1)$ has bits 5 and above set.

4.  **Sum and Shift:** Sum the masked values and shift right by the minimum of the logarithmic dimensions to produce an index in $[0, 7]$:
    $$ Index = (i'' + j'' + k'') \gg \min(N_x, N_y, N_z) $$
    [S1, S2, unofficial]
    *Constraint:* This formula assumes brick dimensions are powers of two and "a power of two apart" (i.e., $B_x, B_y, B_z$ are distinct powers of two or at least the shift amount correctly separates the contributions). If dimensions are not powers of two apart, a virtual shift or adjustment is needed [S1, unofficial].

5.  **Lookup Offsets:** Use the $Index$ to retrieve the 7 precomputed offsets from the global LUT.
    $$ Offset_m = LUT[Index][m] \quad \text{for } m \in \{0, \dots, 6\} $$
    [S1, unofficial]

6.  **Compute Addresses:** Add each offset to the base address of the current sample $(i, j, k)$ to get the addresses of the 7 neighbors.
    [S1, unofficial]

### Numbered Steps (Gradient - 26-neighborhood)
1.  **Extract Brick Offset:**
    $$ i_{local} = i \ \& \ (B_x - 1) $$
    $$ j_{local} = j \ \& \ (B_y - 1) $$
    $$ k_{local} = k \ \& \ (B_z - 1) $$
    [S2, unofficial]

2.  **Subtract 1:**
    $$ i_{tmp} = i_{local} - 1 $$
    $$ j_{tmp} = j_{local} - 1 $$
    $$ k_{tmp} = k_{local} - 1 $$
    [S2, unofficial]

3.  **Mask Zero Case:** AND with $(2B - 1)$. This maps any component that was 0 (now -1) to $2B-1$, while keeping other values in $[0, B-2]$.
    $$ i_{tmp} = i_{tmp} \ \& \ (2B_x - 1) $$
    $$ j_{tmp} = j_{tmp} \ \& \ (2B_y - 1) $$
    $$ k_{tmp} = k_{tmp} \ \& \ (2B_z - 1) $$
    [S2, unofficial]

4.  **Separate Boundary Case:** OR with 1, then add 1. This maps the value $B-1$ (which became $B-2$ after sub, then masked) to 2, and the value 0 (mapped to $2B-1$) remains separated. The result is a ternary variable $\{0, 1, 2\}$.
    $$ i'' = ((i_{tmp} \ | \ 1) + 1) $$
    $$ j'' = ((j_{tmp} \ | \ 1) + 1) $$
    $$ k'' = ((k_{tmp} \ | \ 1) + 1) $$
    [S2, unofficial]
    *Correction from Source Logic:* The source formula is $i' = ((((i \ \& \ (B_x-1)) - 1) \ \& \ (2 B_x - 1)) \ | \ 1) + 1$.
    Let's trace:
    - If $i_{local}=0$: $(0-1)=-1$. $-1 \ \& \ (2B-1) = 2B-1$. $(2B-1) \ | \ 1 = 2B-1$ (since $2B-1$ is odd). $(2B-1)+1 = 2B$.
    - If $i_{local}=B-1$: $(B-1-1)=B-2$. $(B-2) \ \& \ (2B-1) = B-2$. $(B-2) \ | \ 1$. If $B-2$ is even, it becomes $B-1$. If odd, $B-1$. Then $+1 \rightarrow B$.
    - If $0 < i_{local} < B-1$: Result is $i_{local}$.
    Wait, the source says "Now all the cases are mapped to $\{0,1,2\}$ to obtain a ternary system. This is done by dividing the components by the corresponding block dimensions."
    So the intermediate value $i''$ is divided by $B_x$ (shifted by $N_x$).
    $$ i_{ternary} = i'' \gg N_x $$
    $$ j_{ternary} = j'' \gg N_y $$
    $$ k_{ternary} = k'' \gg N_z $$
    [S2, unofficial]

5.  **Compute Index:**
    $$ Index = 9 \cdot i_{ternary} + 3 \cdot j_{ternary} + k_{ternary} $$
    [S2, unofficial]
    This maps the 27 cases to $[0, 26]$.

6.  **Lookup Offsets:** Retrieve 27 offsets from a gradient LUT (size not explicitly given in bytes, but 27 entries).

### Formulas
**Resampling Index:**
$$ Index = \frac{(((i \ \& \ (B_x-1)) + 1) \ \& \ \sim(B_x-1)) + (((j \ \& \ (B_y-1)) + 1) \ \& \ \sim(B_y-1)) + (((k \ \& \ (B_z-1)) + 1) \ \& \ \sim(B_z-1))}{\min(B_x, B_y, B_z)} $$
[ S1 ]

**Gradient Index:**
$$ i' = ((((i \ \& \ (B_x-1)) - 1) \ \& \ (2 B_x - 1)) \ | \ 1) + 1 $$
$$ j' = ((((j \ \& \ (B_y-1)) - 1) \ \& \ (2 B_y - 1)) \ | \ 1) + 1 $$
$$ k' = ((((k \ \& \ (B_z-1)) - 1) \ \& \ (2 B_z - 1)) \ | \ 1) + 1 $$
$$ Index = 9(i' \gg N_x) + 3(j' \gg N_y) + (k' \gg N_z) $$
[ S2 ]

### Parameters
- **Brick Dimensions:** $B_x, B_y, B_z$ should be powers of two. Optimal brick size is cited as $32^3$ [S1, unofficial].
- **Logarithmic Dimensions:** $N_x = \log_2 B_x$, etc. [S2, unofficial].
- **LUT Size:** 224 bytes for resampling (56 offsets $\times$ 4 bytes) [S1, S2, unofficial].

### Edge Cases and Pitfalls
- **Power of Two Constraint:** The index calculation relies on brick dimensions being powers of two [S1, S2, unofficial].
- **Power of Two Apart:** The resampling index formula assumes dimensions are "a power of two apart" to ensure the sum of masked values maps uniquely to $[0,7]$ via the shift. If dimensions are not distinct powers of two (e.g., $32 \times 32 \times 32$), the shift amount $\min(N_x, N_y, N_z)$ may cause collisions or incorrect mapping. The source suggests introducing a simple shift operation to virtually maintain the constraint or removing the constraint by other means [S1, unofficial].
- **Boundary Conditions:** The method handles brick boundaries implicitly by mapping boundary samples to specific cases in the LUT. No explicit boundary checks are needed in the kernel [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed LUT for resampling (7 neighbors)
# LUT is a list of 56 integers (4 bytes each)
# Structure: LUT[case_index][neighbor_index]
# neighbor_index: 0=(+1,0,0), 1=(0,+1,0), 2=(+1,+1,0), 3=(0,0,+1), 
#                 4=(+1,0,+1), 5=(0,+1,+1), 6=(+1,+1,+1)

def compute_resample_offsets(i, j, k, Bx, By, Bz, Nx, Ny, Nz, LUT):
    """
    Computes the 7 neighbor offsets for a sample at (i, j, k) in a bricked volume.
    
    Args:
        i, j, k: Integer coordinates of the current sample.
        Bx, By, Bz: Brick dimensions (must be powers of two).
        Nx, Ny, Nz: Logarithms of brick dimensions (Bx = 2^Nx, etc.).
        LUT: Global lookup table of size 56.
        
    Returns:
        List of 7 byte offsets relative to the current sample.
    """
    # Step 1: Extract brick offset
    i_local = i & (Bx - 1)
    j_local = j & (By - 1)
    k_local = k & (Bz - 1)
    
    # Step 2: Increment
    i_inc = i_local + 1
    j_inc = j_local + 1
    k_inc = k_local + 1
    
    # Step 3: Mask to boundary (maps 0..B-1 to 0, B to B)
    # ~ (B-1) is the bitwise NOT of (B-1)
    mask_x = ~(Bx - 1)
    mask_y = ~(By - 1)
    mask_z = ~(Bz - 1)
    
    i_masked = i_inc & mask_x
    j_masked = j_inc & mask_y
    k_masked = k_inc & mask_z
    
    # Step 4: Sum and shift to get index [0, 7]
    # min(Nx, Ny, Nz) is the shift amount
    shift = min(Nx, Ny, Nz)
    index = (i_masked + j_masked + k_masked) >> shift
    
    # Step 5: Retrieve offsets from LUT
    # LUT is indexed by case (0-7), then neighbor (0-6)
    # Flattened index: index * 7 + neighbor
    base_offset = index * 7
    
    offsets = []
    for n in range(7):
        offsets.append(LUT[base_offset + n])
        
    return offsets

def compute_gradient_offsets(i, j, k, Bx, By, Bz, Nx, Ny, Nz, LUT_grad):
    """
    Computes the 27 neighbor offsets for gradient computation.
    """
    # Step 1: Extract brick offset
    i_local = i & (Bx - 1)
    j_local = j & (By - 1)
    k_local = k & (Bz - 1)
    
    # Step 2: Subtract 1
    i_tmp = i_local - 1
    j_tmp = j_local - 1
    k_tmp = k_local - 1
    
    # Step 3: Mask zero case (maps -1 to 2B-1)
    mask_x = 2 * Bx - 1
    mask_y = 2 * By - 1
    mask_z = 2 * Bz - 1
    
    i_tmp = i_tmp & mask_x
    j_tmp = j_tmp & mask_y
    k_tmp = k_tmp & mask_z
    
    # Step 4: Separate boundary case and map to ternary
    # OR with 1, then add 1
    i_val = ((i_tmp | 1) + 1)
    j_val = ((j_tmp | 1) + 1)
    k_val = ((k_tmp | 1) + 1)
    
    # Step 5: Divide by B (shift) to get ternary {0, 1, 2}
    i_ternary = i_val >> Nx
    j_ternary = j_val >> Ny
    k_ternary = k_val >> Nz
    
    # Step 6: Compute index [0, 26]
    index = 9 * i_ternary + 3 * j_ternary + k_ternary
    
    # Step 7: Retrieve offsets
    # Assuming LUT_grad has 27 entries
    return LUT_grad[index]
```

## Subtopics

### Brick Volume Addressing with If-Else
The parent method that uses conditional branches to handle brick boundaries. Choose Lookup Table Addressing when branch prediction is poor or when maximizing instruction throughput is critical. [Brick Volume Addressing with If-Else](3340-brick-volume-addressing-with-if-else.md)

### Inflated Blocks
An alternative method that adds a border of samples to each brick to avoid cross-brick addressing. Choose Lookup Table Addressing when memory bandwidth is limited, as inflated blocks increase memory usage by ~20% for $32^3$ blocks, whereas LUT addressing uses no additional memory [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. The index calculation involves only bitwise operations, additions, and shifts, which are very fast on modern CPUs [S2, unofficial].
- **Memory Complexity:** The LUT size is constant and small (224 bytes for resampling), fitting entirely in L1 cache [S1, S2, unofficial].
- **Measured Performance:** The compressed LUT approach reduces table size by a factor of 4096 compared to a naive per-position lookup table (896 KB vs 224 bytes) [S1, unofficial]. This fits in cache, avoiding cache misses associated with larger tables [S1, unofficial].
- **Strengths:** No conditional branches, no expensive division/modulo operations, minimal memory overhead, high cache coherency [S1, S2, unofficial].
- **Limitations:** Requires brick dimensions to be powers of two. The resampling index formula has constraints on dimensions being "a power of two apart" to avoid index collisions [S1, unofficial].
- **When to Use:** Volume raycasting on hardware with limited cache or where branch misprediction penalties are high. Suitable for fixed brick sizes.
- **When Not to Use:** If brick dimensions cannot be powers of two or if the overhead of managing the LUT is prohibitive for very small volumes.

## Background
- **Origin:** The method was introduced in the context of efficient volume visualization of large medical datasets. It was proposed as an improvement over naive lookup tables and hierarchical addressing schemes [S1, S2, unofficial].
- **History:** Parker et al. [11] (cited in [S2, unofficial]) used a two-level subdivision hierarchy, which introduced addressing costs. This method uses one-level subdivision with a global LUT to avoid those costs [S2, unofficial]. The specific bitwise indexing technique for resampling is detailed in [S1, unofficial] and [S2, unofficial].
- **Usage:** Used in real-time mono- and multi-volume rendering of large medical datasets on standard PC hardware [S1, unofficial].

## Sources
[S1] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S2] Efficient Volume Visualization of Large Medical Datasets, https://old.cescg.org/CESCG-2004/web/Bruckner-Stefan/html/, fetched 2026-10-30 (unofficial)
