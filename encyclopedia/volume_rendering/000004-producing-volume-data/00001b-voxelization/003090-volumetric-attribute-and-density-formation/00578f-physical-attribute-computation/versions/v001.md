# Physical Attribute Computation

Also called: Volumetric Attribute Computation, Secondary Attribute Generation

## Overview
Physical Attribute Computation generates secondary volumetric fields (such as velocity, size, or distance) from primary data (intensity, geometry, or segmentation) to enhance Direct Volume Rendering (DVR) or fluid simulation. It differs from siblings like [Interior Voxelize](995-interior-voxelize.md) or [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md) by computing derived physical properties rather than raw density or occupancy, and from [Multi-pattern Volumetric Construction via Voxel Multiplication](8874-multi-pattern-volumetric-construction-via-voxel-multiplicati.md) by focusing on attribute derivation rather than pattern composition.

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color, velocity) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Velocity Voxelization](3082-velocity-voxelization.md)
Computes the instantaneous velocity of an obstacle at each grid cell containing part of the obstacle's boundary, producing a volumetric velocity field for fluid simulation. It differs from standard density voxelization by explicitly interpolating vertex velocities across triangle-slice intersections and rasterizing "thickened" segments (quads) to populate the velocity field, rather than merely counting coverage or depth. It is best suited for watertight closed meshes where front and back faces are distinct. See [Velocity Voxelization](3082-velocity-voxelization.md).

### [Size Volume Computation](9487-size-volume-computation.md)
Generates a tertiary volumetric attribute representing the characteristic size of local structures, used to drive transfer functions in Direct Volume Rendering (DVR). It addresses the difficulty of defining mathematical descriptors for size variations in multiscale fields. Unlike siblings that compute scalar intensity or velocity, this method categorizes objects by their spatial extent, allowing users to distinguish features based on scale rather than just value. See [Size Volume Computation](9487-size-volume-computation.md).

### Distance Attribute
#### definition
A volumetric attribute not directly derived from intensity, but calculated on pre-segmented parts of a volume. It identifies regions close to a location of interest [S1, unofficial].
#### usage
Used to delimit spherical neighborhoods, such as layers adjacent to a tumor in a vascular dataset [S1, unofficial].

## Background
The concept of using attributes beyond intensity for visualization was explored in the context of transfer functions. Tappenbeck, Preim, and Dicken (2006) introduced distance as a volumetric attribute calculated on pre-segmented parts to identify regions near a location of interest, such as delimiting spherical neighborhoods around a tumor [S1, unofficial]. Correa and Ma (2008) adapted the concept of "size" to Direct Volume Rendering, noting the difficulty in defining mathematical descriptors for size variations in multiscale fields, similar to challenges in Computer Vision .

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] Correa, T., & Ma, K.-L. (2008). Size Volume Computation. (Unofficial notes)
