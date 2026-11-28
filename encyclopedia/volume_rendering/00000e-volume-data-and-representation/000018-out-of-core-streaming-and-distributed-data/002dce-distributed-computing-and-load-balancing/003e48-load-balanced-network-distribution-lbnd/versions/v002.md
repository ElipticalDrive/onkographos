# Load Balanced Network Distribution (LBND)


## Overview
LBND is an optimization problem in distributed volume rendering where volumetric data is partitioned across a cluster to minimize the end-to-end rendering time, which is determined by the slowest render node, image composition time, and network communication overhead [S1, unofficial]. It differs from static space-subdivision methods (like kd-trees or octrees) by explicitly guiding partitioning based on the distribution of empty space and computational cost, rather than relying on geometric uniformity, thereby addressing load imbalance issues in sparse datasets [S1, unofficial].

## Shared principles
The LBND framework operates under the following shared principles for distributed ray casting:

1.  **Objective Function:** The goal is to minimize the end-to-end rendering time $T_{total}$. This time is restricted by the slowest render node in the cluster. The cost function reflects the maximum completion time among all processors, incorporating rendering cost, network communication time, and image composition time [S1, unofficial].
2.  **Precedence Order:** A strict view-dependent priority order is required for correct image composition. The precedence order is defined as the relative distance of each voxel (or partition unit) with respect to a given view direction [S1, unofficial]. Partitions must be assigned such that this order is maintained across nodes to allow for correct alpha compositing.
3.  **Partitioning Granularity:** Data is divided into units (bricks or subvolumes) sized for rendering hardware. The partitioning must balance two conflicting constraints:
    *   **Load Balance:** Partitions should be small enough to distribute computational load evenly based on data density (non-empty space).
    *   **Priority Continuity:** Partitions must be large enough or structured such that local images rendered on a node have consecutive depth priorities, minimizing the need for complex inter-node synchronization during compositing [S1, unofficial].
4.  **Compositing Strategy:** Images are composited into a final image using an alpha blending operation. The alpha compositing operator is associative, allowing for hierarchical composition. To minimize network bandwidth, each render node composites its local contributions into a single intermediate image before transmitting it for global blending. This requires that the local partitions on a node maintain the correct relative depth precedence [S1, unofficial].
5.  **Cost Factors:** The optimization considers:
    *   **Render Cost:** Computation time per voxel, influenced by opacity and transfer function evaluation.
    *   **Empty Space:** Regions with no data require little computation but consume partition slots if not handled correctly.
    *   **Network Cost:** Bandwidth required to transport data to nodes and intermediate images between nodes [S1, unofficial].

## Subtopics

### Brick Grouping
A dynamic programming solution that starts with a directed acyclic graph of segmented and cropped bricks. It groups bricks to minimize the cost function while respecting priority constraints. It is chosen when fine-grained control over brick composition is needed to handle irregular sparsity [S1, unofficial].

### Dynamic Programming Solutions
General algorithmic approaches used to solve the LBND optimization problem. These methods attempt to minimize the end-to-end rendering cost without violating image composition priority constraints. They are chosen when exact optimization of the partitioning is feasible within computational limits [S1, unofficial].

## Background
The LBND problem was identified in the context of distributed ray casting on clusters, where traditional space-subdivision methods (kd-trees, octrees, grids) failed to achieve good load balancing for datasets with large variations in sparsity [S1, unofficial]. Lombeyda et al. demonstrated the arithmetic equivalence of single ray casting composite computations and combined smaller composite computations, establishing the associativity of the alpha compositing operator which enables flexible partitioning strategies [S1, unofficial]. The specific LBND formulation and dynamic programming solutions were developed to address the granularity issues inherent in static partitions, where non-uniform empty space leads to disproportionate computational loads [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
