# RobotSDF
Also called: Robot Model to SDF

## Overview
RobotSDF generates a Signed Distance Field (SDF) representation of a multi-link robot model conditioned on a specific joint configuration, supporting batched queries across multiple configurations and spatial points [S1, unofficial]. It differs from its siblings by explicitly composing individual link SDFs via forward kinematics transforms rather than treating the robot as a static mesh or a simple volumetric union; it builds on [Signed Distance Function (SDF) Generation and Processing](5342-signed-distance-function-sdf-generation-and-processing.md) by managing the dynamic composition of these implicit volumes. Trade-offs include higher computational cost per query compared to pre-baked volumes, but it provides exact geometric fidelity for any joint state without discretization artifacts inherent to voxelization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `pytorch_kinematics.Chain` object describing the robot topology and link geometries (meshes); a joint configuration vector (or batch of vectors); query points in the object frame.
*   **Output:** SDF values $d(\mathbf{x})$ for each query point; optionally, gradients $\nabla d(\mathbf{x})$ and surface normals.

**Data Structures:**
*   `RobotSDF`: Inherits from `sdf.ObjectFrameSDF`.
*   `link_sdf_cls`: A factory function (default `sdf.MeshSDF`) that creates an `ObjectFrameSDF` instance for each visual link.
*   `ComposedSDF`: An internal structure holding a list of link SDFs and their corresponding object-to-link transformation matrices.
*   `configuration_batch`: Stores multiple joint configurations reshaped to $(-1, M)$ where $M$ is the number of joints.

**Steps:**

1.  **Initialization:**
    *   Iterate over frame names in the `chain`.
    *   For each visual geometry of type 'mesh', instantiate a `MeshObjectFactory` and create a link SDF using `link_sdf_cls`.
    *   Store the link names and their initial offsets.
    *   Stack offset transforms and create a `ComposedSDF` instance from the list of link SDFs and object-to-link frames [S1, unofficial].

2.  **Set Joint Configuration:**
    *   If `joint_config` has multiple batch dimensions, save them in `configuration_batch` and reshape to $(-1, M)$.
    *   Compute forward kinematics using `pk.build_serial_chain_from_urdf` or equivalent. If using `SerialChain`, set `end_only=False` to retrieve all link transforms; otherwise use default behavior [S1, unofficial].
    *   Extract transformation matrices $T_{link}$ for each link name.
    *   Compute the object-to-link frame transforms: $T_{obj \to link} = T_{offset}^{-1} \cdot T_{link}$.
    *   If `configuration_batch` exists, expand the offset transform matrix dimensions and repeat to match the batch size before reshaping [S1, unofficial].

3.  **Query (SDF Evaluation):**
    *   Pass `points_in_object_frame` and `compute_grad` flag to the underlying `ComposedSDF`.
    *   For each link SDF, evaluate the distance to the query point in the link's local frame.
    *   Transform the query point to the link's frame using $T_{obj \to link}$.
    *   The global SDF value is the minimum of the local SDF values across all links:
        $$ d_{global}(\mathbf{x}) = \min_{i} d_{link_i}(T_{obj \to link_i} \mathbf{x}) $$
    *   If gradients are requested, compute the gradient for each link and transform it back to the object frame using the Jacobian of the transformation. The global gradient is the gradient corresponding to the minimum distance link [S1, unofficial].

