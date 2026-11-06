# CPU only software rasterizer
3D textured rasterizer using the CPU only.
## Usage instructions
To use this you must have SDL2 installed such that the program can access `SDL2/SDL.h`.
#### Compile with g++
```
git clone https://github.com/keanu-thakalath/software-rasterizer.git
cd software-rasterizer
g++ main.cpp 3DG.cpp Math.cpp RenderEngine.cpp lodepng.cpp -lSDL2 -lSDL2main -std=c++17 -Ofast -o app.exe
./app.exe mp5\mp5
```
#### File format
The program finds the files to render by appending `.obj` and `.mtl` to the file provided. If no `mtl` file is found the material will be a shiny gold color.
#### Blender
Compatible with Blender, make sure to export as an `obj` file with `Triangulate faces` checked.
##### My objects don't look like they do in Blender?
Blender uses a right handed coordinate system, whereas this program uses a left handed system. What this means is that if you look down the positive z axis in Blender the positive x axis would be on your left and the positive y axis would be above you. However when you look down the positive z axis in this program the positive x axis is on your right and the positive y axis would be above you.
These coordinate systems are arbitrary but it has the consequence that objects in Blender have their x value negated.
