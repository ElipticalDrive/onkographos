# Intelligent Memory Module (IMM) Architecture

Also called: DRAM Bank Interleaving, Volume Memory Architecture

## Overview
The Intelligent Memory Module (IMM) architecture is a specialized hardware memory layout for volume rendering accelerators designed to maximize the utilization of DRAM Page Mode for sequential ray traversal. It solves the problem of memory access conflicts and latency by organizing voxel data into "P-blocks" (32x64x32 subvolumes) that map to single DRAM pages, ensuring that contiguous ray samples reside in the same page. It sets itself apart from sibling methods like [Skewed Memory Organization](2351-skewed-memory-organization-for-conflict-free-ray-access.md) by relying on physical page locality and bank interleaving rather than coordinate permutation to avoid conflicts.

## How it works
**Inputs and Outputs**
*   **Input:** Ray traversal coordinates (Reference Point $P, Q, R$) and set type (S, A, B, or C).
*   **Output:** Memory addresses for 8 samples (one S-set) delivered per clock cycle, or wait states if page boundaries are crossed.

**Data Structures and Layout**
The memory system consists of 128 DRAM devices (2Mx8, 2048 rows x 1024 columns x 8 bits), totaling 256 MByte capacity [S1].
*   **Memory Unit:** A pair of DRAM devices forms one 16-bit memory unit. There are 64 such units.
*   **Intelligent Memory Module (IMM):** 8 memory units are grouped into one IMM. Each unit has independent address registers and control logic.
*   **Volume Memory:** 8 IMMs build the complete volume memory.
*   **Row Capacity:** One row across all 128 devices holds 1 Mbit, equivalent to 64K samples [S1].
*   **P-block:** A 32x64x32 subvolume of samples is placed into one DRAM page. This is the fundamental unit of page locality [S1].
*   **Cache:** A one-entry cache (fast register) is placed behind each memory unit to handle repeated accesses to the same sample without penalty [S1].

**Address Calculation**
To avoid access conflicts, samples are grouped according to the least significant bits of their coordinates ($X_0, Y_0, Z_0$). The bank number $\beta$ is calculated as:
$$ \beta = Z_0 Y_0 X_0 $$
where $X_0, Y_0, Z_0$ are the least significant bits of the X, Y, and Z coordinates respectively [S1].

For tri-linear interpolation, the address function $\sigma$ transforms coordinates into 8 linear memory addresses. Let $P, Q, R$ be the reference point coordinates. The coordinates for the 8 samples in a set are determined by the bank number $\beta$ (represented by bits $\beta_0, \beta_1, \beta_2$).

For an **S-set** defined by Reference Point $(P, Q, R)$, the coordinates $(u, v, w)$ for bank $\beta$ are:
$$
u = \lfloor P/2 \rfloor + P_0 \quad \text{for } \beta_0 = 0; \quad u = \lfloor P/2 \rfloor \quad \text{for } \beta_0 = 1
$$
$$
v = \lfloor Q/2 \rfloor + Q_0 \quad \text{for } \beta_1 = 0; \quad v = \lfloor Q/2 \rfloor \quad \text{for } \beta_1 = 1
$$
$$
w = \lfloor R/2 \rfloor + R_0 \quad \text{for } \beta_2 = 0; \quad w = \lfloor R/2 \rfloor \quad \text{for } \beta_2 = 1
$$
where $P_0, Q_0, R_0$ are the least significant bits of $P, Q, R$ [S1].

For **A-sets**, the $u$ coordinate calculation changes:
$$
u = \lfloor P/2 \rfloor + P_0 \quad \text{for } \beta_0 = 0; \quad u = (-1)^{P_0} + \lfloor P/2 \rfloor \quad \text{for } \beta_0 = 1
$$
$$
v = (-1)^{Q_0} + \lfloor Q/2 \rfloor \quad \text{for } \beta_1 = 1
$$
$$
w = (-1)^{R_0} + \lfloor R/2 \rfloor \quad \text{for } \beta_2 = 1
$$
[Note: The source provides specific lines for A, B, and C sets but implies the other coordinates remain as in the S-set unless specified. The text explicitly lists replacements for specific lines. For A-set, it replaces the first line (u) and specifies v and w conditions for $\beta_1=1, \beta_2=1$. It is implied that for $\beta_1=0, \beta_2=0$, the S-set logic applies or defaults. The source text is fragmented here. We strictly follow the provided equations (7)-(10).]

For **B-sets**, the $v$ coordinate calculation changes:
$$
v = \lfloor Q/2 \rfloor + Q_0 \quad \text{for } \beta_1 = 0; \quad v = (-1)^{Q_0} + \lfloor Q/2 \rfloor \quad \text{for } \beta_1 = 1
$$
[Source specifies replacement for the second line] [S1].

