# Perfect Spatial Hashing

Also called: Perfect Spatial Hash, Perfect Spatial Hashing (PSH) Encoding

## Overview
Perfect Spatial Hashing (PSH) is a hashing technique for the compact storage and efficient indexing of sparse volume data, using an injective map to store volume data in a hash table and offsets in a separate table [S1]. It is a single-resolution spatial hashing encoding method that uses a precomputed offset table to ensure zero hash collisions and maximize cache coherence by mapping spatially adjacent points to adjacent slots in the hash table [S2, unofficial]. Unlike standard hash encoding which suffers from cache thrashing due to uniform memory distribution, PSH minimizes hash collisions and improves memory request efficiency [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of occupied block positions $S$ (where $|S| = N$) [S1]. For encoding queries, an input position $x$ [S2, unofficial].
*   **Output:** A hash table $H$ storing volume data and an offset table $\Phi$ [S1]. For a query $x$, the final encoding $E(x)$ [S2, unofficial].

**Data Structures:**
1.  **Hash Table $H$:** A 3D hash table of size $m = \bar{m}^3 \approx N$, storing volume data (feature vectors) [S1]. The table contains features $F$ indexed by hash values [S2, unofficial].
2.  **Offset Table $\Phi$:** A 3D offset table of size $r = \bar{r}^3 = \sigma N$, where $1/6 \le \sigma < 1$ [S1]. $\Phi$ is a lookup table for offsets constructed by a heuristic approach to minimize hash collisions [S2, unofficial].

**Parameters:**
*   $N$: Number of occupied blocks (size of set $S$) [S1].
*   $N$: Grid resolution used to scale input coordinates. Typical values tested: 128, 256, 512 [S2, unofficial].
*   $\Phi$: The offset table [S2, unofficial].

**Formulas:**
The hash function $h(p)$ maps an integer coordinate $p$ to a unique slot in the hash table $H$ [S1].

$$ h(p) = ((p \mod \bar{m}) + (\Phi[p] \mod \bar{r})) \mod \bar{m} $$

Where:
*   $h$ is the hash function [S1].
*   $p$ is the position (integer coordinate) [S1].
*   $\bar{m}$ is the cube root of the hash table size ($m = \bar{m}^3$) [S1].
*   $\bar{r}$ is the cube root of the offset table size ($r = \bar{r}^3$) [S1].
*   $\Phi[p]$ is the value from the offset table at index $p$ [S1].

Alternatively, expressed with two hashing functions $h_0$ and $h_1$:
$$ h(p) = h_0(p) + \Phi(h_1(p)) $$
Where $h_0$ maps to base indices and $h_1$ maps to indices $q$ for the offset table [S2, unofficial].

**Steps:**
1.  **Construct Offset Table:** Perform a Shape-from-Silhouette (SfS) algorithm to obtain a coarse occupancy field from static data [S2, unofficial]. Use a heuristic approach inspired by [Fox et al. 1992] to compute the offset table $\Phi$ from this occupancy field, ensuring that spatially adjacent points are mapped to adjacent slots in the hash table [S2, unofficial].
2.  **Scale Input:** For a given input position $x$, scale it by grid resolution $N$ and round to find the integer vertex coordinate $p = \lfloor N * x \rfloor$ [S2, unofficial].
3.  **Hash Coordinates:** Hash the integer coordinate $p$ using two hashing functions $h_0$ and $h_1$ to get base indices and offset indices [S2, unofficial]. Specifically, compute $q = h_1(p)$ [S2, unofficial].
4.  **Look Up Features:** Look up feature vectors from the hash table using corner indices derived from $h_0(p) + \Phi(h_1(p))$ [S2, unofficial]. This involves retrieving the feature vectors for each vertex of the voxel containing $x$ [S2, unofficial].
5.  **Interpolate:** Generate the final encoding $E(x)$ by tri-linearly interpolating the feature vectors of the voxel corners [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def construct_offset_table(occupied_positions, N, sigma):
    """
    Constructs the offset table Phi using a heuristic approach.
    occupied_positions: List of integer coordinates p in S.
    N: Number of occupied blocks.
    sigma: Scaling factor for offset table size (1/6 <= sigma < 1).
    Returns:
    Phi: 3D array of size r_bar^3.
    r_bar: Cube root of offset table size.
    m_bar: Cube root of hash table size.
    """
    # Calculate dimensions
    # m_bar^3 approx N. We assume m_bar is chosen such that m_bar^3 >= N.
    # r_bar^3 = sigma * N.
    m_bar = int(math.ceil(N ** (1/3)))
    r_bar = int(math.ceil((sigma * N) ** (1/3)))
    
    # Initialize tables
    # H is not explicitly built here, but Phi is required for hashing.
    # The heuristic approach [Fox et al. 1992] is used to find Phi.
    # Since the exact heuristic algorithm is not detailed in the sources,
    # we note that Phi is computed to ensure injectivity for p in S.
    
    # Placeholder for the heuristic computation of Phi
    # In practice, this involves iterating through occupied positions
    # and assigning offsets to avoid collisions.
    Phi = compute_offset_heuristic(occupied_positions, m_bar, r_bar)
    
    return Phi, r_bar, m_bar

def compute_offset_heuristic(occupied_positions, m_bar, r_bar):
    """
    Placeholder for the heuristic approach inspired by Fox et al. 1992.
    This function would assign values to Phi such that h(p) is injective
    for all p in occupied_positions.
    """
    # Not given in the sources.
    # Implementation would involve finding a Phi table that minimizes collisions.
    pass

def perfect_spatial_hash(p, m_bar, r_bar, Phi):
    """
    Computes the perfect spatial hash for a single integer coordinate p.
    """
    # h(p) = ((p mod m_bar) + (Phi[p] mod r_bar)) mod m_bar
    # Note: p is a 3D coordinate. The mod operation is applied component-wise
    # or the coordinate is flattened. The source formula implies component-wise
    # or a specific 3D mod interpretation.
    # Assuming p is a tuple (px, py, pz) or flattened integer.
    # If p is flattened, the formula applies to the integer.
    # If p is 3D, the mod is likely component-wise before combination or
    # the formula represents a specific 3D hashing scheme.
    # Given h(p) = h0(p) + Phi(h1(p)), let's assume standard 3D hashing.
    
    # Let's assume p is a 3D vector.
    # The formula h(p) = ((p mod m_bar) + (Phi[p] mod r_bar)) mod m_bar
    # suggests that p mod m_bar is a 3D vector, and Phi[p] is a 3D vector.
    # The addition and final mod are likely component-wise.
    
    # However, the alternative form h(p) = h0(p) + Phi(h1(p)) suggests
    # h0 and h1 are hash functions.
    
    # Let's stick to the explicit formula provided in [S1]:
    # h(p) = ((p mod m_bar) + (Phi[p] mod r_bar)) mod m_bar
    
    # We need to handle p as a 3D coordinate.
    # Let's assume p is a 3D vector (px, py, pz).
    # m_bar and r_bar are scalars.
    
    # Component-wise modulo
    p_mod_m = [coord % m_bar for coord in p]
    
    # Phi lookup. Phi is a 3D table.
    # We need to index Phi with p.
    # If p is outside the bounds of Phi, this is an error or requires wrapping.
    # The source says Phi is size r_bar^3.
    # We assume p is scaled and fits within the context of the hash.
    
    # Let's assume p is already within the bounds of the hash grid dimensions
    # or that Phi is indexed by the hash of p.
    # The formula says Phi[p]. This implies p is an index into Phi.
    # But p is a 3D coordinate.
    # This implies Phi is indexed by the 3D coordinate p.
    # But Phi size is r_bar^3.
    # This suggests that p is mapped to an index in Phi, or p is small enough.
    # Given the context of "integer vertex coordinate p", and Phi being an offset table,
    # it is likely that p is used to index Phi directly, implying p < r_bar.
    # Or, more likely, h1(p) is used to index Phi.
    
    # Let's use the alternative form for clarity in implementation:
    # h(p) = h0(p) + Phi(h1(p))
    
    # h0(p) and h1(p) are hash functions.
    # Common hash functions for 3D coordinates:
    # h0(p) = (p[0] * 73856093 ^ p[1] * 19349663 ^ p[2] * 83492791) % m_bar
    # h1(p) = (p[0] * 35797 ^ p[1] * 2654435761 ^ p[2] * 1597334677) % r_bar
    
    # However, the source does not specify h0 and h1.
    # It only gives the structure.
    # We will implement the structure as given in [S1] assuming p is an index.
    
    # Re-reading [S1]: "h(p) = ((p mod m_bar) + (Phi[p] mod r_bar)) mod m_bar"
    # This formula is ambiguous for 3D p.
    # Let's assume p is a flattened integer index or the formula is component-wise.
    # Given "injective map when p in S", and S is a set of positions.
    
    # Let's assume the implementation uses the structure:
    # index = h0(p) + Phi[h1(p)]
    
    # We need to define h0 and h1.
    # Since they are not defined, we cannot provide a complete pseudocode for h0/h1.
    # We will provide the structure.
    
    pass

def get_encoding(x, N, Phi, m_bar, r_bar):
    """
    Computes the final encoding E(x) for a position x.
    """
    # 1. Scale and round to find integer vertex coordinate p
    p = [math.floor(N * coord) for coord in x]
    
    # 2. Find corners of the voxel
    # The voxel containing x has corners at integer coordinates.
    # Since p = floor(N*x), the voxel is defined by p.
    # The corners are p and p+1 in each dimension.
    # There are 8 corners.
    
    corners = []
    for i in range(2):
        for j in range(2):
            for k in range(2):
                corner_p = [p[d] + (i if d==0 else 0) + (j if d==1 else 0) + (k if d==2 else 0) for d in range(3)]
                corners.append(corner_p)
    
    # 3. Hash each corner
    # We need to compute h(corner_p) for each corner.
    # Using the formula h(p) = h0(p) + Phi(h1(p))
    # We need h0 and h1.
    
    # Since h0 and h1 are not specified, we cannot compute the exact indices.
    # We will assume a standard hash function for demonstration.
    
    def hash_func_3d(coord, mod):
        # Simple hash function
        h = 0
        for c in coord:
            h = (h * 31 + c) % mod
        return h

    # 4. Look up features
    features = []
    for corner_p in corners:
        # Compute h0 and h1
        h0 = hash_func_3d(corner_p, m_bar)
        h1 = hash_func_3d(corner_p, r_bar)
        
        # Look up offset
        # Phi is indexed by h1? Or by corner_p?
        # Source [S2, unofficial] says: "indices q to offset table Phi respectively" where q = h1(p).
        # So Phi is indexed by h1(p).
        
        offset = Phi[h1] # Assuming Phi is 1D or flattened 3D
        
        # Compute final index
        index = (h0 + offset) % m_bar
        
        # Look up feature from Hash Table H
        # H is indexed by index.
        feature = H[index] # H is the hash table
        features.append(feature)
    
    # 5. Tri-linear interpolation
    # Interpolate the 8 features based on the fractional part of x.
    # fractional part of x in the voxel
    frac = [N * coord - math.floor(N * coord) for coord in x]
    
    # Tri-linear interpolation of 8 features
    # This requires the features to be ordered corresponding to the corners.
    # The order of corners in the list above is:
    # (0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)
    # relative to p.
    
    # Interpolation logic
    f000, f001, f010, f011, f100, f101, f110, f111 = features
    
    # Interpolate along x
    f00 = lerp(f000, f001, frac[2])
    f01 = lerp(f010, f011, frac[2])
    f10 = lerp(f100, f101, frac[2])
    f11 = lerp(f110, f111, frac[2])
    
    # Interpolate along y
    f0 = lerp(f00, f01, frac[1])
    f1 = lerp(f10, f11, frac[1])
    
    # Interpolate along z
    E_x = lerp(f0, f1, frac[0])
    
    return E_x

def lerp(a, b, t):
    return a + t * (b - a)
```

**Edge Cases and Pitfalls:**
*   **Cache Thrashing:** Standard hash encoding causes cache thrashing because memory access is distributed evenly across the entire hash table [S2, unofficial]. PSH mitigates this by mapping spatially adjacent points to adjacent slots [S2, unofficial].
*   **Memory Bandwidth Waste:** Independent memory requests at each level in multiresolution hash encoding waste memory bandwidth because only a small fraction of the returned cache line is used [S2, unofficial]. PSH uses single-resolution feature vectors, reducing memory requests to one per sample point and consuming half of the returned cache line [S2, unofficial].
*   **Collision:** The offset table $\Phi$ is constructed to ensure zero hash collisions for the static data [S2, unofficial]. If the heuristic fails to find a collision-free mapping, the injectivity is lost [S1].

## Operating characteristics
*   **Time Complexity:** Access time is O(1) for arbitrary locations [S1].
*   **Memory Complexity:** The hash table size is $m \approx N$ and the offset table size is $r = \sigma N$ where $1/6 \le \sigma < 1$ [S1].
*   **Measured Performance:** On the 'ficus' scene at 4K resolution, PSH-256 achieved 17.83 FPS and 94.18% L2 cache hit rate, compared to i-NGP's 14.38 FPS and 72.82% L2 hit rate [S2, unofficial].
*   **Strengths:** Maximizes cache coherence by mapping spatially adjacent points to adjacent slots [S2, unofficial]. Reduces memory requests to one per sample point [S2, unofficial].
*   **Limitations:** Requires precomputation of the offset table $\Phi$ from static data [S2, unofficial]. Single-resolution encoding may not capture multi-scale details as effectively as multiresolution approaches, though it offers better memory efficiency [S2, unofficial].

## Background
Perfect Spatial Hashing was introduced by Lefebvre and Hoppe in 2006 [S1]. It has been adapted for use in neural rendering, specifically to improve the memory request efficiency of i-NGP [S2, unofficial]. The offset table construction is inspired by a heuristic approach from Fox et al. 1992 [S2, unofficial].

## Sources
[S1] VMesh: Hybrid Volume-Mesh Representation for Efficient View Synthesis, https://core.ac.uk/download/560406650.pdf, fetched 2026-10-27
[S2] NEPHELE: A Neural Platform for Highly Realistic Cloud Radiance Rendering, https://arxiv.org/abs/2303.04086v1, fetched 2026-10-27 (unofficial)
