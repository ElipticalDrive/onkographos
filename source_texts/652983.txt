# Controls:
- Press Arrow keys to orbit camera around origin.
- Press T to go into translation mode: click and drag to translate.
- Press R to go into rotation mode as well as to switch rotational axis : click and drag to rotate.
- Press S to go into scale mode: click and drag to scale.
- Press L to reset model as well as load second model.
- Press K to remove duplicate model.
- Press C to turn on and off changing light colours.
- Press Z to make lightsource 1 increase it's position in the y axis
- Press X to make lightsource 1 decrease it's position in the y axis
- Press V to make lightsource 1 decrease it's position in the x axis
- Press B to make lightsource 1 increase it's position in the x axis
- Press U to make lightsource 2 increase it's position in the y axis
- Press I to make lightsource 2 decrease it's position in the y axis
- Press O to make lightsource 2 decrease it's position in the x axis
- Press P to make lightsource 2 increase it's position in the x axis
- Press A to make lights rotate around object.
- Press Q to turn off world axis rotation and reset position to center. This allows to rotate the object on the spot anywhere.
# Compilation
Compilation on linux:
1. Make sure the objects folder is inside the build folder as well as all the texture files.
2. Make sure the shaders: "simple.vert" and "simple.frag" are in the build folder.
3. Make sure all libraries required are installed namely libsdl2-dev and libglew-dev.
4. To install these packages type sudo apt-get install <package name> in a terminal window and hit enter. 
5. Once the packages are installed, open a terminal window and navigate to the project's root directory. 
6. Use "make" to build your project and "make run" to run it. You should now see a window with the suzanne model in it.
7. Use controls above to manipulate the obj with the mouse. The default setting doesn't allow manipulation so click a button before trying to do anything
8. Mouse controls are drag and drop.
