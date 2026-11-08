# DualNumber_Quaternions
This project exposes two classes of mathematic sets written in Python (Dual Numbers and Quaternions). Also, those two classes can interac lo create Dual Quaternions and perform Automatic Differentiation.

The class of Quaternions is able to perform the basic arithmetic operations and even to rise power to a real number using the De Moivre's theorem as is shown in the following picture.

![imagen](https://github.com/LuisAntonio1929/DualNumber_Quaternions/assets/83978263/96e0f956-76a7-49ca-b8b7-b92cefabbc36)

On the other hand, the user is free to these two classes to create Dual Quaternions whether creating dual numbers composed by quaternions (as is usually perceived) or creating quaternions composed by dual numbers; the result of the basic arithmetic operations for both conceptions is always the same, with the exeption of the power operation which only is correctly performed in quaternions composed by dual numbers as is shown in the following picture.

![imagen](https://github.com/LuisAntonio1929/DualNumber_Quaternions/assets/83978263/6aaa1fd3-ffae-4cc6-8427-e34cb25afe1a)

Also, the Dual Number class can be used to perform automatic differentiation. However, if there is the need to compute the gradient of a multivariable function, dual numbers with a vector dual part, as is shown in the following image, can be used to perform partial derivatives.

![imagen](https://github.com/LuisAntonio1929/DualNumber_Quaternions/assets/83978263/25851f42-5fa7-476e-901c-7825e909c60f)
