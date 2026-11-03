## SofaDistanceGrid

The plugin's core is the DistanceGrid structure, a regular 3D grid storing signed distances to an object's surface, loaded from files (e.g., .fmesh via the bundled miniFlowVR library) or computed from meshes.

It also provides two collision models: RigidDistanceGridCollisionModel for rigid bodies and FFDDistanceGridCollisionModel for deformable objects, where the grid is deformed through free-form deformation. Dedicated intersection components (RigidDistanceGridDiscreteIntersection, FFDDistanceGridDiscreteIntersection, ray-based contacts) plug these models into SOFA's collision pipeline with penalty-based contact response.

It also ships a DistanceGridForceField that pushes particles out of (or attracts them to) the implicit surface directly, without the full collision pipeline.

A CUDA extension accelerates the grid computations, and example scenes (liver demos with rigid/FFD variants and both animation loops) show typical usage.
