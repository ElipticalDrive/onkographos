# Multi-scale Voxel Hashing

Also called: Multi-scale voxel representation, Voxel hashing with merging

## Overview
Multi-scale Voxel Hashing is a sparse 3D representation algorithm designed for mobile Augmented Reality (AR) systems that converts sparse 3D point clouds from Visual-Inertial Odometry (VIO) systems into a hierarchical voxel structure [S1, unofficial]. It solves the problem of high memory footprint and inefficient collision detection inherent in uniform voxel grids by using spatial hashing and bottom-up merging to create variable-sized voxels [S1, unofficial]. It sets itself apart from uniform sparse voxel storage by allowing non-uniform voxel dimensions ($l_x, l_y, l_z$) and from SVO-based methods by relying on a flat hash table with linked lists rather than a tree structure, prioritizing real-time insertion and collision detection over surface reconstruction [S1, unofficial].

## How it works
**Inputs:** A stream of 3D points $(X, Y, Z)$ in world coordinates (e.g., from ARKit/ARCore).
**Outputs:** A hash table mapping integer keys to linked lists of voxel entries.

**Data Structures:**
1.  **Hash Table:** An array of size $n$. Each slot contains a pointer to a linked list of voxel entries.
2.  **Voxel Entry (Non-uniform):** A 7-tuple stored in 10 bytes:
    *   `x, y, z` (3 bytes total): Integer coordinates of the vertex closest to the origin.
    *   `lx, ly, lz` (3 bytes total): Scale levels for each dimension.
    *   `d` (4 bytes): Density (number of input points enclosed by the voxel).
    *   *Note:* Uniform voxels use a 5-tuple `{x, y, z, l, d}` where $l_x=l_y=l_z=l$.

**Parameters:**
*   $S$: Scaling integer. Multiplies world coordinates to ensure sufficient integer precision. Typical value $S=100$ for meters-to-centimeters conversion [S1, unofficial].
*   $n$: Hash table size.
*   $p_1, p_2, p_3$: Large prime numbers for hashing.
*   $d_{thresh}$: Density threshold for filtering outliers. Typical value $d < 50$ [S1, unofficial].
*   $m_v$: Minimum voxel size, determined by scene depth $D$:
    *   $D < 0.75$m $\rightarrow m_v = 2$cm
    *   $0.75$m $\le D < 1$m $\rightarrow m_v = 4$cm
    *   $D \ge 1$m $\rightarrow m_v = 8$cm [S1, unofficial].

**Algorithm Steps:**

1.  **Coordinate Scaling and Hashing:**
    For each input point $(X, Y, Z)$:
    *   Scale coordinates: $x' = \text{round}(X \cdot S)$, $y' = \text{round}(Y \cdot S)$, $z' = \text{round}(Z \cdot S)$.
    *   Compute hash key:
        $$H(x', y', z') = (x' \cdot p_1 \oplus y' \cdot p_2 \oplus z' \cdot p_3) \pmod n$$
        where $\oplus$ is bitwise XOR [S1, unofficial].

