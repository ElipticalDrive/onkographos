# Note

All the contents of this folder is me learning how to use AI coding tools, I didn't write a single line myself. Though dont get me wrong, I didn't just tell it to make a whole file, I told it what functions to create what what they do and what to watch out for.

# Beta-1.0

This folder is full of prototypes that was thrown togethor in the time of a week or 2. Holding a weak visualEngine with who know how many bugs, and 2 versions of prototypes of a 3D modeling software.

# Beta-2.0

This folder contains:

- A more complex yet still messy VisualEngine--quite an upgrade of the prototype--created in 40 days, designed specifically for GPU shaders, memory, and OpenGL management.
- A actual version of a 3D modeling software made for voxalBased games, it has the flexibility to create any shape within a square yet the placement of the models are purely designed for cordinate placements. It exports through GLB (texture + vertex mix so that it reduces the needed UV texture sampling calls) and uses greedymeshing for cubes faces and decently complex face culling. 
- A PerlinNoiseVisualizer which is a prototype of world generation, mostly testing around with Perlin generation, CPU thread management and piplining through the GPU. This version runs at 4000 16^3-chunks per second.
