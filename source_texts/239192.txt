# GrassTool
Allows add grass rendring on the scene. Grass renders via DrawMeshInstancedIndirect and uses GPU culling via compute shaders

## Drawing example
<img width="720" height="405" alt="Drawing" src="https://github.com/user-attachments/assets/6ef5b421-1776-4c9a-beae-ad839de75c1a" />


## Culling example
<img width="720" height="405" alt="Culling" src="https://github.com/user-attachments/assets/dca2b647-86f2-4415-92c3-c64831ef578c" />

## Content

Open Tools -> GrassEditor. It opens grass editor window:

<img width="206" height="82" alt="Location" src="https://github.com/user-attachments/assets/8da57183-6538-45b8-adb8-f85b369c916a" />

____

Grass editor window contains few foldouts:

<img width="261" height="213" alt="Foldouts" src="https://github.com/user-attachments/assets/6180b1f6-26ac-40a7-b633-6f37710ad2c6" />

____

**Draw Grass Settings** 

Contains parameters for drawing:

<img width="329" height="174" alt="DrawGrassSettings" src="https://github.com/user-attachments/assets/79ab688e-f9e7-459d-a13e-29bbe58abca5" />

**Editor mesh** field contains mesh, which will render in edit mode

____

**Grass data SO settings** 

Contains parameters for Scriptable Object, which stores positions and rest of parameters of every single grass. This SO uses for edit mode and baking into chunks

<img width="337" height="137" alt="GrassDataSo" src="https://github.com/user-attachments/assets/8c95dbc6-146c-43f6-b470-d9f5151b01f3" />

You can create new SO via setting path and name in this foldout, or via Create -> Grass Objects -> Grass SO data:

<img width="729" height="619" alt="createSOdata" src="https://github.com/user-attachments/assets/2c9be232-8dc7-40e7-ad94-f64d2fc03639" />


Every changes of drawn grass needs saving Grass data SO via pressing ***Save grass data***
____

**Texture atlassing settings** 

Contains settings for grass textures:

<img width="321" height="377" alt="Texture settings" src="https://github.com/user-attachments/assets/2587ee88-c580-4edf-8427-dbd901970c59" />

Grass Tool can render grass, using one or two (as atlas) different textures via single instance. Also, you can create your own atlas, by setting fields **Texture 1** and **Texture 2** and name for atlas

____

**Vertex color settings**

Allows to paint lower and upper vertices for grass mesh. UV coordinates uses as mask for detecting low/high position for vertex color

<img width="337" height="136" alt="VC" src="https://github.com/user-attachments/assets/d5495794-20bf-409e-b894-05ca1ff55f61" />

Don't forget to press ***Save grass data*** after editing vertex color

____

**Eraser settings**

Allows to erase grass

<img width="337" height="75" alt="Eraser" src="https://github.com/user-attachments/assets/37762b6f-9af1-4abd-8050-08403bfc9e50" />

Don't forget to press ***Save grass data*** after erasing

____

**Baker settings**

<img width="339" height="223" alt="Baker" src="https://github.com/user-attachments/assets/0d8a3bc5-a526-4ebf-b63c-dfc33206c242" />


For rendering grass in play mode, Grass Tool bakes grass data into chunks, transferred into GPU and processed via compute shader. When your grass is drawn, follow next steps:

+ Create **chunk data SO** (if it is not exist) by same way as **Grass data SO**
+ Set chunks density on horizontal and vertical axes (bounds will calculate automatically)
+ Press **Bake chunks**

____

For rendering grass in play mode, create empty Game Object and add Grass Drawer component:

<img width="416" height="367" alt="Drawer" src="https://github.com/user-attachments/assets/d81d0419-7e32-4c1d-9c52-2dad185c2c58" />

Fill all fields. 
+ For **Grass Mat** field create new material with shader ***Unlit/Instance Grass*** and assign grass texture
+ For **Culling Compute Shader** field use CullingComputeSHader in ***Assets/Shaders/*** folder
+ For **Cam** field use your active camera