2.  **Insertion:**
    *   Access the linked list at hash table index $H(x', y', z')$.
    *   Perform a linear search through the list to check if the point is already encompassed by an existing voxel.
    *   If found, stop.
    *   If not found, create a new voxel entry at the smallest scale level ($l_x=l_y=l_z=0$, corresponding to $m_v$). Set density $d=1$.
    *   Insert the new entry at the end of the linked list.

3.  **Bottom-Up Merging (Non-Uniform):**
    After insertion, attempt to merge neighboring voxels to reduce memory footprint.
    *   **Uniform Merging (Inefficient for surfaces):** Requires 8 neighbors to form a larger voxel. Discarded for surface data [S1, unofficial].
    *   **Non-Uniform Merging:** Merge 4 voxels that share the same $z$-level ($l_z$) but differ in $x$ and $y$ levels by 1, forming a $2 \times 2$ block.
        *   Condition: Four voxels exist at $(x, y, z, l_x, l_y, l_z)$, $(x+1, y, z, l_x, l_y, l_z)$, $(x, y+1, z, l_x, l_y, l_z)$, and $(x+1, y+1, z, l_x, l_y, l_z)$.
        *   Action: Create a new voxel with $l_x' = l_x + 1$, $l_y' = l_y + 1$, $l_z' = l_z$.
        *   Density: $d' = \sum d_{constituents}$.
        *   Remove the 4 constituent voxels from the hash table.
    *   **Buddy Location:** Locate the 3 buddy voxels using integer division and multiplication on coordinates [S1, unofficial].
    *   **Ambiguity:** If multiple merge options exist (e.g., at plane intersections), choose one arbitrarily and ignore others [S1, unofficial].
    *   **Optional:** Allow merging 2 voxels along one axis or 3 out of 4 voxels for flexibility, though experiments showed minimal performance gain [S1, unofficial].

4.  **Room Orientation Alignment (Preprocessing):**
    To improve merging efficiency for planar structures:
    *   Guide user to scan vertical surfaces.
    *   Use RANSAC on 3D points to estimate a vertical plane perpendicular to the ground.
    *   Estimate ground plane orientation via motion sensing hardware (gravity).
    *   If vertical plane inliers > threshold, use its normal as the new X-axis.
    *   Rotate all 3D points to this new coordinate system. Y-axis remains opposite to gravity [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
P1, P2, P3 = 1000003, 1000033, 1000037 # Example large primes
N = 1000000 # Hash table size
S = 100     # Scaling factor

class Voxel:
    def __init__(self, x, y, z, lx, ly, lz, d):
        self.x = x
        self.y = y
        self.z = z
        self.lx = lx
        self.ly = ly
        self.lz = lz
        self.d = d

def hash_key(x, y, z):
    return ((x * P1) ^ (y * P2) ^ (z * P3)) % N

def get_buddy_coords(x, y, z, lx, ly, lz, axis):
    """
    Returns coordinates of the 3 buddies needed to form a merged voxel
    with the current voxel at the next level.
    For non-uniform merging (2D merge), we look for buddies in x and y.
    """
    buddies = []
    # Buddy 1: flip bit in x
    bx1 = x ^ (1 << lx)
    buddies.append((bx1, y, z, lx, ly, lz))
    # Buddy 2: flip bit in y
    by1 = y ^ (1 << ly)
    buddies.append((x, by1, z, lx, ly, lz))
    # Buddy 3: flip both
    buddies.append((bx1, by1, z, lx, ly, lz))
    return buddies

def insert_point(hash_table, x, y, z, lx, ly, lz, d):
    key = hash_key(x, y, z)
    # Linear search in linked list
    current = hash_table[key]
    while current:
        # Check if point is encompassed (simplified: exact coordinate match for hash slot)
        # In practice, check if voxel bounds contain point
        if voxel_contains(current, x, y, z):
            current.d += 1
            return current
        current = current.next
    
    # Create new voxel
    new_voxel = Voxel(x, y, z, lx, ly, lz, d)
    # Insert at end of list
    if not hash_table[key]:
        hash_table[key] = new_voxel
    else:
        tail = hash_table[key]
        while tail.next:
            tail = tail.next
        tail.next = new_voxel
    
    # Attempt merge
    try_merge(hash_table, new_voxel)
    return new_voxel

def try_merge(hash_table, voxel):
    # Check if 4 voxels (current + 3 buddies) exist and can merge
    buddies = get_buddy_coords(voxel.x, voxel.y, voxel.z, voxel.lx, voxel.ly, voxel.lz)
    
    # Verify all buddies exist in hash table
    # Note: This requires searching hash tables for each buddy's key
    # For simplicity, assume helper function lookup_voxel exists
    if all(lookup_voxel(hash_table, b) for b in buddies):
        # Merge
        new_lx = voxel.lx + 1
        new_ly = voxel.ly + 1
        new_lz = voxel.lz
        new_x = voxel.x & ~(1 << voxel.lx) # Align to new grid
        new_y = voxel.y & ~(1 << voxel.ly)
        new_z = voxel.z
        
        new_d = voxel.d + sum(b.d for b in buddies)
        new_voxel = Voxel(new_x, new_y, new_z, new_lx, new_ly, new_lz, new_d)
        
        # Remove old voxels from hash table
        remove_voxel(hash_table, voxel)
        for b in buddies:
            remove_voxel(hash_table, lookup_voxel(hash_table, b))
            
        # Insert new merged voxel
        insert_point(hash_table, new_x, new_y, new_z, new_lx, new_ly, new_lz, new_d)
```

## Subtopics

### Room Orientation Alignment
A preprocessing step that rotates 3D points so that room axes align with voxel axes, improving merging efficiency for planar structures. It uses RANSAC to estimate a vertical plane perpendicular to the ground (estimated via gravity sensors) and rotates the coordinate system so the plane's normal becomes the new X-axis [S1, unofficial].

### Voxel Collision Detection
A real-time collision detection method between digital objects (bounding boxes/cylinders) and the voxel map. It checks for box-box overlap (6 comparisons) and registers a collision if the count of collided voxels exceeds a threshold (e.g., 10). It filters out voxels close to the object's support plane to prevent unintended collisions with floor features [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion is $O(1)$ average case for hashing, but linear in the worst case for hash collisions (linked list traversal). Merging requires checking neighbors, which is constant time per voxel but depends on hash table lookup costs.
*   **Memory Complexity:** Proportional to the number of non-empty voxels. Compression ratios are high for simple scenes (e.g., Conference Room) and lower for complex scenes (e.g., Kitchen) [S1, unofficial].
*   **Performance:** Tested on iPhone 8 with ARKit, processing dozens to hundreds of points per frame in real time [S1, unofficial].
*   **Strengths:** Small memory footprint, real-time processing, efficient collision detection.
*   **Limitations:** Axis-aligned voxels may fail to merge if not aligned with room orientation. Hash collisions require linked lists. Interior surfaces are not represented well due to non-uniform merging [S1, unofficial].

## Background
Developed for mobile AR applications (ARKit/ARCore) to address the lack of collision detection and avoidance [S1, unofficial]. It builds on spatial hashing techniques used in [21] (not cited in provided text) and extends them with bottom-up merging. The method was presented in a CVPR 2018 workshop paper [S1, unofficial].

## Sources
[S1] CVPR 2018 Open Access Repository, https://openaccess.thecvf.com/content_cvpr_2018_workshops/w30/html/Xu_Multi-Scale_Voxel_Hashing_CVPR_2018_paper.html, fetched 2026-10-30 (unofficial)
