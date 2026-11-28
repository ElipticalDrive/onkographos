# Eight Logical Memories Architecture


## Overview
The Eight Logical Memories Architecture is a memory layout optimization for dense volume rendering that maps voxel data to physical SDRAM banks to minimize access latency [S1]. It solves the problem of high switching times by grouping neighboring subcubes into a logical unit that spans two physical SDRAM devices, assuming four banks per SDRAM [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel dataset with 32 bits per voxel [S1].
*   **Output:** A physical memory layout mapping logical memory banks to SDRAM devices and a command sequence for ray traversal [S1].

**Data Structures:**
The architecture organizes memory into logical banks labeled A through H [S1].
*   **Voxel Width:** 32 bits [S1].
*   **Devices per Logical Memory:** Two 16-bit wide SDRAM devices are required to service one logical memory, providing the 32-bit width [S1].
*   **Banks per SDRAM:** 4 internal banks per SDRAM device [S1].
*   **Logical Bank Size:** Each individual logical bank covers a volume of $16^3$ voxels [S1].
*   **Precharge Cube Size:** The cube encompassing the neighborhood requiring a precharge command is $32^3$ voxels [S1].

**Method Steps:**
1.  **Map Subcubes to Banks:** Divide the voxel space into subcubes and place neighboring subcubes in the x, y, and z directions into neighboring SDRAM banks [S1].
2.  **Distribute Across SDRAMs:** Assign three of the neighboring subcubes to the same physical SDRAM device and assign the remaining four neighboring subcubes to a second physical SDRAM device, assuming 4 banks per SDRAM [S1]. (Note: The source describes a 3+4 distribution totaling seven subcubes, while naming the architecture eight logical memories [S1].)
3.  **Construct Logical Memory:** Combine the banks from the two SDRAMs to form one logical memory unit containing 8 logical banks (A–H) [S1].
4.  **Issue Precharge Commands:** When a ray passes into the neighborhood of the eight banks, issue a precharge command across all eight banks simultaneously [S1].
5.  **Optimize Switching:** By precharging all banks in the neighborhood, switching between any two banks within the logical memory takes only the time of a bank activate command instead of a precharge-activate sequence [S1].

**Formulas:**
*   **Logical Bank Volume:** $V_{bank} = 16^3$ voxels [S1].
*   **Precharge Neighborhood Volume:** $V_{precharge} = 32^3$ voxels [S1].
*   **Data Width Requirement:** $W_{voxel} = 32$ bits, requiring $N_{devices} = 2$ devices of width $W_{device} = 16$ bits [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def init_eight_logical_memories(voxel_data, width_32bit=True):
    """Initializes the memory layout for the Eight Logical Memories Architecture."""
    if not width_32bit:
        raise ValueError("Eight Logical Memories requires 32-bit voxels.")
    
    # Constants from source [S1]
    BANK_SIZE = 16
    PRECHARGE_CUBE_SIZE = 32
    BANKS_PER_SDRAM = 4
    DEVICES_PER_LOGICAL_MEM = 2

    # Map logical banks A-H to physical SDRAM banks
    # SDRAM 0 provides 3 logical banks, SDRAM 1 provides 4 logical banks [S1]
    logical_banks = {
        'A': ('SDRAM_0', 0), 'B': ('SDRAM_0', 1), 'C': ('SDRAM_0', 2),
        'D': ('SDRAM_1', 0), 'E': ('SDRAM_1', 1), 'F': ('SDRAM_1', 2),
        'G': ('SDRAM_1', 3), 'H': ('SDRAM_1', 0)
    }
    return logical_banks

def ray_traverse_precharge(ray_origin, ray_direction, logical_banks, current_cube_origin):
    """Issues precharge commands when entering the neighborhood."""
    # Check if ray has entered the 32^3 voxel precharge neighborhood [S1]
    in_neighborhood = (
        abs(ray_origin.x - current_cube_origin.x) < PRECHARGE_CUBE_SIZE and
        abs(ray_origin.y - current_cube_origin.y) < PRECHARGE_CUBE_SIZE and
        abs(ray_origin.z - current_cube_origin.z) < PRECHARGE_CUBE_SIZE
    )
    if in_neighborhood:
        # Issue precharge across all eight logical banks simultaneously [S1]
        for bank_id in logical_banks.values():
            issue_precharge_command(bank_id)
        # Subsequent accesses within this neighborhood only require activate [S1]
        return True
    return False

def issue_precharge_command(bank_id):
    """Simulates hardware precharge command for the specified SDRAM bank."""
    # Hardware-specific precharge sequence for bank_id[0] + "_Bank_" + str(bank_id[1])
    pass
```

**Parameters:**
*   **Bits per voxel:** 32 bits [S1].
*   **Effective size of each individual bank:** $16^3$ voxels [S1].
*   **Size of the cube between precharge commands:** $32^3$ voxels [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The architecture assumes 32-bit voxels; using 16-bit voxels would require different device configurations not described in the source [S1].
*   **Boundary Conditions:** The precharge command is issued based on the ray entering the neighborhood of the eight banks [S1].

## Subtopics

### Four DIMMs Architecture
A low-cost variant using four Dual In-line Memory Modules (DIMMs) for a single PCI board solution [S1]. It replicates data to allow fetching eight voxels, storing two consecutive voxels instead of one [S1]. It calculates only four addresses (M0–M3 if the z coordinate is even, M4–M7 if the z coordinate is odd) [S1]. It uses SDRAMs with 4 internal banks (A–D) [S1]. The voxel neighborhood contained across all caches of all memory devices is reduced by 50% compared to the eight logical memory architecture, increasing bank activates and precharge commands, but this has hardly any impact on frame-rate due to relatively large page caches [S1]. Performance loss is always less than 1 frame per second compared to the eight logical memory architecture [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in terms of Big-O, but the architecture aims for near-optimal performance by reducing bank switch time to a single activate command [S1].
*   **Memory Complexity:** Requires two 16-bit wide SDRAM devices per logical memory [S1].
*   **Measured Performance:** On a $256^3$ voxel dataset (CT scan of a human foot), the difference in frame rate between the four DIMMs architecture and the eight logical memories architecture is always less than 1 frame per second [S1].
*   **Strengths:** Practical solution leveraging modern SDRAM switching characteristics; near-optimal performance; enables a single PCI board solution running at full 100MHz [S1].
*   **Limitations:** Requires data replication in the four DIMM variant; reduced cache neighborhood in the four DIMM variant increases bank commands [S1].
*   **When to use:** When implementing interactive ray casting on PCI-based hardware with limited board space and cost constraints [S1].

## Background
The Eight Logical Memories Architecture was presented in the paper "A low-cost memory architecture for PCI-based interactive ray casting" [S1]. It builds on the VIZARD II design which proposed 32 bits per voxel [S1]. The architecture was developed to provide a practical, low-cost solution for interactive volume rendering on PCI boards, leveraging the shorter switching times between banks in modern SDRAMs [S1].

## Sources
[S1] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
