# RobotSDF
Also called: Robot Model to SDF
## Overview
RobotSDF generates a Signed Distance Field (SDF) representation of a multi-link robot model conditioned on a specific joint configuration [S1, unofficial]. It solves the problem of creating differentiable, batched geometric representations for robotic arms to enable gradient-based planning and collision checking [S1, unofficial]. It differs from static SDF methods by dynamically composing link-level SDFs via forward kinematics transforms, allowing the volume to change shape as joint angles vary [S1, unofficial].
## How it works
**Inputs:**
- `chain`: A `pytorch_kinematics.Chain` object describing the robot. Links must be of mesh type; non-mesh geometries are ignored [S1, unofficial].
- `default_joint_config`: Optional initial joint values (None defaults to zeros) [S1, unofficial].
- `path_prefix`: Directory path for resolving relative mesh file paths in the URDF [S1, unofficial].
- `link_sdf_cls`: Factory class for individual link SDFs (default: `sdf.MeshSDF`) [S1, unofficial].
- `points_in_object_frame`: Query points $\mathbf{P} \in \mathbb{R}^{N \times 3}$ in the robot's base frame [S1, unofficial].
- `compute_grad`: Boolean flag to enable/disable gradient computation [S1, unofficial].

**Outputs:**
- `distance`: SDF values $d(\mathbf{x})$ for each query point [S1, unofficial].
- `gradient`: Surface normals $\nabla d(\mathbf{x})$ if `compute_grad` is True [S1, unofficial].

**Data Structures:**
- `ComposedSDF`: Holds a list of link SDFs and their corresponding object-to-link transformation matrices [S1, unofficial].
- `MeshObjectFactory`: Manages the underlying mesh geometry and raycasting scene for a single link [S1, unofficial].

**Steps:**

1. **Initialization**:
   - Iterate over frame names in the `chain`.
   - For each visual geometry of type 'mesh', instantiate a `MeshObjectFactory` and create a link SDF using `link_sdf_cls` [S1, unofficial].
   - Store offsets and link names [S1, unofficial].
   - Stack offset transforms and create a `ComposedSDF` from the list of link SDFs and object-to-link frames [S1, unofficial].
   - If `link_sdf_cls` is `CachedSDF` or `TSDF`, call `s.compile()` on each link SDF to optimize the `_forward_no_grad` method using `torch.compile` [S1, unofficial].

2. **Set Joint Configuration**:
   - If `joint_config` has multiple batch dimensions, save them in `configuration_batch` and reshape to $(-1, M)$ where $M$ is the number of joints [S1, unofficial].
   - Compute forward kinematics using `SerialChain` with `end_only=False` (or default for other chain types) [S1, unofficial].
   - Extract transformation matrices for each link name [S1, unofficial].
   - Inverse offset transforms and compose with link transforms to get `object_to_link_frames` [S1, unofficial].
   - If `configuration_batch` exists, expand offset transform matrix dimensions and repeat to match batch size before reshaping [S1, unofficial].

3. **Query (SDF Evaluation)**:
   - Pass `points_in_object_frame` and `compute_grad` flag to the underlying `ComposedSDF` [S1, unofficial].
   - For each link, transform query points to the link's local frame using the current `object_to_link_frames` [S1, unofficial].
   - Evaluate the link's SDF at the transformed points [S1, unofficial].
   - Compose results from all links (typically taking the minimum distance for the union of solids) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import pytorch_kinematics as pk
from pytorch_volumetric import sdf