For **C-sets**, the $w$ coordinate calculation changes:
$$
w = \lfloor R/2 \rfloor + R_0 \quad \text{for } \beta_2 = 0; \quad w = (-1)^{R_0} + \lfloor R/2 \rfloor \quad \text{for } \beta_2 = 1
$$
[Source specifies replacement for the third line] [S1].

The final linear address $N(\beta)$ is calculated based on the set type and access mode (Page Mode vs. Random). For Page Mode (column) accesses:
$$ N(\beta) = W_{7..1}(\beta) \times 2^7 + V_{4..1}(\beta) \times 2^3 + U_{3..1}(\beta) $$
For Page Mode (row) accesses or other modes, the bit ranges differ ($W_{7..1}, V_{7..5}, U_{7..4}$) [S1].

**Command Word Structure**
A 16-bit command word is broadcast to all 8 units in an IMM. It contains:
*   11 or 10 bits for row/column address.
*   3 bits for unit address.
*   2 or 3 bits for command word (row access, read, write, no operation) [S1].
The bus width from the Address Sequencer (ASQ) to the Volume Memory is 128 bits (8 units $\times$ 16 bits) [S1].

**Execution Steps**
1.  **Address Generation:** The ASQ generates the coordinates for the next S-set (8 samples) based on the ray direction and current position.
2.  **Bank Selection:** Calculate the bank number $\beta$ for each of the 8 samples using the least significant bits of their coordinates.
3.  **P-block Check:** Determine if all 8 samples reside within the same P-block (32x64x32 subvolume).
4.  **Cache Check:** If the access is to the same sample as the previous cycle, the one-entry cache provides the data immediately, avoiding memory latency [S1].
5.  **Page Mode Access:** If all samples are in the same P-block and the page is loaded, issue one set of addresses per clock cycle. One S-set is delivered each clock cycle [S1].
6.  **High-Quality Mode:** In High-Quality rendering, samples for S-, A-, B-, and C-sets are fetched within 4 clock cycles, provided no set is distributed across more than one P-block [S1].
7.  **Write Cycle:** For writes, the host places 8 samples into bank data registers (sequentially or simultaneously). One S-set can be loaded each clock, resulting in a load time of ~0.35ms for a $256^3$ dataset [S1].
8.  **Page Miss Handling:** If a set is spread over multiple P-blocks, the scheduler halts the ASQ pipeline, inserts wait-states, sends a row access command to the required memory banks, and issues the next read command after two additional wait-states [S1].
9.  **Resample Detection:** The scheduler detects if a resample location steps outside the 26-neighborhood of its predecessor. If so, read operations become random accesses, bypassing the page mode optimization [S1].

**Parameters**
*   **DRAM Organization:** 2Mx8 (2048 rows x 1024 columns x 8 bits) [S1].
*   **Total Devices:** 128 (using 16MBit technology) [S1].
*   **Memory Capacity:** 256 MByte [S1].
*   **Page Mode Cycle Time:** ~40ns [S1].
*   **Row Access Time:** ~60ns [S1].

**Edge Cases and Pitfalls**
*   **Cross-Page Access:** If a set is distributed across more than one P-block, the performance penalty is significant due to pipeline stalls and row access commands [S1].
*   **Random Access:** Resampling locations stepping outside the 26-environment of the previous sample force random accesses, negating the benefits of the IMM architecture [S1].
*   **Synchronization:** The scheduler maintains synchronization between different memory banks. There are no explicit synchronization elements between the memory units themselves, relying on the central scheduler [S1].

## Operating characteristics
*   **Time Complexity:** O(1) per sample under ideal conditions (same P-block). O(N) for page misses where N is the number of wait states required for row access.
*   **Memory Complexity:** 256 MByte for a $512^3$ dataset (16 bits per sample) [S1].
*   **Performance:** In High-Speed Rendering mode, one S-set is delivered per clock cycle as long as movement remains within one P-block. In High-Quality mode, S, A, B, and C sets are fetched within 4 clock cycles if they do not cross P-block boundaries [S1].
*   **Strengths:** Maximizes DRAM Page Mode usage; low latency for sequential ray traversal; efficient handling of tri-linear interpolation sets.
*   **Limitations:** Performance degrades significantly if ray traversal crosses P-block boundaries frequently; requires complex address calculation logic; not suitable for random access patterns.
*   **When to use:** Sequential volume rendering with high coherence in ray traversal.
*   **When not to use:** Random access patterns, resampling outside local neighborhoods, or scenes with high discontinuity causing frequent P-block crossings.

## Background
The IMM architecture was developed for the COLOSSUS and COMET volume graphics subsystems, described in a paper on a scalable architecture for volume rendering [S1]. The "back end" of these accelerators was under development at the time of publication. The architecture addresses the memory bandwidth and latency bottlenecks in early volume rendering hardware by optimizing for the access patterns of ray casting and tri-linear interpolation.

## Sources
[S1] A scalable architecture for volume rendering, https://doi.org/10.1016/0097-8493(95)00044-5, fetched 2026-10-24
