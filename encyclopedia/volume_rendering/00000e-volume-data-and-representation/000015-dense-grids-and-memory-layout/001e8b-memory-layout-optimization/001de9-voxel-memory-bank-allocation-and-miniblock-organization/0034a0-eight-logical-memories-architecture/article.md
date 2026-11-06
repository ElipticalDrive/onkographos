# Eight Logical Memories Architecture

Also called: Eight Logical Memory Banks

## Overview
The Eight Logical Memories Architecture is a memory layout optimization for dense volume rendering that maps voxel data to physical SDRAM banks to minimize access latency. It solves the problem of bank conflicts and high switching times by grouping eight neighboring subcubes into a logical unit that spans two physical SDRAM devices, assuming four banks per SDRAM. It sets itself apart from the parent [Voxel Memory Bank Allocation and Miniblock Organization](1638-voxel-memory-bank-allocation-and-miniblock-organization.md) by introducing a hierarchical grouping of banks into logical memories, and from its sibling [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md) by relying on coordinated precharging rather than spatial skewing to handle ray traversal.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel dataset with 32 bits per voxel.
*   **Output:** A physical memory layout mapping logical memory banks (A-H) to SDRAM devices and a command sequence for ray traversal.

**Data Structures:**
The architecture organizes memory into logical banks labeled A through H. Each logical bank corresponds to a specific region of the voxel space.
*   **Voxel Width:** 32 bits.
*   **SDRAM Width:** 16 bits.
*   **Devices per Logical Memory:** Two 16-bit wide SDRAM devices are required to service one logical memory (providing the 32-bit width).
*   **Banks per SDRAM:** 4 (labeled A, B, C, D in the source text for a single device).
*   **Logical Bank Size:** Each individual logical bank (A-H) covers a volume of $16^3$ voxels.
*   **Precharge Cube Size:** The cube encompassing the neighborhood requiring a precharge command is $32^3$ voxels.

**Method Steps:**
1.  **Map Subcubes to Banks:** Divide the voxel space into subcubes. Place neighboring subcubes in the x, y, and z directions into neighboring SDRAM banks.
2.  **Distribute Across SDRAMs:** Assign three of the neighboring subcubes to the same physical SDRAM device. Assign the remaining four neighboring subcubes to a second physical SDRAM device. This assumes each SDRAM has 4 internal banks.
3.  **Construct Logical Memory:** Combine the banks from the two SDRAMs to form one logical memory unit containing 8 logical banks (A-H).
4.  **Issue Precharge Commands:** When a ray traverses into the neighborhood of these eight logical banks, issue a precharge command across all eight banks simultaneously.
5.  **Optimize Switching:** By precharging all banks in the neighborhood, switching between any two banks within the logical memory takes only the time of a bank activate command, rather than a precharge-activate sequence.

