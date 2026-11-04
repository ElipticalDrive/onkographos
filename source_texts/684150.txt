# Baked vs Realtime Lighting
Baked lighting and shadows with APV versus realtime in Unity 6.3
<br><br>

![Project Image](<./.gitimages/Baked vs Realtime Lighting 01.png>)
<br><br>

![Project Image](<./.gitimages/Baked vs Realtime Lighting 02.png>)
<br><br>

## Project Description
This is an example focusing on baked/mixed lighting and shadows versus realtime. The main scene uses baked lightmaps by default, in conjunction with mixed lighting in Shadowmask mode. Being able to activate and deactivate the lightmaps during runtime helps illustrate how well quality lightmaps can benefit a scene. In addition, this project allows for the user to adjust the shadow strength in realtime or turn them off completely.
<br>

In conjunction with mixed lighting, an Adaptive Probe Volume is applied to blend the environment's shadows upon the player between dark and light areas. This technique replaces the traditional light probe grouping system.
<br>

Note: Some assets from the Unity Asset Store are required for a complete build. They are not included in this repo. Please see the Licenses section below for links to the assets. Otherwise, feel free to browse through the project files. Thanks.
<br><br>

## Player Controls
Move Player: WASD Keys -or- Arrows
<br>
Jump: Spacebar
<br><br>

## Project Features
* Enable and disable baked lightmaps during runtime
* Adjustable shadows using mixed lighting in Shadowmask mode
* Adaptive Probe Volume in place of tradition Light Probe Groups
* Framerate counter
<br><br>

## Licenses
[MIT](./LICENSE)
<br>