class RobotSDF:
    def __init__(self, chain: pk.Chain, default_joint_config=None, path_prefix='',
                 link_sdf_cls=sdf.MeshSDF):
        self.chain = chain
        self.link_sdfs = []
        self.object_to_link_frames = []
        
        # Initialization: Build link SDFs
        for frame_name in chain.frame_names:
            # Assuming chain provides access to visual geometries
            # Create MeshObjectFactory and link SDF
            # Store offsets and link names
            # link_sdf = link_sdf_cls(mesh_factory)
            # self.link_sdfs.append(link_sdf)
            pass 
        
        # Create ComposedSDF
        # self.composed_sdf = ComposedSDF(self.link_sdfs, self.object_to_link_frames)
        
        # Compile if using CachedSDF/TSDF
        # for s in self.link_sdfs:
        #     if isinstance(s, CachedSDF):
        #         s.compile()

    def set_joint_configuration(self, joint_config: torch.Tensor):
        """
        joint_config: shape (M,) or (B, M) where M is number of joints
        """
        # Handle batch dimensions
        if joint_config.dim() > 1:
            self.configuration_batch = joint_config
            joint_config = joint_config.reshape(-1, joint_config.shape[-1])
        else:
            joint_config = joint_config.unsqueeze(0)
            
        # Compute Forward Kinematics
        # fk_matrices = self.chain.forward_kinematics(joint_config, end_only=False)
        
        # Extract transformation matrices for each link
        # Inverse offset transforms and compose with link transforms
        # self.object_to_link_frames = ...
        
        # Handle batch expansion for offsets if needed
        # if self.configuration_batch is not None:
        #     expand and repeat offset transforms

    def __call__(self, points_in_object_frame: torch.Tensor, compute_grad: bool = False):
        """
        points_in_object_frame: shape (N, 3) or (B, N, 3)
        Returns: distance, gradient (optional)
        """
        # Pass to ComposedSDF
        # distance, gradient = self.composed_sdf(points_in_object_frame, self.object_to_link_frames, compute_grad)
        # return distance, gradient
        pass

# Example Usage
# chain = pk.build_serial_chain_from_urdf(urdf_string, root_link)
# robot_sdf = RobotSDF(chain, link_sdf_cls=sdf.MeshSDF)
# robot_sdf.set_joint_configuration(joint_angles)
# dist, grad = robot_sdf(query_points, compute_grad=True)
```

**Parameters:**
- `resolution`: Grid resolution for `CachedSDF`/`TSDF`. Default: 0.02 [S1, unofficial].
- `truncation_distance`: Truncation distance for `TSDF`. Default: 0.1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Non-mesh geometries in the URDF are ignored [S1, unofficial].
- Points very close to the surface ($|d| < 10^{-3}$) may have noisy gradients from closest-point raycasting; these are corrected using surface normals [S1, unofficial].
- Gradient direction is flipped for interior points to ensure $\nabla d$ points outward [S1, unofficial].
- Ray destinations for intersection tests are extended outside the bounding box with small random noise to reduce artifacts [S1, unofficial].

## Subtopics

### [TetraSDF Skeleton Generation](14203-tetrasdf-skeleton-generation.md)
Extracts topological skeletons from the SDF volume produced by RobotSDF. Choose this when centerlines are needed for path planning.

## Operating characteristics
- **Time Complexity**: Depends on the number of links $L$, query points $N$, and the underlying link SDF implementation. MeshSDF is slower due to per-point raycasting; CachedSDF/TSDF are faster due to grid lookups [S1, unofficial].
- **Memory Complexity**: Proportional to the number of links and the resolution of cached grids [S1, unofficial].
- **Performance**: Using `CachedSDF` or `TSDF` with `torch.compile()` provides approximately a 2x speedup on GPU for no-gradient queries [S1, unofficial].
- **Strengths**: Differentiable, batched, supports arbitrary joint configurations [S1, unofficial].
- **Limitations**: Only supports mesh geometries; performance degrades with high-resolution grids or many links if not cached [S1, unofficial].
## Background
RobotSDF is part of the `pytorch_volumetric` library developed by the University of Michigan ARM Lab [S1, unofficial]. It enables the use of implicit representations for robotic kinematics and collision detection in deep learning pipelines [S1, unofficial].
## Sources
[S1] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
