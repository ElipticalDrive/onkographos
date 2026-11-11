# Camera Trajectory and Path Generation

Also called: Camera Path Generation, Trajectory Generation

## Overview
This family generates sequences of camera poses (origins and orientations) for volume rendering, novel view synthesis, and 3D reconstruction. It solves the problem of defining valid viewpoints that cover a scene or structure, ensuring continuous motion and appropriate sampling density. It differs from static camera models by producing a time-varying or parameterized set of extrinsics, and from simple orbit generators by supporting complex geometries (spirals, linear elongations) and data-driven constraints (point cloud density).

## Shared principles
All methods in this family output a sequence of camera poses, typically represented as $4 \times 4$ transformation matrices (camera-to-world or world-to-camera) or explicit position/target/up tuples. The core mechanism involves:
1.  **Scene Analysis**: Determining the spatial extent, centroid, or principal axes of the target scene or structure using input data (training poses or point clouds) [S1, S2, S3, S5, unofficial].
2.  **Path Parameterization**: Defining a trajectory in 3D space using parameters such as time $t \in [0, 1]$, rotation angles, or linear interpolation along principal axes [S1, S2, S3, S5, unofficial].
3.  **Pose Construction**: For each point on the path, computing the camera center $\mathbf{c}$, the viewing direction $\mathbf{z}$ (towards a target or centroid), and the up vector $\mathbf{u}$ to form an orthonormal basis [S1, S2, S3, S5, unofficial].
4.  **Validation**: Filtering or adjusting poses to ensure they remain within valid bounds, avoid singularities, or satisfy density constraints [S1, G].

## Subtopics

### [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md)
Creates a helical sequence of camera poses around a scene centroid. It adds vertical oscillation to a horizontal orbit, suitable for 360-degree novel view synthesis. Choose this for scenes with rotational symmetry where vertical variation is desired. Differs from circular paths by its sinusoidal vertical component.

### [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md)
Generates multi-view poses for large-scale linear structures (e.g., walls) using PCA to align with the principal axis. It samples stations along the axis and validates against a density grid. Choose this for outdoor, elongated scenes. Differs from spiral paths by assuming linear geometry and placing cameras on parallel tracks.

## Background
Originating from the needs of NeRF (Neural Radiance Fields) and 3D Gaussian Splatting novel view synthesis, these methods automate the creation of training or rendering datasets. Early implementations used fixed circular orbits [S2, S3]. Spiral paths were introduced to provide better vertical coverage for indoor and outdoor scenes [S1, S5]. Data-driven trajectories for elongated structures emerged to handle large-scale outdoor assets where scene-centric assumptions fail [S1].

## Sources