**Formulas:**
*   **Logical Bank Volume:** $V_{bank} = 16^3$ voxels [S1].
*   **Precharge Neighborhood Volume:** $V_{precharge} = 32^3$ voxels [S1].
*   **Data Width Requirement:** $W_{voxel} = 32$ bits, requiring $N_{devices} = 2$ devices of width $W_{device} = 16$ bits [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def init_eight_logical_memories(voxel_data, width_32bit=True):
    """
    Initializes the memory layout for the Eight Logical Memories Architecture.
    voxel_data: 3D array of voxels.
    width_32bit: Boolean, must be True for this architecture as per source.
    """
    if not width_32bit:
        raise ValueError("Eight Logical Memories requires 32-bit voxels.")
    
    # Constants from source
    BANK_SIZE_VOXELS = 16
    PRECHARGE_CUBE_SIZE = 32
    BANKS_PER_SDRAM = 4
    DEVICES_PER_LOGICAL_MEM = 2
    
    # Logical banks A-H
    # Each logical bank covers a 16^3 voxel region.
    # The 8 logical banks are distributed across 2 SDRAMs.
    # SDRAM 0 provides 3 logical banks (e.g., A, B, C)
    # SDRAM 1 provides 4 logical banks (e.g., D, E, F, G)
    # Note: Source mentions 8 banks total (A-H). 
    # 3 + 4 = 7. The 8th bank (H) distribution is not explicitly detailed 
    # beyond "neighboring subcubes... into neighbouring SDRAM banks".
    # We assume a symmetric distribution or that the 8th bank is handled 
    # by the second SDRAM as well, or the source implies 3+4+1 logic 
    # but explicitly states "place three into the same SDRAM and use a 
    # second SDRAM to provide four more". This sums to 7. 
    # However, the architecture is named "Eight Logical Memories".
    # We assume the 8th bank is part of the second SDRAM's allocation 
    # or the "four more" includes the remainder. 
    # Given the constraint "assuming 4 banks per SDRAM", 
    # SDRAM 0 uses 3 banks. SDRAM 1 uses 4 banks. Total 7 banks used?
    # Or perhaps the 8th bank is a duplicate or the source text has a 
    # slight inconsistency between "Eight" and "3+4".
    # We implement the explicit mapping described:
    
    logical_banks = {
        'A': SDRAM_0_Bank_0,
        'B': SDRAM_0_Bank_1,
        'C': SDRAM_0_Bank_2,
        'D': SDRAM_1_Bank_0,
        'E': SDRAM_1_Bank_1,
        'F': SDRAM_1_Bank_2,
        'G': SDRAM_1_Bank_3,
        'H': SDRAM_1_Bank_0 # Placeholder for 8th bank, likely SDRAM 1 Bank 0 reused or source implies 8 banks total via 2 SDRAMs with 4 banks each, but only 7 are explicitly mapped in the text snippet. 
                             # [G] Assuming standard 8-bank logical grouping where the 8th bank is the next available bank in the sequence, likely on SDRAM 1.
    }
    
    return logical_banks

def ray_traverse_precharge(ray_origin, ray_direction, logical_banks):
    """
    Issues precharge commands when entering the neighborhood.
    """
    # Determine if ray is in the neighborhood of the 8 logical banks
    if is_in_neighborhood(ray_origin, logical_banks):
        # Issue precharge across all 8 banks
        for bank in logical_banks.values():
            bank.precharge()
        
        # Subsequent accesses within this neighborhood only require activate
        # This reduces switch time from (precharge + activate) to (activate)
```

**Parameters:**
*   **Bits per voxel:** 32 bits [S1].
*   **Effective size of each individual bank:** $16^3$ voxels [S1].
*   **Size of the cube between precharge commands:** $32^3$ voxels [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The architecture assumes 32-bit voxels. Using 16-bit voxels would require different device configurations not described in the source [S1].
*   **Boundary Conditions:** The precharge command is issued based on the ray entering the neighborhood of the eight banks. Care must be taken to ensure the neighborhood definition aligns with the $32^3$ voxel cube size [S1].

## Subtopics

### Four DIMMs Architecture
A low-cost variant using four Dual In-line Memory Modules (DIMMs) for a single PCI board solution [S1]. It replicates data to allow fetching eight voxels, storing two consecutive voxels instead of one [S1]. It calculates only four addresses (M0-M3 if z is even, M4-M7 if z is odd) [S1]. It uses SDRAMs with 4 internal banks (A-D) [S1]. The voxel neighborhood contained in all caches is reduced by 50% compared to the eight logical memory architecture, increasing bank activates and precharge commands, but this has hardly any impact on frame-rate due to large page caches [S1]. Performance loss is less than 1 frame per second compared to the eight logical memory architecture [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in terms of Big-O, but the architecture aims for near-optimal performance by reducing bank switch time to a single activate command [S1].
*   **Memory Complexity:** Requires two 16-bit wide SDRAM devices per logical memory [S1].
*   **Measured Performance:** In tests on a 256^3 voxel dataset (CT scan of a human foot), the difference in frame rate between the four DIMMs architecture and the eight logical memories architecture is always less than 1 frame per second [S1].
*   **Strengths:** Practical solution using modern SDRAM switching characteristics; near-optimal performance; allows for a single PCI board solution with four DIMMs [S1].
*   **Limitations:** Requires data replication in the four DIMM variant; reduced cache neighborhood in the four DIMM variant leads to more bank commands [S1].
*   **When to use:** When implementing interactive ray casting on PCI-based hardware with limited board space (single PCI board) and cost constraints [S1].

## Background
The Eight Logical Memories Architecture was presented in the paper "A low-cost memory architecture for PCI-based interactive ray casting" [S1]. It builds on the VIZARD II design which proposed 32 bits per voxel [S1]. The architecture was developed to provide a practical, low-cost solution for interactive volume rendering on PCI boards, leveraging the shorter switching times between banks in modern SDRAMs [S1].

## Sources
[S1] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
