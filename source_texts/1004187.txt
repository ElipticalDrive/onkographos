# VoxBotanica

**VoxBotanica** is a procedural voxel vegetation generator developed as part of an Honours Project at **Abertay University**. The project explores how algorithmic plant modelling techniques can be adapted to a voxel-based pipeline to efficiently produce structured, game-ready flora.

At its core, VoxBotanica combines a Lindenmayer System (L-system) with a custom voxel rendering framework. The L-system is used to generate hierarchical plant structures, which are then parsed into trunk and branch segments before being converted into voxel data. These voxels are subsequently transformed into optimised meshes, allowing the final assets to be rendered efficiently within Unity.

The system supports:
- Procedural tree generation using configurable L-system rules
- Separate control over trunk, branches, and canopy formation
- Voxel-based mesh construction with face culling and mesh merging
- Dynamic leaf generation using clustered voxel distributions
- Real-time parameter editing through a custom UI
- Export of generated assets to FBX format for external use

The goal of the project is to reduce the time and effort required to create voxel vegetation assets while maintaining visual consistency and structural coherence. VoxBotanica demonstrates how procedural techniques can be integrated into a practical tool for game development workflows, particularly within voxel-based or stylised environments.
