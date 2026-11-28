# 3d-engine
A special equation that converts any 3-D space coordinate to a 2-D perspective projection. I don't like linear algebra, so I made my own formula that converts a single point in 3D space to a 2D plane that is 1920x1080. I used the concept of parallax from astronomy where stars far away appear to move slow and close stars appear to move fast. I also included rotations in my equation, and it works, but that code had a lot of bugs. I don't have much time anymore to debug, so that's a later problem. The main goal of this project was for me to make my own way of representing 3D data on a 2D screen for partial differential equtions. <br />
You should try out the two programs in the "3d engine 2" folder called: "test7" and "3d23" (yes it took me 23 different save states to get it to work smh)<br />
"test7.py" is the 3D gravity simulation, and "3d23.py" is just a box.<br />
"convert3.py" is the file that I am now using that does the 3D to 2D conversion<br /><br />
Use arrows to move U/D/L/R; use W and S to move Fd/Bk





![Gravity Particles in 3D!](https://github.com/BryceP-44/3d-engine/blob/main/3d%20gif.gif)
<br /><br />
![A boring box I used for original testing](https://github.com/BryceP-44/3d-engine/blob/main/3d%20engine%202/3d%20box%20pic.png)
