# Yet another try at re-creating Minecraft

I always loved Minecraft, and the way it works fascinates me. To me, it is the game you can be the most imaginative, using voxels as base blocks to create anything.

So naturally, I tried several times to do a minecraft clone to understand the inner workings of the game, my first tries were about the procedural generation, this one is about the back-end: mesh and chunk loading optimisations, block management and more.

I'm trying to make the fastest, most reliable, and most extensible voxel chunk system I can, so I can then be able to build a game on top of it.

## Features

- Chunk system, with loading, unloading, serializing and support for procedural generation
- Basic frustum culling of the chunks (only in 2D for the moment)
- Block descriptions manager, to manage the block textures in a kind of palette

## ToDo

- [ ] Project the 3D frustum onto the 2D plane to make it complete
- [x] Make the block management system load blocks from description files
- [ ] Change the textures from an atlas to an array of textures to avoid texture bleeding
- [ ] Better procedural generation (not the priority)
- [x] Block placing and destruction
- [ ] Collisions and player physics
- [ ] Change a vertex representation in GPU memory. Goal : from 8x32 bit floats to a single 32bit integer
- [ ] Add ImGui for debug
- [ ] Make an actual UI system
- [ ] Tick system (20 ticks per second)
- [ ] Make the chunk jobs queue update only when the player move a certain amount
- [ ] Make a ChunkDealer class, that don't free the chunks memory, but resets them and re-use them
