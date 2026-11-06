# ros_pkg_angles_conversion
services to convert from quaternion to Eular angles and Rodrigues representation. Besides, from  Rotation Matrix to Quaternions.

## An example of rotation matrix to quaterion where the matrix is:

```
[1 0       0     ]
[0 0.9848 -0.1736]
[0 0.1736   0.9848] 
```
Which represent a rotation of 10 degrees in x-axis.

![Screenshot from 2022-04-17 17-51-51](https://user-images.githubusercontent.com/15234283/163733342-87c3c0c5-5437-475d-9139-9caedf7a7268.png)


## Two examples of quaternion to zyx Euler representation where the quaternion is:

```
q = [0.0871 0 0 0.9961] (qx, qy, qz, qw)
```
Which represent a rotation of 10 degrees in x-axis.

![Screenshot from 2022-04-17 17-57-12](https://user-images.githubusercontent.com/15234283/163733545-76b1be2c-1fc3-4f30-aac0-406c6e242300.png)

```
q = [0.0075 0-0868 0.0868 0.9924] (qx, qy, qz, qw)
```
Which represent a rotation of 10 degrees in y-axis and then a 10 degrees in z-axis.

![Screenshot from 2022-04-17 18-02-47](https://user-images.githubusercontent.com/15234283/163733708-5b395a77-5ba0-490e-a962-d997b868eeb0.png)


## One example of quaternion to Rodrigues representation where the quaternion is:

```
q = [0.0075 0-0868 0.0868 0.9924] (qx, qy, qz, qw)
```
Which represent a rotation of 10 degrees in y-axis and then a 10 degrees in z-axis.

![Screenshot from 2022-04-17 18-05-58](https://user-images.githubusercontent.com/15234283/163733904-fb7c0fed-37df-42d3-a493-ad1fdc6ba7c3.png)