4.  **Optimization (Compilation):**
    *   For each link SDF, if it is a `CachedSDF`, compile its `_forward_no_grad` method using `torch.compile` for faster queries [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import pytorch_kinematics as pk
from pytorch_volumetric import sdf

class RobotSDF:
    def __init__(self, chain, default_joint_config=None, path_prefix='',
                 link_sdf_cls=sdf.MeshSDF):
        self.chain = chain
        self.link_sdfs = []
        self.link_names = []
        self.offset_transforms = []
        
        # Initialization: Create SDFs for each mesh link
        for frame_name in chain.frame_names:
            # Check if frame has visual mesh geometry
            if chain.get_visual_geometry(frame_name) is not None:
                # Create factory and SDF for this link
                factory = sdf.MeshObjectFactory(
                    path_prefix=path_prefix,
                    geometry=chain.get_visual_geometry(frame_name)
                )
                link_sdf = link_sdf_cls(factory)
                self.link_sdfs.append(link_sdf)
                self.link_names.append(frame_name)
                # Store initial offset (identity or base transform)
                self.offset_transforms.append(torch.eye(4))
        
        self.num_links = len(self.link_sdfs)
        self.joint_config = default_joint_config if default_joint_config is not None else torch.zeros(self.chain.num_joints)
        
    def set_joint_configuration(self, joint_config):
        """Compute forward kinematics and update link transforms."""
        # Handle batched configurations
        if joint_config.dim() > 1:
            self.configuration_batch = joint_config
            joint_config = joint_config.view(-1, self.chain.num_joints)
        else:
            self.configuration_batch = None
            
        # Compute forward kinematics
        fk_matrices = self.chain.forward_kinematics(joint_config, end_only=False)
        
        # Extract and compose transforms for each link
        self.link_transforms = []
        for name in self.link_names:
            # Get transform from base to link
            T_base_to_link = fk_matrices[name]
            # Apply offset inverse and compose
            T_obj_to_link = torch.linalg.inv(self.offset_transforms[self.link_names.index(name)]) @ T_base_to_link
            self.link_transforms.append(T_obj_to_link)
            
    def __call__(self, points_in_object_frame, compute_grad=False):
        """Query the SDF for points in the object frame."""
        if self.configuration_batch is not None:
            # Handle batched queries
            # Expand link transforms to match batch size
            B = self.configuration_batch.shape[0]
            expanded_transforms = [T.unsqueeze(0).expand(B, -1, -1) for T in self.link_transforms]
        else:
            expanded_transforms = self.link_transforms
            
        # Initialize results
        min_distances = torch.full((points_in_object_frame.shape[0],), float('inf'), device=points_in_object_frame.device)
        gradients = None
        
        # Iterate over each link
        for i, link_sdf in enumerate(self.link_sdfs):
            T = expanded_transforms[i]
            
            # Transform points to link frame
            # points_link = T_inv @ points_object (homogeneous)
            # Simplified: assume T is 4x4, points are Nx3
            points_homogeneous = torch.cat([points_in_object_frame, torch.ones_like(points_in_object_frame[:, :1])], dim=1)
            points_link_homogeneous = (T.inverse() @ points_homogeneous.T).T
            points_link = points_link_homogeneous[:, :3]
            
            # Query link SDF
            dist, grad = link_sdf(points_link, compute_grad=True)
            
            # Update minimum distance
            mask = dist < min_distances
            min_distances[mask] = dist[mask]
            
            if compute_grad:
                # Transform gradient back to object frame
                # grad_obj = T @ grad_link
                grad_homogeneous = torch.cat([grad, torch.zeros_like(grad[:, :1])], dim=1)
                grad_obj_homogeneous = (T @ grad_homogeneous.T).T
                grad_obj = grad_obj_homogeneous[:, :3]
                
                if gradients is None:
                    gradients = torch.zeros_like(points_in_object_frame)
                gradients[mask] = grad_obj[mask]
                
        return min_distances, gradients

    def compile(self):
        """Compile CachedSDFs for speedup."""
        for link_sdf in self.link_sdfs:
            if isinstance(link_sdf, sdf.CachedSDF):
                link_sdf.compile()
```

**Parameters:**
*   `chain`: Robot description (pytorch_kinematics Chain). Non-mesh geometries are ignored [S1, unofficial].
*   `link_sdf_cls`: Factory for link SDFs. Default `sdf.MeshSDF`. Supports `CachedSDF` or `TSDF` for performance [S1, unofficial].
*   `compute_grad`: Boolean. If `False`, only SDF values are returned, improving performance [S1, unofficial].
*   `resolution`: Grid resolution for `CachedSDF`/`TSDF`. Default 0.02 [S1, unofficial].
*   `truncation_distance`: Truncation distance for `TSDF`. Default 0.1 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Points very close to the surface ($|d| < 10^{-3}$) may have noisy gradients due to closest-point estimation. The implementation fixes this by using surface normals for these points [S1, unofficial].
*   **Inside/Outside:** Determined by ray casting intersection counts. Points with odd intersections are inside [S1, unofficial].
*   **Batching:** Joint configurations must be reshaped to $(-1, M)$ if multiple batches are provided. Transform matrices must be expanded to match the batch dimension [S1, unofficial].

## Subtopics

### [TetraSDF Skeleton Generation](14203-tetrasdf-skeleton-generation.md)
Extracts topological skeletons from the SDF volume produced by RobotSDF. It differs by operating on the resulting grid data rather than the implicit representation. Choose when centerlines are needed for path planning.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$ per query, where $N$ is the number of query points and $L$ is the number of links. Each link SDF query is $O(1)$ for mesh-based SDFs (raycasting) or $O(\log V)$ for tree-based structures.
*   **Memory Complexity:** $O(L \cdot V)$, where $V$ is the voxel count if `CachedSDF` is used. `MeshSDF` stores only mesh data, which is typically smaller.
*   **Measured Performance:** Using `CachedSDF` with `torch.compile` provides approximately a 2x GPU speedup for no-gradient queries compared to default `MeshSDF` [S1, unofficial].
*   **Strengths:** Exact geometric representation for any joint configuration; supports batched queries; differentiable (if gradients are computed).
*   **Limitations:** Slower than pre-baked voxel grids for static scenes; requires forward kinematics computation for each configuration; memory usage scales with number of links if caching is enabled.
*   **When to Use:** Robot motion planning, collision detection in dynamic environments, inverse kinematics with geometric constraints.
*   **When Not to Use:** Static scenes where pre-computed SDFs are sufficient; real-time applications with strict latency constraints and many links where caching overhead is prohibitive.

## Background
RobotSDF was introduced as part of the `pytorch_volumetric` library by the UM-ARM-Lab [S1, unofficial]. It addresses the need for differentiable, geometry-accurate collision checking in robot learning and control tasks, where traditional voxel-based methods may lack precision or require frequent recomputation. The method leverages `pytorch_kinematics` for efficient forward kinematics and integrates with PyTorch for GPU acceleration [S1, unofficial].

## Sources
[S1] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
