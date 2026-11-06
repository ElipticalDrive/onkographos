## First steps

```bash
sudo apt update
sudo apt install build-essential
sudo apt install libglfw3-dev
sudo apt install libglew-dev
sudo apt install cmake
```

## Create a Project structure

```bash
mkdir voxel_game
cd voxel_game
mkdir src
```

## Create CMakeList.txt file:

```bash
cmake_minimum_required(VERSION 3.10)
project(VoxelGame)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find required packages
find_package(OpenGL REQUIRED)
find_package(GLEW REQUIRED)
find_package(glfw3 REQUIRED)

# Add executable
add_executable(VoxelGame src/main.cpp)

# Link libraries
target_link_libraries(VoxelGame
    OpenGL::GL
    GLEW::GLEW
    glfw
)

```

## Save the provided game code in src/main.cpp

## Build and run:

```bash

mkdir build
cd build
cmake ..
make
./VoxelGame

```
