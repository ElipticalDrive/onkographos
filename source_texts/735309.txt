## 📋 Overview

This project is a simplified recreation of the classic game Wolfenstein 3D, implemented in C. The game uses a raycasting algorithm to render a 3D maze from a 2D map. The player navigates the maze from a first-person perspective, experiencing a retro-style 3D environment. The display is handled using the MiniLibX (MiniLibX Graphics Library).

The focus of this project is to explore foundational game development techniques and computer graphics concepts, such as:

* Raycasting
  
* Grid-based movement
  
* Handling player input
  
* Rendering 2D maps into 3D spaces

<p align="center">
<img src="https://github.com/user-attachments/assets/8e70713a-f783-40ec-9ce2-808b7ad3b5ee" alt="Screenshot3" width="300"/>
<img src="https://github.com/user-attachments/assets/c8264cf0-3b79-47ae-a3f3-e6ac32ac0019" alt="Screenshot1" width="300"/>
<img src="https://github.com/user-attachments/assets/63d83588-4a65-43d0-a7d6-6cc8f623fc89"  alt="Screenshot2" width="300"/>
</p>

## 🖥️ Technologies Used

**Language**: C

**Graphics Library**: MiniLibX (mlx)

**Platform**: Linux


## 📐 How Raycasting Works

Raycasting is a rendering technique that simulates a 3D perspective by casting rays from the player's position into the game world to determine what walls and objects should be displayed on the screen.

Here's a brief breakdown of the process:

* **Raycasting Loop**: The player’s field of view is divided into vertical strips, and rays are cast to detect walls in the maze.

* **Distance Calculation**: Using trigonometry, the distance between the player and the walls is calculated to determine how tall the walls should appear.

* **Wall Rendering**: Based on the distance, walls are drawn as vertical strips, creating the illusion of depth.


## 🚀 Getting Started

**Prerequisites**

* Linux

* MiniLibX library (included in the project)

**Building the Project**

Clone the repository and run the make command to compile the project:

```
git clone https://github.com/lotte-detaeye/cub3D.git
make
```

**Running the Game**

After building the project, launch the game with:

```
./cub3d <map_file>
```
Example:

```
./cub3d maps/test1.cub
```


## 🎮 Controls

| Keys | Actions |
| ------------|-------------| 
| **W** | Move forward | 
| **S** | Move backward | 
| **A** | Move left | 
| **D** | Move right | 
| **Left arrow** | Turn left | 
| **Right arrow** | Turn right | 
| **ESC** | Quit the game| 


## 📋 Map Files

Map files are simple text files that define the layout of the maze. They use the following format:

| Symbol | Map feature |
| ------------|-------------| 
| **0** | Empty Space| 
| **1** | Wall| 
| **O, N, S or W** | Player starting position and viewing direction | 

Example of a basic map:

```
111111
100001
100W01
100001
111111
```
The .cub files in the map folder also include four paths to the textures for the walls, and the color codes for the ceiling and floor.


## 🖼️ MiniLibX Library

The MiniLibX (mlx) is a lightweight graphics library used to handle window creation, drawing, and event handling in the game. It's compatible with Linux.



## ⚙️ Extras

**Included in the project:**

* Textured walls
  
* A moving minimap
  
* Wall collision detection

**Some potential features to add in future updates:**

* Doors that can open to other rooms

* Sprites (for collectibles, enemies etc.)
  

## Acknowledgements

Shoutout to @mcombeau for making available a very nice collection of maps and textures

And thanks to @MacMosby for being a chill and meticulous partner at the same time
