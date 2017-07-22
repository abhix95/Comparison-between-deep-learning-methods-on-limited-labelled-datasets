In this post I will explain my setup for studying effects( !!!spoiler!!! possible benefits) of using pre-initialized weights in a Convolutional Neural Network (CNN). 

I have tried to study the difference by comparing results of a model that started training from random weights with results of the same model that started training from weights given by the convolutional autoencoder. I will be using MNIST to study the difference.

For this post, my library of choice is Keras (1.?.?) with Theano serving in its backend. I have used my laptop whuch is equipped with a Nvidia Quadro M1000M for this project. Running the same without a GPU will take a noticably long time although being MNIST it won't be too much.

So let's begin.

#### Before the code

Let us understand the basics of Convolutional Neural Networks(CNN) and Convolutional Auto-Encoders(CAE).

##### 1> CNN
> To understand CNN use this [post](ashirgao.github.io/CNN-basics/) of mine as reference.

##### 2> CAE
> As for CAEs

>![Convolutional Autoencoder][CAE]

[CAE]: https://github.com/ashirgao/Comparison-between-deep-learning-methods-on-limited-labelled-datasets/blob/gh-pages/CAEandCNN_files/CAE.jpg?raw=true

> CAE are made of 2 halves:
> 1> The encoder
> 2> The decoder

> The encoder is made of convolution and downsampling layers while the decoder is made of deconvolution(convolution) and upsampling layers. The encoder is resposible to encode the image using spatial features into an intermediate representation which holds information of the whole image. The decoder accepts this intermediate representation and tries to regenerate the orignal image. Loss for a CAE is the euclidean distance between respective pixels of the input image and decoded image. As the CAE is trained, we can say that the intermediate representation is an embedding of that image. We can also say that the convolutions in the encoder part of the CAE have become increasingly tuned to extract features from images that were used for the training. 
>This is exactly what we need. If we initialize the CNN with the weights of this encoder, from intution we can infer that the CNN will start looking at more relevant features in image that the convolutions with random initializations. Thus,
>
>>if we have lots of unlabelled data and limited labelled data, we can use the unlabelled data to train a CAE. Then, we can use the weights from the encoder to initialize weights of CNN. Then we train the CNN on the labelled data.
>
>In this post we will compare the model mentioned right above with a randomly initialized CNN trained only with the labelled data available.

![Approaches][Apps]

[Apps]: https://github.com/ashirgao/Comparison-between-deep-learning-methods-on-limited-labelled-datasets/blob/gh-pages/CAEandCNN_files/approaches.jpg?raw=true

#### Import libraries

For (model structure) visualization purpose, I have used [model_to_dot](https://keras.io/visualization/). This requires ```pydot``` and ```graphviz``` which are not installed automatically with keras. Also you might need to install ```h5py``` which is needed to save model weights. 

```shell
$ pip install h5py pydot graphviz
```
Now that we have the environment setup, let's start.

```python
from keras.layers import Input, Dense, Conv2D, MaxPooling2D, UpSampling2D, Activation, Flatten, Dropout
from keras.models import Sequential
from keras import backend as K
from keras.models import Model
from keras.utils import np_utils

from keras.datasets import mnist
import numpy as np

import matplotlib.pyplot as plt
from IPython.display import SVG
from keras.utils.visualize_util import model_to_dot

%matplotlib inline
```

    Using Theano backend.
    


#### Load data

If only loading data was so simple for project every time.....
.
.
Don't worry you'll used to it after a while.

I have appended the training and test set to get ```x``` and ```y``` ie. all of my labelled data.

```python
(x_train, y_train), (x_test, y_test) = mnist.load_data()
x_train = x_train.astype('float32') / 255.
x_test = x_test.astype('float32') / 255.

x = np.append(x_train,x_test,axis=0)
y = np.append(y_train,y_test,axis=0)
print"x shape : ", x.shape
print"y shape : ", y.shape

count = 0
```

    x shape :  (70000, 28, 28)
    y shape :  (70000,)


#### Define variables

I plan to vary the amount of data I appropriate for my CNN and CAE.I have added this block for easy manipulation of these variables so as to try and find the ratio of labelled to unlabelled data when benefits of using this method are maximum.

```python
epochs_for_cae = 40
epochs_for_cnn = 20

training_samples_for_cae   = 25000
validation_samples_for_cae = 5000

training_samples_for_cnn   = 25000
testing_samples_for_cnn = 10000
validation_samples_for_cnn = 5000

sum = training_samples_for_cae+training_samples_for_cnn+validation_samples_for_cae+validation_samples_for_cnn+testing_samples_for_cnn

if(sum>70000):
    print("***********\nOnly 70,000 data points, re-define variables such that their sum is less than 70,000.\n***********")
```

#### Autoencoder data

As MNIST data is small, I can easily fit the whole data in memory(RAM). In case of larger datasets use generators to load batch online( CPU loads and preprocesses a batch at a time and GPU trains on it). 
```python
cae_x_train = np.reshape(x[count:count+training_samples_for_cae], (training_samples_for_cae, 1, 28, 28))
count = count + training_samples_for_cae

cae_x_validation = np.reshape(x[count:count+validation_samples_for_cae], (validation_samples_for_cae, 1, 28, 28))
count = count + validation_samples_for_cae

print "cae_x_train shape : ", cae_x_train.shape
print "cae_x_validation shape : ", cae_x_validation.shape
```

    cae_x_train shape :  (25000, 1, 28, 28)
    cae_x_validation shape :  (5000, 1, 28, 28)


#### CAE model

The CNN uses weights from the encoder part of the CAE. Thus the model architecture for them must be same. In keras, it is necessary for the layers of CAE encoder and CNN model to have the same name. 

```python
print "Output shapes after subsequent layers : "

cae = Sequential()

cae.add(Conv2D(8, 3, 3, border_mode='same',  input_shape=(1, 28, 28),name='conv1' ))
cae.add(Activation('relu'))
cae.add(MaxPooling2D(pool_size=(2, 2), border_mode='same'))
print(cae.layers[-1].output_shape)

cae.add(Conv2D(16, 3, 3, border_mode='same', name='conv2'))
cae.add(Activation('relu'))
cae.add(MaxPooling2D(pool_size=(2, 2), border_mode='same'))
print(cae.layers[-1].output_shape)

cae.add(Conv2D(8, 3, 3 , border_mode='same', name='conv3'))
cae.add(Activation('relu'))
cae.add(MaxPooling2D(pool_size=(2, 2), border_mode='same', name='encoded'))
print(cae.layers[-1].output_shape)

# at this point the representation is (4, 4, 8) i.e. 128-dimensional

cae.add(Conv2D(8, 3, 3, activation='relu', border_mode='same'))
cae.add(UpSampling2D(size=(2, 2)))
print(cae.layers[-1].output_shape)

cae.add(Conv2D(16, 3, 3, activation='relu', border_mode='same'))
cae.add(UpSampling2D(size=(2, 2)))
print(cae.layers[-1].output_shape)

cae.add(Conv2D(8, 3, 3, activation='relu'))
cae.add(UpSampling2D(size=(2, 2)))
print(cae.layers[-1].output_shape)

cae.add(Conv2D(1, 3, 3, activation='sigmoid', border_mode='same', name='decoded'))
print(cae.layers[-1].output_shape)

cae.compile(optimizer='adadelta', loss='binary_crossentropy')

```

    Output shapes after subsequent layers : 
    (None, 8, 14, 14)
    (None, 16, 7, 7)
    (None, 8, 4, 4)
    (None, 8, 8, 8)
    (None, 16, 16, 16)
    (None, 8, 28, 28)
    (None, 1, 28, 28)


#### Model summary

As CNNs share weights over many pixels, the total number of parameters is less than a deep neural network made of onle ```Dense``` layers. This makes it easier to train CNNs.

```python
cae.summary()
```

    ____________________________________________________________________________________________________
    Layer (type)                     Output Shape          Param #     Connected to                     
    ====================================================================================================
    conv1 (Convolution2D)            (None, 8, 28, 28)     80          convolution2d_input_1[0][0]      
    ____________________________________________________________________________________________________
    activation_1 (Activation)        (None, 8, 28, 28)     0           conv1[0][0]                      
    ____________________________________________________________________________________________________
    maxpooling2d_1 (MaxPooling2D)    (None, 8, 14, 14)     0           activation_1[0][0]               
    ____________________________________________________________________________________________________
    conv2 (Convolution2D)            (None, 16, 14, 14)    1168        maxpooling2d_1[0][0]             
    ____________________________________________________________________________________________________
    activation_2 (Activation)        (None, 16, 14, 14)    0           conv2[0][0]                      
    ____________________________________________________________________________________________________
    maxpooling2d_2 (MaxPooling2D)    (None, 16, 7, 7)      0           activation_2[0][0]               
    ____________________________________________________________________________________________________
    conv3 (Convolution2D)            (None, 8, 7, 7)       1160        maxpooling2d_2[0][0]             
    ____________________________________________________________________________________________________
    activation_3 (Activation)        (None, 8, 7, 7)       0           conv3[0][0]                      
    ____________________________________________________________________________________________________
    encoded (MaxPooling2D)           (None, 8, 4, 4)       0           activation_3[0][0]               
    ____________________________________________________________________________________________________
    convolution2d_1 (Convolution2D)  (None, 8, 4, 4)       584         encoded[0][0]                    
    ____________________________________________________________________________________________________
    upsampling2d_1 (UpSampling2D)    (None, 8, 8, 8)       0           convolution2d_1[0][0]            
    ____________________________________________________________________________________________________
    convolution2d_2 (Convolution2D)  (None, 16, 8, 8)      1168        upsampling2d_1[0][0]             
    ____________________________________________________________________________________________________
    upsampling2d_2 (UpSampling2D)    (None, 16, 16, 16)    0           convolution2d_2[0][0]            
    ____________________________________________________________________________________________________
    convolution2d_3 (Convolution2D)  (None, 8, 14, 14)     1160        upsampling2d_2[0][0]             
    ____________________________________________________________________________________________________
    upsampling2d_3 (UpSampling2D)    (None, 8, 28, 28)     0           convolution2d_3[0][0]            
    ____________________________________________________________________________________________________
    decoded (Convolution2D)          (None, 1, 28, 28)     73          upsampling2d_3[0][0]             
    ====================================================================================================
    Total params: 5,393
    Trainable params: 5,393
    Non-trainable params: 0
    ____________________________________________________________________________________________________


#### Diagram

Visual representation of model. Using tensorflow as the backend opens access to TensorBoard which has much fancy visuals.

```python
SVG(model_to_dot(cae).create(prog='dot', format='svg'))
```




![svg](data:image/svg+xml;base64,PHN2ZyBoZWlnaHQ9IjEyMTNwdCIgdmlld0JveD0iMC4wMCAwLjAwIDIxOC4wMCAxMjEzLjAwIiB3aWR0aD0iMjE4cHQiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgeG1sbnM6eGxpbms9Imh0dHA6Ly93d3cudzMub3JnLzE5OTkveGxpbmsiPgo8ZyBjbGFzcz0iZ3JhcGgiIGlkPSJncmFwaDAiIHRyYW5zZm9ybT0ic2NhbGUoMSAxKSByb3RhdGUoMCkgdHJhbnNsYXRlKDQgMTIwOSkiPgo8dGl0bGU+RzwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9IndoaXRlIiBwb2ludHM9Ii00LDQgLTQsLTEyMDkgMjE0LC0xMjA5IDIxNCw0IC00LDQiIHN0cm9rZT0ibm9uZSIvPgo8IS0tIDEzOTk4MTYzMjc5NjMwNCAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlMSI+PHRpdGxlPjEzOTk4MTYzMjc5NjMwNDwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iMCwtMTE2OC41IDAsLTEyMDQuNSAyMTAsLTEyMDQuNSAyMTAsLTExNjguNSAwLC0xMTY4LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii0xMTgyLjgiPmNvbnZvbHV0aW9uMmRfaW5wdXRfMTogSW5wdXRMYXllcjwvdGV4dD4KPC9nPgo8IS0tIDEzOTk4MTYzMjc5NjA0OCAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlMiI+PHRpdGxlPjEzOTk4MTYzMjc5NjA0ODwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iMzQuNSwtMTA5NS41IDM0LjUsLTExMzEuNSAxNzUuNSwtMTEzMS41IDE3NS41LC0xMDk1LjUgMzQuNSwtMTA5NS41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItMTEwOS44Ij5jb252MTogQ29udm9sdXRpb24yRDwvdGV4dD4KPC9nPgo8IS0tIDEzOTk4MTYzMjc5NjMwNCYjNDU7Jmd0OzEzOTk4MTYzMjc5NjA0OCAtLT4KPGcgY2xhc3M9ImVkZ2UiIGlkPSJlZGdlMSI+PHRpdGxlPjEzOTk4MTYzMjc5NjMwNC0mZ3Q7MTM5OTgxNjMyNzk2MDQ4PC90aXRsZT4KPHBhdGggZD0iTTEwNSwtMTE2OC4zMUMxMDUsLTExNjAuMjkgMTA1LC0xMTUwLjU1IDEwNSwtMTE0MS41NyIgZmlsbD0ibm9uZSIgc3Ryb2tlPSJibGFjayIvPgo8cG9seWdvbiBmaWxsPSJibGFjayIgcG9pbnRzPSIxMDguNSwtMTE0MS41MyAxMDUsLTExMzEuNTMgMTAxLjUsLTExNDEuNTMgMTA4LjUsLTExNDEuNTMiIHN0cm9rZT0iYmxhY2siLz4KPC9nPgo8IS0tIDEzOTk4MTYzMjc5NjExMiAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlMyI+PHRpdGxlPjEzOTk4MTYzMjc5NjExMjwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iMzEsLTEwMjIuNSAzMSwtMTA1OC41IDE3OSwtMTA1OC41IDE3OSwtMTAyMi41IDMxLC0xMDIyLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii0xMDM2LjgiPmFjdGl2YXRpb25fMTogQWN0aXZhdGlvbjwvdGV4dD4KPC9nPgo8IS0tIDEzOTk4MTYzMjc5NjA0OCYjNDU7Jmd0OzEzOTk4MTYzMjc5NjExMiAtLT4KPGcgY2xhc3M9ImVkZ2UiIGlkPSJlZGdlMiI+PHRpdGxlPjEzOTk4MTYzMjc5NjA0OC0mZ3Q7MTM5OTgxNjMyNzk2MTEyPC90aXRsZT4KPHBhdGggZD0iTTEwNSwtMTA5NS4zMUMxMDUsLTEwODcuMjkgMTA1LC0xMDc3LjU1IDEwNSwtMTA2OC41NyIgZmlsbD0ibm9uZSIgc3Ryb2tlPSJibGFjayIvPgo8cG9seWdvbiBmaWxsPSJibGFjayIgcG9pbnRzPSIxMDguNSwtMTA2OC41MyAxMDUsLTEwNTguNTMgMTAxLjUsLTEwNjguNTMgMTA4LjUsLTEwNjguNTMiIHN0cm9rZT0iYmxhY2siLz4KPC9nPgo8IS0tIDEzOTk4MTYzMjMzODM4NCAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlNCI+PHRpdGxlPjEzOTk4MTYzMjMzODM4NDwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iNSwtOTQ5LjUgNSwtOTg1LjUgMjA1LC05ODUuNSAyMDUsLTk0OS41IDUsLTk0OS41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItOTYzLjgiPm1heHBvb2xpbmcyZF8xOiBNYXhQb29saW5nMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk5ODE2MzI3OTYxMTImIzQ1OyZndDsxMzk5ODE2MzIzMzgzODQgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTMiPjx0aXRsZT4xMzk5ODE2MzI3OTYxMTItJmd0OzEzOTk4MTYzMjMzODM4NDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTEwMjIuMzFDMTA1LC0xMDE0LjI5IDEwNSwtMTAwNC41NSAxMDUsLTk5NS41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTk5NS41MjkgMTA1LC05ODUuNTI5IDEwMS41LC05OTUuNTI5IDEwOC41LC05OTUuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk5ODE2Mjk0OTE2MDAgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTUiPjx0aXRsZT4xMzk5ODE2Mjk0OTE2MDA8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjM0LjUsLTg3Ni41IDM0LjUsLTkxMi41IDE3NS41LC05MTIuNSAxNzUuNSwtODc2LjUgMzQuNSwtODc2LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii04OTAuOCI+Y29udjI6IENvbnZvbHV0aW9uMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk5ODE2MzIzMzgzODQmIzQ1OyZndDsxMzk5ODE2Mjk0OTE2MDAgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTQiPjx0aXRsZT4xMzk5ODE2MzIzMzgzODQtJmd0OzEzOTk4MTYyOTQ5MTYwMDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTk0OS4zMTNDMTA1LC05NDEuMjg5IDEwNSwtOTMxLjU0NyAxMDUsLTkyMi41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTkyMi41MjkgMTA1LC05MTIuNTI5IDEwMS41LC05MjIuNTI5IDEwOC41LC05MjIuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk5ODE2Mjk0OTI1NjAgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTYiPjx0aXRsZT4xMzk5ODE2Mjk0OTI1NjA8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjMxLC04MDMuNSAzMSwtODM5LjUgMTc5LC04MzkuNSAxNzksLTgwMy41IDMxLC04MDMuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTgxNy44Ij5hY3RpdmF0aW9uXzI6IEFjdGl2YXRpb248L3RleHQ+CjwvZz4KPCEtLSAxMzk5ODE2Mjk0OTE2MDAmIzQ1OyZndDsxMzk5ODE2Mjk0OTI1NjAgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTUiPjx0aXRsZT4xMzk5ODE2Mjk0OTE2MDAtJmd0OzEzOTk4MTYyOTQ5MjU2MDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTg3Ni4zMTNDMTA1LC04NjguMjg5IDEwNSwtODU4LjU0NyAxMDUsLTg0OS41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTg0OS41MjkgMTA1LC04MzkuNTI5IDEwMS41LC04NDkuNTI5IDEwOC41LC04NDkuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk5ODE2Mjg5MDg0OTYgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTciPjx0aXRsZT4xMzk5ODE2Mjg5MDg0OTY8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjUsLTczMC41IDUsLTc2Ni41IDIwNSwtNzY2LjUgMjA1LC03MzAuNSA1LC03MzAuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTc0NC44Ij5tYXhwb29saW5nMmRfMjogTWF4UG9vbGluZzJEPC90ZXh0Pgo8L2c+CjwhLS0gMTM5OTgxNjI5NDkyNTYwJiM0NTsmZ3Q7MTM5OTgxNjI4OTA4NDk2IC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2U2Ij48dGl0bGU+MTM5OTgxNjI5NDkyNTYwLSZndDsxMzk5ODE2Mjg5MDg0OTY8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC04MDMuMzEzQzEwNSwtNzk1LjI4OSAxMDUsLTc4NS41NDcgMTA1LC03NzYuNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC03NzYuNTI5IDEwNSwtNzY2LjUyOSAxMDEuNSwtNzc2LjUyOSAxMDguNSwtNzc2LjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5OTgxNjI4OTA4OTQ0IC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGU4Ij48dGl0bGU+MTM5OTgxNjI4OTA4OTQ0PC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSIzNC41LC02NTcuNSAzNC41LC02OTMuNSAxNzUuNSwtNjkzLjUgMTc1LjUsLTY1Ny41IDM0LjUsLTY1Ny41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItNjcxLjgiPmNvbnYzOiBDb252b2x1dGlvbjJEPC90ZXh0Pgo8L2c+CjwhLS0gMTM5OTgxNjI4OTA4NDk2JiM0NTsmZ3Q7MTM5OTgxNjI4OTA4OTQ0IC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2U3Ij48dGl0bGU+MTM5OTgxNjI4OTA4NDk2LSZndDsxMzk5ODE2Mjg5MDg5NDQ8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC03MzAuMzEzQzEwNSwtNzIyLjI4OSAxMDUsLTcxMi41NDcgMTA1LC03MDMuNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC03MDMuNTI5IDEwNSwtNjkzLjUyOSAxMDEuNSwtNzAzLjUyOSAxMDguNSwtNzAzLjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5OTgxNjI4OTExMzc2IC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGU5Ij48dGl0bGU+MTM5OTgxNjI4OTExMzc2PC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSIzMSwtNTg0LjUgMzEsLTYyMC41IDE3OSwtNjIwLjUgMTc5LC01ODQuNSAzMSwtNTg0LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii01OTguOCI+YWN0aXZhdGlvbl8zOiBBY3RpdmF0aW9uPC90ZXh0Pgo8L2c+CjwhLS0gMTM5OTgxNjI4OTA4OTQ0JiM0NTsmZ3Q7MTM5OTgxNjI4OTExMzc2IC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2U4Ij48dGl0bGU+MTM5OTgxNjI4OTA4OTQ0LSZndDsxMzk5ODE2Mjg5MTEzNzY8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC02NTcuMzEzQzEwNSwtNjQ5LjI4OSAxMDUsLTYzOS41NDcgMTA1LC02MzAuNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC02MzAuNTI5IDEwNSwtNjIwLjUyOSAxMDEuNSwtNjMwLjUyOSAxMDguNSwtNjMwLjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5OTgxNjI4NjEzODQwIC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGUxMCI+PHRpdGxlPjEzOTk4MTYyODYxMzg0MDwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iMjguNSwtNTExLjUgMjguNSwtNTQ3LjUgMTgxLjUsLTU0Ny41IDE4MS41LC01MTEuNSAyOC41LC01MTEuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTUyNS44Ij5lbmNvZGVkOiBNYXhQb29saW5nMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk5ODE2Mjg5MTEzNzYmIzQ1OyZndDsxMzk5ODE2Mjg2MTM4NDAgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTkiPjx0aXRsZT4xMzk5ODE2Mjg5MTEzNzYtJmd0OzEzOTk4MTYyODYxMzg0MDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTU4NC4zMTNDMTA1LC01NzYuMjg5IDEwNSwtNTY2LjU0NyAxMDUsLTU1Ny41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTU1Ny41MjkgMTA1LC01NDcuNTI5IDEwMS41LC01NTcuNTI5IDEwOC41LC01NTcuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk5ODE2Mjg2MTMzOTIgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTExIj48dGl0bGU+MTM5OTgxNjI4NjEzMzkyPC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSI1LC00MzguNSA1LC00NzQuNSAyMDUsLTQ3NC41IDIwNSwtNDM4LjUgNSwtNDM4LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii00NTIuOCI+Y29udm9sdXRpb24yZF8xOiBDb252b2x1dGlvbjJEPC90ZXh0Pgo8L2c+CjwhLS0gMTM5OTgxNjI4NjEzODQwJiM0NTsmZ3Q7MTM5OTgxNjI4NjEzMzkyIC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2UxMCI+PHRpdGxlPjEzOTk4MTYyODYxMzg0MC0mZ3Q7MTM5OTgxNjI4NjEzMzkyPC90aXRsZT4KPHBhdGggZD0iTTEwNSwtNTExLjMxM0MxMDUsLTUwMy4yODkgMTA1LC00OTMuNTQ3IDEwNSwtNDg0LjU2OSIgZmlsbD0ibm9uZSIgc3Ryb2tlPSJibGFjayIvPgo8cG9seWdvbiBmaWxsPSJibGFjayIgcG9pbnRzPSIxMDguNSwtNDg0LjUyOSAxMDUsLTQ3NC41MjkgMTAxLjUsLTQ4NC41MjkgMTA4LjUsLTQ4NC41MjkiIHN0cm9rZT0iYmxhY2siLz4KPC9nPgo8IS0tIDEzOTk4MTYyODMxMzA0MCAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlMTIiPjx0aXRsZT4xMzk5ODE2MjgzMTMwNDA8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjUsLTM2NS41IDUsLTQwMS41IDIwNSwtNDAxLjUgMjA1LC0zNjUuNSA1LC0zNjUuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTM3OS44Ij51cHNhbXBsaW5nMmRfMTogVXBTYW1wbGluZzJEPC90ZXh0Pgo8L2c+CjwhLS0gMTM5OTgxNjI4NjEzMzkyJiM0NTsmZ3Q7MTM5OTgxNjI4MzEzMDQwIC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2UxMSI+PHRpdGxlPjEzOTk4MTYyODYxMzM5Mi0mZ3Q7MTM5OTgxNjI4MzEzMDQwPC90aXRsZT4KPHBhdGggZD0iTTEwNSwtNDM4LjMxM0MxMDUsLTQzMC4yODkgMTA1LC00MjAuNTQ3IDEwNSwtNDExLjU2OSIgZmlsbD0ibm9uZSIgc3Ryb2tlPSJibGFjayIvPgo8cG9seWdvbiBmaWxsPSJibGFjayIgcG9pbnRzPSIxMDguNSwtNDExLjUyOSAxMDUsLTQwMS41MjkgMTAxLjUsLTQxMS41MjkgMTA4LjUsLTQxMS41MjkiIHN0cm9rZT0iYmxhY2siLz4KPC9nPgo8IS0tIDEzOTk4MTYyODI0NzE4NCAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlMTMiPjx0aXRsZT4xMzk5ODE2MjgyNDcxODQ8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjUsLTI5Mi41IDUsLTMyOC41IDIwNSwtMzI4LjUgMjA1LC0yOTIuNSA1LC0yOTIuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTMwNi44Ij5jb252b2x1dGlvbjJkXzI6IENvbnZvbHV0aW9uMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk5ODE2MjgzMTMwNDAmIzQ1OyZndDsxMzk5ODE2MjgyNDcxODQgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTEyIj48dGl0bGU+MTM5OTgxNjI4MzEzMDQwLSZndDsxMzk5ODE2MjgyNDcxODQ8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC0zNjUuMzEzQzEwNSwtMzU3LjI4OSAxMDUsLTM0Ny41NDcgMTA1LC0zMzguNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC0zMzguNTI5IDEwNSwtMzI4LjUyOSAxMDEuNSwtMzM4LjUyOSAxMDguNSwtMzM4LjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5OTgxNjI3NjgyMDAwIC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGUxNCI+PHRpdGxlPjEzOTk4MTYyNzY4MjAwMDwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iNSwtMjE5LjUgNSwtMjU1LjUgMjA1LC0yNTUuNSAyMDUsLTIxOS41IDUsLTIxOS41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItMjMzLjgiPnVwc2FtcGxpbmcyZF8yOiBVcFNhbXBsaW5nMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk5ODE2MjgyNDcxODQmIzQ1OyZndDsxMzk5ODE2Mjc2ODIwMDAgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTEzIj48dGl0bGU+MTM5OTgxNjI4MjQ3MTg0LSZndDsxMzk5ODE2Mjc2ODIwMDA8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC0yOTIuMzEzQzEwNSwtMjg0LjI4OSAxMDUsLTI3NC41NDcgMTA1LC0yNjUuNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC0yNjUuNTI5IDEwNSwtMjU1LjUyOSAxMDEuNSwtMjY1LjUyOSAxMDguNSwtMjY1LjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5OTgxNjI3NjgwNDY0IC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGUxNSI+PHRpdGxlPjEzOTk4MTYyNzY4MDQ2NDwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9Im5vbmUiIHBvaW50cz0iNSwtMTQ2LjUgNSwtMTgyLjUgMjA1LC0xODIuNSAyMDUsLTE0Ni41IDUsLTE0Ni41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItMTYwLjgiPmNvbnZvbHV0aW9uMmRfMzogQ29udm9sdXRpb24yRDwvdGV4dD4KPC9nPgo8IS0tIDEzOTk4MTYyNzY4MjAwMCYjNDU7Jmd0OzEzOTk4MTYyNzY4MDQ2NCAtLT4KPGcgY2xhc3M9ImVkZ2UiIGlkPSJlZGdlMTQiPjx0aXRsZT4xMzk5ODE2Mjc2ODIwMDAtJmd0OzEzOTk4MTYyNzY4MDQ2NDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTIxOS4zMTNDMTA1LC0yMTEuMjg5IDEwNSwtMjAxLjU0NyAxMDUsLTE5Mi41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTE5Mi41MjkgMTA1LC0xODIuNTI5IDEwMS41LC0xOTIuNTI5IDEwOC41LC0xOTIuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk5ODE2MjY5MjAzMzYgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTE2Ij48dGl0bGU+MTM5OTgxNjI2OTIwMzM2PC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSI1LC03My41IDUsLTEwOS41IDIwNSwtMTA5LjUgMjA1LC03My41IDUsLTczLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii04Ny44Ij51cHNhbXBsaW5nMmRfMzogVXBTYW1wbGluZzJEPC90ZXh0Pgo8L2c+CjwhLS0gMTM5OTgxNjI3NjgwNDY0JiM0NTsmZ3Q7MTM5OTgxNjI2OTIwMzM2IC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2UxNSI+PHRpdGxlPjEzOTk4MTYyNzY4MDQ2NC0mZ3Q7MTM5OTgxNjI2OTIwMzM2PC90aXRsZT4KPHBhdGggZD0iTTEwNSwtMTQ2LjMxM0MxMDUsLTEzOC4yODkgMTA1LC0xMjguNTQ3IDEwNSwtMTE5LjU2OSIgZmlsbD0ibm9uZSIgc3Ryb2tlPSJibGFjayIvPgo8cG9seWdvbiBmaWxsPSJibGFjayIgcG9pbnRzPSIxMDguNSwtMTE5LjUyOSAxMDUsLTEwOS41MjkgMTAxLjUsLTExOS41MjkgMTA4LjUsLTExOS41MjkiIHN0cm9rZT0iYmxhY2siLz4KPC9nPgo8IS0tIDEzOTk4MTYyNjk1Mjc4NCAtLT4KPGcgY2xhc3M9Im5vZGUiIGlkPSJub2RlMTciPjx0aXRsZT4xMzk5ODE2MjY5NTI3ODQ8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjI4LjUsLTAuNSAyOC41LC0zNi41IDE4MS41LC0zNi41IDE4MS41LC0wLjUgMjguNSwtMC41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItMTQuOCI+ZGVjb2RlZDogQ29udm9sdXRpb24yRDwvdGV4dD4KPC9nPgo8IS0tIDEzOTk4MTYyNjkyMDMzNiYjNDU7Jmd0OzEzOTk4MTYyNjk1Mjc4NCAtLT4KPGcgY2xhc3M9ImVkZ2UiIGlkPSJlZGdlMTYiPjx0aXRsZT4xMzk5ODE2MjY5MjAzMzYtJmd0OzEzOTk4MTYyNjk1Mjc4NDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTczLjMxMjlDMTA1LC02NS4yODk1IDEwNSwtNTUuNTQ3NSAxMDUsLTQ2LjU2OTEiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTQ2LjUyODggMTA1LC0zNi41Mjg4IDEwMS41LC00Ni41Mjg5IDEwOC41LC00Ni41Mjg4IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPC9nPgo8L3N2Zz4=)



#### Train

Output of CAE (decoder) is expected to be similar to the input of CAE (encoder). Thus, there is no metric to calculate accuracy here. Loss is the eucleadian distance between the output image of model with the input image that it was passeed. Thus we try to reduce this loss.

```python
cae.fit(cae_x_train, cae_x_train,
	                nb_epoch=epochs_for_cae,
	                batch_size=128,
                    verbose=1,
	                shuffle=True,
	                validation_data=(cae_x_validation, cae_x_validation))
```

    Train on 25000 samples, validate on 5000 samples
Epoch 1/40
25000/25000 [==============================] - 13s - loss: 0.1737 - val_loss: 0.0773 - ETA: 5s - loss: 0.2432 - ETA: 2s - loss: 0.1935
Epoch 2/40
25000/25000 [==============================] - 13s - loss: 0.0744 - val_loss: 0.0725 - ETA: 10s - loss: 0.0757 - ETA: 9s - loss: 0.0759
Epoch 3/40
25000/25000 [==============================] - 13s - loss: 0.0716 - val_loss: 0.0701 - ETA: 11s - loss: 0.0724 - ETA: 10s - loss: 0.0722 - ETA: 4s - loss: 0.0718 - ETA: 2s - loss: 0.0716 - ETA: 0s - loss: 0.0717
Epoch 4/40
25000/25000 [==============================] - 13s - loss: 0.0701 - val_loss: 0.0698 - ETA: 11s - loss: 0.0704 - ETA: 8s - loss: 0.0708
Epoch 5/40
25000/25000 [==============================] - 13s - loss: 0.0692 - val_loss: 0.0683
Epoch 6/40
25000/25000 [==============================] - 13s - loss: 0.0683 - val_loss: 0.0673
Epoch 7/40
25000/25000 [==============================] - 13s - loss: 0.0676 - val_loss: 0.0668
Epoch 8/40
25000/25000 [==============================] - 13s - loss: 0.0670 - val_loss: 0.0668
Epoch 9/40
25000/25000 [==============================] - 13s - loss: 0.0666 - val_loss: 0.0655
Epoch 10/40
25000/25000 [==============================] - 13s - loss: 0.0661 - val_loss: 0.0655
Epoch 11/40
25000/25000 [==============================] - 13s - loss: 0.0657 - val_loss: 0.0659
Epoch 12/40
25000/25000 [==============================] - 13s - loss: 0.0654 - val_loss: 0.0642
Epoch 13/40
25000/25000 [==============================] - 13s - loss: 0.0650 - val_loss: 0.0647 - ETA: 9s - loss: 0.0650 - ETA: 8s - loss: 0.0650
Epoch 14/40
25000/25000 [==============================] - 13s - loss: 0.0647 - val_loss: 0.0648 - ETA: 4s - loss: 0.0648
Epoch 15/40
25000/25000 [==============================] - 13s - loss: 0.0645 - val_loss: 0.0644 - ETA: 9s - loss: 0.0646 - ETA: 8s - loss: 0.0645
Epoch 16/40
25000/25000 [==============================] - 13s - loss: 0.0642 - val_loss: 0.0643 - ETA: 12s - loss: 0.0657 - ETA: 8s - loss: 0.0645
Epoch 17/40
25000/25000 [==============================] - 13s - loss: 0.0641 - val_loss: 0.0639
Epoch 18/40
25000/25000 [==============================] - 13s - loss: 0.0639 - val_loss: 0.0636 - ETA: 0s - loss: 0.0640
Epoch 19/40
25000/25000 [==============================] - 13s - loss: 0.0637 - val_loss: 0.0636 - ETA: 11s - loss: 0.0636 - ETA: 8s - loss: 0.0641 - ETA: 8s - loss: 0.0641
Epoch 20/40
25000/25000 [==============================] - 13s - loss: 0.0636 - val_loss: 0.0639 - ETA: 8s - loss: 0.0636
Epoch 21/40
25000/25000 [==============================] - 13s - loss: 0.0634 - val_loss: 0.0635 - ETA: 12s - loss: 0.0638 - ETA: 11s - loss: 0.0642 - ETA: 6s - loss: 0.0636
Epoch 22/40
25000/25000 [==============================] - 13s - loss: 0.0633 - val_loss: 0.0630 - ETA: 12s - loss: 0.0639 - ETA: 1s - loss: 0.0634
Epoch 23/40
25000/25000 [==============================] - 13s - loss: 0.0632 - val_loss: 0.0637
Epoch 24/40
25000/25000 [==============================] - 13s - loss: 0.0631 - val_loss: 0.0633
Epoch 25/40
25000/25000 [==============================] - 13s - loss: 0.0630 - val_loss: 0.0628
Epoch 26/40
25000/25000 [==============================] - 13s - loss: 0.0629 - val_loss: 0.0627 - ETA: 1s - loss: 0.0629
Epoch 27/40
25000/25000 [==============================] - 13s - loss: 0.0628 - val_loss: 0.0627
Epoch 28/40
25000/25000 [==============================] - 13s - loss: 0.0627 - val_loss: 0.0625
Epoch 29/40
25000/25000 [==============================] - 13s - loss: 0.0626 - val_loss: 0.0626 - ETA: 7s - loss: 0.0625
Epoch 30/40
25000/25000 [==============================] - 13s - loss: 0.0626 - val_loss: 0.0627
Epoch 31/40
25000/25000 [==============================] - 13s - loss: 0.0625 - val_loss: 0.0624
Epoch 32/40
25000/25000 [==============================] - 13s - loss: 0.0625 - val_loss: 0.0623
Epoch 33/40
25000/25000 [==============================] - 13s - loss: 0.0624 - val_loss: 0.0624
Epoch 34/40
25000/25000 [==============================] - 13s - loss: 0.0623 - val_loss: 0.0622
Epoch 35/40
25000/25000 [==============================] - 13s - loss: 0.0623 - val_loss: 0.0623
Epoch 36/40
25000/25000 [==============================] - 13s - loss: 0.0623 - val_loss: 0.0621
Epoch 37/40
25000/25000 [==============================] - 13s - loss: 0.0622 - val_loss: 0.0621
Epoch 38/40
25000/25000 [==============================] - 13s - loss: 0.0622 - val_loss: 0.0618
Epoch 39/40
25000/25000 [==============================] - 13s - loss: 0.0621 - val_loss: 0.0617
Epoch 40/40
25000/25000 [==============================] - 13s - loss: 0.0620 - val_loss: 0.0618
Out[8]:
<keras.callbacks.History at 0x7fd8b2c4fd50>



#### Save model

We need ```h5py``` module for this cell. It is a good practice to include epoch number, training set size or other distinguishing factors in the name of the file containing weights of the model. 

```python
cae_name = 'mnist_cae_'+str(epochs_for_cae)+'e_'+str(training_samples_for_cae)+'i.h5'
cae.save(cae_name)
```

#### Check results and encoded representation

Visualize! Visualize! 

![jpg](https://github.com/ashirgao/Comparison-between-deep-learning-methods-on-limited-labelled-datasets/blob/gh-pages/CAEandCNN_files/bret.jpg?raw=true)

(remember Bret Stiles?)

```python
test = cae_x_validation
test = np.reshape(test,(validation_samples_for_cae,1,28,28))
decoded_imgs = cae.predict(test)
encoder = Model(input=cae.input, output=cae.get_layer('encoded').output)
encoded_imgs = encoder.predict(test)

print "Results (orignal vs reconstructed image): "
n = 10
plt.figure(figsize=(20, 4))
for i in range(n):
    # display original
    ax = plt.subplot(2, n, i+1)
    plt.imshow(test[i].reshape(28, 28))
    plt.gray()
    ax.get_xaxis().set_visible(False)
    ax.get_yaxis().set_visible(False)

    # display reconstruction
    ax = plt.subplot(2, n, i + 1 + n)
    plt.imshow(decoded_imgs[i].reshape(28, 28))
    plt.gray()
    ax.get_xaxis().set_visible(False)
    ax.get_yaxis().set_visible(False)
plt.show()

print "Encoded representation : "
n = 10
plt.figure(figsize=(20, 8))
for i in range(n):
    ax = plt.subplot(1, n, i+1)
    plt.imshow(encoded_imgs[i].reshape(4, 4 * 8).T)
    plt.gray()
    ax.get_xaxis().set_visible(False)
    ax.get_yaxis().set_visible(False)
plt.show()
```

    Results (orignal vs reconstructed image): 



![png](https://github.com/ashirgao/Comparison-between-deep-learning-methods-on-limited-labelled-datasets/blob/gh-pages/CAEandCNN_files/CAEandCNN_20_1.png?raw=true)


    Encoded representation : 



![png](https://github.com/ashirgao/Comparison-between-deep-learning-methods-on-limited-labelled-datasets/blob/gh-pages/CAEandCNN_files/CAEandCNN_20_3.png?raw=true)


#### CNN data

Preparing data for CNN training. Number of samples for CNN training is decided beforehand.

```python
cnn_x_train = np.reshape(x[count:count+training_samples_for_cnn], (training_samples_for_cnn, 1, 28, 28))
cnn_y_train = np_utils.to_categorical(y[count:count+training_samples_for_cnn], nb_classes=10)
count = count + training_samples_for_cnn

cnn_x_test = np.reshape(x[count:count+testing_samples_for_cnn], (testing_samples_for_cnn, 1, 28, 28))
cnn_y_test = np_utils.to_categorical(y[count:count+testing_samples_for_cnn], nb_classes=10)
count = count + testing_samples_for_cnn

cnn_x_validation = np.reshape(x[count:count+validation_samples_for_cnn], (validation_samples_for_cnn, 1, 28, 28))
cnn_y_validation = np_utils.to_categorical(y[count:count+validation_samples_for_cnn], nb_classes=10)
count = count + validation_samples_for_cnn

print "cnn_x_train shape : ", cnn_x_train.shape
print "cnn_x_test shape : ", cnn_x_test.shape
print "cnn_x_validation shape : ", cnn_x_validation.shape

print "cnn_y_train shape : ", cnn_y_train.shape
print "cnn_y_test shape : ", cnn_y_test.shape
print "cnn_y_validation shape : ", cnn_y_validation.shape
```

    cnn_x_train shape :  (25000, 1, 28, 28)
    cnn_x_test shape :  (10000, 1, 28, 28)
    cnn_x_validation shape :  (5000, 1, 28, 28)
    cnn_y_train shape :  (25000, 10)
    cnn_y_test shape :  (10000, 10)
    cnn_y_validation shape :  (5000, 10)


#### CNN model

Define model for CNN. It must be same as the encoder of CAE. In keras, the names of individual layers must also match.

```python
model = Sequential()

model.add(Conv2D(8, 3, 3, activation='relu', border_mode='same',  input_shape=(1, 28, 28), name='conv1' ))
model.add(MaxPooling2D(pool_size=(2, 2), border_mode='same'))

model.add(Conv2D(16, 3, 3, activation='relu', border_mode='same', name='conv2'))
model.add(MaxPooling2D(pool_size=(2, 2), border_mode='same'))

model.add(Conv2D(8, 3, 3, activation='relu', border_mode='same', name='conv3'))
model.add(MaxPooling2D(pool_size=(2, 2), border_mode='same', name='encoded'))

model.add(Flatten())
model.add(Dense(16))

model.add(Activation('relu'))
model.add(Dropout(0.5))

model.add(Dense(10))
model.add(Activation('softmax'))

```

#### Model summary

Thank-you keras for this.

```python
model.summary()
```

    ____________________________________________________________________________________________________
    Layer (type)                     Output Shape          Param #     Connected to                     
    ====================================================================================================
    conv1 (Convolution2D)            (None, 8, 28, 28)     80          convolution2d_input_6[0][0]      
    ____________________________________________________________________________________________________
    maxpooling2d_11 (MaxPooling2D)   (None, 8, 14, 14)     0           conv1[0][0]                      
    ____________________________________________________________________________________________________
    conv2 (Convolution2D)            (None, 16, 14, 14)    1168        maxpooling2d_11[0][0]            
    ____________________________________________________________________________________________________
    maxpooling2d_12 (MaxPooling2D)   (None, 16, 7, 7)      0           conv2[0][0]                      
    ____________________________________________________________________________________________________
    conv3 (Convolution2D)            (None, 8, 7, 7)       1160        maxpooling2d_12[0][0]            
    ____________________________________________________________________________________________________
    encoded (MaxPooling2D)           (None, 8, 4, 4)       0           conv3[0][0]                      
    ____________________________________________________________________________________________________
    flatten_3 (Flatten)              (None, 128)           0           encoded[0][0]                    
    ____________________________________________________________________________________________________
    dense_4 (Dense)                  (None, 16)            2064        flatten_3[0][0]                  
    ____________________________________________________________________________________________________
    activation_10 (Activation)       (None, 16)            0           dense_4[0][0]                    
    ____________________________________________________________________________________________________
    dropout_2 (Dropout)              (None, 16)            0           activation_10[0][0]              
    ____________________________________________________________________________________________________
    dense_5 (Dense)                  (None, 10)            170         dropout_2[0][0]                  
    ____________________________________________________________________________________________________
    activation_11 (Activation)       (None, 10)            0           dense_5[0][0]                    
    ====================================================================================================
    Total params: 4,642
    Trainable params: 4,642
    Non-trainable params: 0
    ____________________________________________________________________________________________________


#### Diagram

![jpg](https://github.com/ashirgao/Comparison-between-deep-learning-methods-on-limited-labelled-datasets/blob/gh-pages/CAEandCNN_files/bret.jpg?raw=true)

(Red John?  (^^) )

```python
SVG(model_to_dot(model).create(prog='dot', format='svg'))
```




![svg](data:image/svg+xml;base64,PHN2ZyBoZWlnaHQ9IjkyMXB0IiB2aWV3Qm94PSIwLjAwIDAuMDAgMjE4LjAwIDkyMS4wMCIgd2lkdGg9IjIxOHB0IiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIj4KPGcgY2xhc3M9ImdyYXBoIiBpZD0iZ3JhcGgwIiB0cmFuc2Zvcm09InNjYWxlKDEgMSkgcm90YXRlKDApIHRyYW5zbGF0ZSg0IDkxNykiPgo8dGl0bGU+RzwvdGl0bGU+Cjxwb2x5Z29uIGZpbGw9IndoaXRlIiBwb2ludHM9Ii00LDQgLTQsLTkxNyAyMTQsLTkxNyAyMTQsNCAtNCw0IiBzdHJva2U9Im5vbmUiLz4KPCEtLSAxMzk3ODY1OTMzNzAyNTYgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTEiPjx0aXRsZT4xMzk3ODY1OTMzNzAyNTY8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjAsLTg3Ni41IDAsLTkxMi41IDIxMCwtOTEyLjUgMjEwLC04NzYuNSAwLC04NzYuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTg5MC44Ij5jb252b2x1dGlvbjJkX2lucHV0XzY6IElucHV0TGF5ZXI8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY1OTMzNzI1NjAgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTIiPjx0aXRsZT4xMzk3ODY1OTMzNzI1NjA8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjM0LjUsLTgwMy41IDM0LjUsLTgzOS41IDE3NS41LC04MzkuNSAxNzUuNSwtODAzLjUgMzQuNSwtODAzLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii04MTcuOCI+Y29udjE6IENvbnZvbHV0aW9uMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY1OTMzNzAyNTYmIzQ1OyZndDsxMzk3ODY1OTMzNzI1NjAgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTEiPjx0aXRsZT4xMzk3ODY1OTMzNzAyNTYtJmd0OzEzOTc4NjU5MzM3MjU2MDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTg3Ni4zMTNDMTA1LC04NjguMjg5IDEwNSwtODU4LjU0NyAxMDUsLTg0OS41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTg0OS41MjkgMTA1LC04MzkuNTI5IDEwMS41LC04NDkuNTI5IDEwOC41LC04NDkuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY1OTMzNzIxMTIgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTMiPjx0aXRsZT4xMzk3ODY1OTMzNzIxMTI8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjEuNSwtNzMwLjUgMS41LC03NjYuNSAyMDguNSwtNzY2LjUgMjA4LjUsLTczMC41IDEuNSwtNzMwLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii03NDQuOCI+bWF4cG9vbGluZzJkXzExOiBNYXhQb29saW5nMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY1OTMzNzI1NjAmIzQ1OyZndDsxMzk3ODY1OTMzNzIxMTIgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTIiPjx0aXRsZT4xMzk3ODY1OTMzNzI1NjAtJmd0OzEzOTc4NjU5MzM3MjExMjwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTgwMy4zMTNDMTA1LC03OTUuMjg5IDEwNSwtNzg1LjU0NyAxMDUsLTc3Ni41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTc3Ni41MjkgMTA1LC03NjYuNTI5IDEwMS41LC03NzYuNTI5IDEwOC41LC03NzYuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY3NzQ3OTc5NjggLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTQiPjx0aXRsZT4xMzk3ODY3NzQ3OTc5Njg8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjM0LjUsLTY1Ny41IDM0LjUsLTY5My41IDE3NS41LC02OTMuNSAxNzUuNSwtNjU3LjUgMzQuNSwtNjU3LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii02NzEuOCI+Y29udjI6IENvbnZvbHV0aW9uMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY1OTMzNzIxMTImIzQ1OyZndDsxMzk3ODY3NzQ3OTc5NjggLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTMiPjx0aXRsZT4xMzk3ODY1OTMzNzIxMTItJmd0OzEzOTc4Njc3NDc5Nzk2ODwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTczMC4zMTNDMTA1LC03MjIuMjg5IDEwNSwtNzEyLjU0NyAxMDUsLTcwMy41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTcwMy41MjkgMTA1LC02OTMuNTI5IDEwMS41LC03MDMuNTI5IDEwOC41LC03MDMuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY2OTE5MDcwODggLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTUiPjx0aXRsZT4xMzk3ODY2OTE5MDcwODg8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjEuNSwtNTg0LjUgMS41LC02MjAuNSAyMDguNSwtNjIwLjUgMjA4LjUsLTU4NC41IDEuNSwtNTg0LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii01OTguOCI+bWF4cG9vbGluZzJkXzEyOiBNYXhQb29saW5nMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY3NzQ3OTc5NjgmIzQ1OyZndDsxMzk3ODY2OTE5MDcwODggLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTQiPjx0aXRsZT4xMzk3ODY3NzQ3OTc5NjgtJmd0OzEzOTc4NjY5MTkwNzA4ODwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTY1Ny4zMTNDMTA1LC02NDkuMjg5IDEwNSwtNjM5LjU0NyAxMDUsLTYzMC41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTYzMC41MjkgMTA1LC02MjAuNTI5IDEwMS41LC02MzAuNTI5IDEwOC41LC02MzAuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY3NzUwNjY5NjAgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTYiPjx0aXRsZT4xMzk3ODY3NzUwNjY5NjA8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjM0LjUsLTUxMS41IDM0LjUsLTU0Ny41IDE3NS41LC01NDcuNSAxNzUuNSwtNTExLjUgMzQuNSwtNTExLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii01MjUuOCI+Y29udjM6IENvbnZvbHV0aW9uMkQ8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY2OTE5MDcwODgmIzQ1OyZndDsxMzk3ODY3NzUwNjY5NjAgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTUiPjx0aXRsZT4xMzk3ODY2OTE5MDcwODgtJmd0OzEzOTc4Njc3NTA2Njk2MDwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTU4NC4zMTNDMTA1LC01NzYuMjg5IDEwNSwtNTY2LjU0NyAxMDUsLTU1Ny41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTU1Ny41MjkgMTA1LC01NDcuNTI5IDEwMS41LC01NTcuNTI5IDEwOC41LC01NTcuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY2OTE4NjAyNDAgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTciPjx0aXRsZT4xMzk3ODY2OTE4NjAyNDA8L3RpdGxlPgo8cG9seWdvbiBmaWxsPSJub25lIiBwb2ludHM9IjI4LjUsLTQzOC41IDI4LjUsLTQ3NC41IDE4MS41LC00NzQuNSAxODEuNSwtNDM4LjUgMjguNSwtNDM4LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii00NTIuOCI+ZW5jb2RlZDogTWF4UG9vbGluZzJEPC90ZXh0Pgo8L2c+CjwhLS0gMTM5Nzg2Nzc1MDY2OTYwJiM0NTsmZ3Q7MTM5Nzg2NjkxODYwMjQwIC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2U2Ij48dGl0bGU+MTM5Nzg2Nzc1MDY2OTYwLSZndDsxMzk3ODY2OTE4NjAyNDA8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC01MTEuMzEzQzEwNSwtNTAzLjI4OSAxMDUsLTQ5My41NDcgMTA1LC00ODQuNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC00ODQuNTI5IDEwNSwtNDc0LjUyOSAxMDEuNSwtNDg0LjUyOSAxMDguNSwtNDg0LjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5Nzg2NjkwODk0MTYwIC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGU4Ij48dGl0bGU+MTM5Nzg2NjkwODk0MTYwPC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSI1MCwtMzY1LjUgNTAsLTQwMS41IDE2MCwtNDAxLjUgMTYwLC0zNjUuNSA1MCwtMzY1LjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii0zNzkuOCI+ZmxhdHRlbl8zOiBGbGF0dGVuPC90ZXh0Pgo8L2c+CjwhLS0gMTM5Nzg2NjkxODYwMjQwJiM0NTsmZ3Q7MTM5Nzg2NjkwODk0MTYwIC0tPgo8ZyBjbGFzcz0iZWRnZSIgaWQ9ImVkZ2U3Ij48dGl0bGU+MTM5Nzg2NjkxODYwMjQwLSZndDsxMzk3ODY2OTA4OTQxNjA8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC00MzguMzEzQzEwNSwtNDMwLjI4OSAxMDUsLTQyMC41NDcgMTA1LC00MTEuNTY5IiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC00MTEuNTI5IDEwNSwtNDAxLjUyOSAxMDEuNSwtNDExLjUyOSAxMDguNSwtNDExLjUyOSIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwhLS0gMTM5Nzg2NjkwMjQ3ODg4IC0tPgo8ZyBjbGFzcz0ibm9kZSIgaWQ9Im5vZGU5Ij48dGl0bGU+MTM5Nzg2NjkwMjQ3ODg4PC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSI1NCwtMjkyLjUgNTQsLTMyOC41IDE1NiwtMzI4LjUgMTU2LC0yOTIuNSA1NCwtMjkyLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii0zMDYuOCI+ZGVuc2VfNDogRGVuc2U8L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY2OTA4OTQxNjAmIzQ1OyZndDsxMzk3ODY2OTAyNDc4ODggLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTgiPjx0aXRsZT4xMzk3ODY2OTA4OTQxNjAtJmd0OzEzOTc4NjY5MDI0Nzg4ODwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTM2NS4zMTNDMTA1LC0zNTcuMjg5IDEwNSwtMzQ3LjU0NyAxMDUsLTMzOC41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTMzOC41MjkgMTA1LC0zMjguNTI5IDEwMS41LC0zMzguNTI5IDEwOC41LC0zMzguNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY3NzQxNDk0NTYgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTEwIj48dGl0bGU+MTM5Nzg2Nzc0MTQ5NDU2PC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSIyNy41LC0yMTkuNSAyNy41LC0yNTUuNSAxODIuNSwtMjU1LjUgMTgyLjUsLTIxOS41IDI3LjUsLTIxOS41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItMjMzLjgiPmFjdGl2YXRpb25fMTA6IEFjdGl2YXRpb248L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY2OTAyNDc4ODgmIzQ1OyZndDsxMzk3ODY3NzQxNDk0NTYgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTkiPjx0aXRsZT4xMzk3ODY2OTAyNDc4ODgtJmd0OzEzOTc4Njc3NDE0OTQ1NjwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTI5Mi4zMTNDMTA1LC0yODQuMjg5IDEwNSwtMjc0LjU0NyAxMDUsLTI2NS41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTI2NS41MjkgMTA1LC0yNTUuNTI5IDEwMS41LC0yNjUuNTI5IDEwOC41LC0yNjUuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY3NzQxNTAwOTYgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTExIj48dGl0bGU+MTM5Nzg2Nzc0MTUwMDk2PC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSI0Mi41LC0xNDYuNSA0Mi41LC0xODIuNSAxNjcuNSwtMTgyLjUgMTY3LjUsLTE0Ni41IDQyLjUsLTE0Ni41IiBzdHJva2U9ImJsYWNrIi8+Cjx0ZXh0IGZvbnQtZmFtaWx5PSJUaW1lcyxzZXJpZiIgZm9udC1zaXplPSIxNC4wMCIgdGV4dC1hbmNob3I9Im1pZGRsZSIgeD0iMTA1IiB5PSItMTYwLjgiPmRyb3BvdXRfMjogRHJvcG91dDwvdGV4dD4KPC9nPgo8IS0tIDEzOTc4Njc3NDE0OTQ1NiYjNDU7Jmd0OzEzOTc4Njc3NDE1MDA5NiAtLT4KPGcgY2xhc3M9ImVkZ2UiIGlkPSJlZGdlMTAiPjx0aXRsZT4xMzk3ODY3NzQxNDk0NTYtJmd0OzEzOTc4Njc3NDE1MDA5NjwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTIxOS4zMTNDMTA1LC0yMTEuMjg5IDEwNSwtMjAxLjU0NyAxMDUsLTE5Mi41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTE5Mi41MjkgMTA1LC0xODIuNTI5IDEwMS41LC0xOTIuNTI5IDEwOC41LC0xOTIuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY2OTAzMDcxNTIgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTEyIj48dGl0bGU+MTM5Nzg2NjkwMzA3MTUyPC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSI1NCwtNzMuNSA1NCwtMTA5LjUgMTU2LC0xMDkuNSAxNTYsLTczLjUgNTQsLTczLjUiIHN0cm9rZT0iYmxhY2siLz4KPHRleHQgZm9udC1mYW1pbHk9IlRpbWVzLHNlcmlmIiBmb250LXNpemU9IjE0LjAwIiB0ZXh0LWFuY2hvcj0ibWlkZGxlIiB4PSIxMDUiIHk9Ii04Ny44Ij5kZW5zZV81OiBEZW5zZTwvdGV4dD4KPC9nPgo8IS0tIDEzOTc4Njc3NDE1MDA5NiYjNDU7Jmd0OzEzOTc4NjY5MDMwNzE1MiAtLT4KPGcgY2xhc3M9ImVkZ2UiIGlkPSJlZGdlMTEiPjx0aXRsZT4xMzk3ODY3NzQxNTAwOTYtJmd0OzEzOTc4NjY5MDMwNzE1MjwvdGl0bGU+CjxwYXRoIGQ9Ik0xMDUsLTE0Ni4zMTNDMTA1LC0xMzguMjg5IDEwNSwtMTI4LjU0NyAxMDUsLTExOS41NjkiIGZpbGw9Im5vbmUiIHN0cm9rZT0iYmxhY2siLz4KPHBvbHlnb24gZmlsbD0iYmxhY2siIHBvaW50cz0iMTA4LjUsLTExOS41MjkgMTA1LC0xMDkuNTI5IDEwMS41LC0xMTkuNTI5IDEwOC41LC0xMTkuNTI5IiBzdHJva2U9ImJsYWNrIi8+CjwvZz4KPCEtLSAxMzk3ODY3MTI4NTcyMzIgLS0+CjxnIGNsYXNzPSJub2RlIiBpZD0ibm9kZTEzIj48dGl0bGU+MTM5Nzg2NzEyODU3MjMyPC90aXRsZT4KPHBvbHlnb24gZmlsbD0ibm9uZSIgcG9pbnRzPSIyNy41LC0wLjUgMjcuNSwtMzYuNSAxODIuNSwtMzYuNSAxODIuNSwtMC41IDI3LjUsLTAuNSIgc3Ryb2tlPSJibGFjayIvPgo8dGV4dCBmb250LWZhbWlseT0iVGltZXMsc2VyaWYiIGZvbnQtc2l6ZT0iMTQuMDAiIHRleHQtYW5jaG9yPSJtaWRkbGUiIHg9IjEwNSIgeT0iLTE0LjgiPmFjdGl2YXRpb25fMTE6IEFjdGl2YXRpb248L3RleHQ+CjwvZz4KPCEtLSAxMzk3ODY2OTAzMDcxNTImIzQ1OyZndDsxMzk3ODY3MTI4NTcyMzIgLS0+CjxnIGNsYXNzPSJlZGdlIiBpZD0iZWRnZTEyIj48dGl0bGU+MTM5Nzg2NjkwMzA3MTUyLSZndDsxMzk3ODY3MTI4NTcyMzI8L3RpdGxlPgo8cGF0aCBkPSJNMTA1LC03My4zMTI5QzEwNSwtNjUuMjg5NSAxMDUsLTU1LjU0NzUgMTA1LC00Ni41NjkxIiBmaWxsPSJub25lIiBzdHJva2U9ImJsYWNrIi8+Cjxwb2x5Z29uIGZpbGw9ImJsYWNrIiBwb2ludHM9IjEwOC41LC00Ni41Mjg4IDEwNSwtMzYuNTI4OCAxMDEuNSwtNDYuNTI4OSAxMDguNSwtNDYuNTI4OCIgc3Ryb2tlPSJibGFjayIvPgo8L2c+CjwvZz4KPC9zdmc+)



#### APPROACH 1 :  Intialiazed CNN 


Finally! Now let us try approach 1 ie. CNN initialized using CAE weights.

```python
model2 = model
model2.load_weights(cae_name, by_name=True)
model2.compile(optimizer='adadelta', loss='categorical_crossentropy',metrics=['accuracy'])
model2.fit(cnn_x_train, cnn_y_train,
                nb_epoch=epochs_for_cnn,
                batch_size=128,
                shuffle=True,
                verbose=1,
                validation_data=(cnn_x_validation, cnn_y_validation))
```

    Train on 25000 samples, validate on 5000 samples
	Epoch 1/20
	25000/25000 [==============================] - 13s - loss: 1.8523 - acc: 0.6000 - val_loss: 0.1355 - val_acc: 0.9746 - ETA: 13s - loss: 13.7110 - acc: 0.1208 - ETA: 8s - loss: 3.5260 - acc: 0.4371
	Epoch 2/20
	25000/25000 [==============================] - 12s - loss: 0.7200 - acc: 0.7369 - val_loss: 0.1029 - val_acc: 0.9794 - ETA: 12s - loss: 0.7538 - acc: 0.7346
	Epoch 3/20
	25000/25000 [==============================] - 12s - loss: 0.6510 - acc: 0.7622 - val_loss: 0.0892 - val_acc: 0.9816
	Epoch 4/20
	25000/25000 [==============================] - 12s - loss: 0.6304 - acc: 0.7743 - val_loss: 0.0841 - val_acc: 0.9798
	Epoch 5/20
	25000/25000 [==============================] - 12s - loss: 0.6055 - acc: 0.7803 - val_loss: 0.0834 - val_acc: 0.9824
	Epoch 6/20
	25000/25000 [==============================] - 12s - loss: 0.5973 - acc: 0.7828 - val_loss: 0.0852 - val_acc: 0.9814
	Epoch 7/20
	25000/25000 [==============================] - 12s - loss: 0.5974 - acc: 0.7811 - val_loss: 0.1044 - val_acc: 0.9794
	Epoch 8/20
	25000/25000 [==============================] - 12s - loss: 0.5812 - acc: 0.7894 - val_loss: 0.0759 - val_acc: 0.9838
	Epoch 9/20
	25000/25000 [==============================] - 12s - loss: 0.5815 - acc: 0.7904 - val_loss: 0.0650 - val_acc: 0.9850
	Epoch 10/20
	25000/25000 [==============================] - 12s - loss: 0.5782 - acc: 0.7878 - val_loss: 0.0912 - val_acc: 0.9810
	Epoch 11/20
	25000/25000 [==============================] - 12s - loss: 0.5729 - acc: 0.7890 - val_loss: 0.0765 - val_acc: 0.9822
	Epoch 12/20
	25000/25000 [==============================] - 12s - loss: 0.5720 - acc: 0.7914 - val_loss: 0.0835 - val_acc: 0.9820
	Epoch 13/20
	25000/25000 [==============================] - 12s - loss: 0.5655 - acc: 0.7977 - val_loss: 0.0617 - val_acc: 0.9850
	Epoch 14/20
	25000/25000 [==============================] - 12s - loss: 0.5710 - acc: 0.7938 - val_loss: 0.0693 - val_acc: 0.9842
	Epoch 15/20
	25000/25000 [==============================] - 12s - loss: 0.5673 - acc: 0.7926 - val_loss: 0.0843 - val_acc: 0.9806
	Epoch 16/20
	25000/25000 [==============================] - 12s - loss: 0.5614 - acc: 0.7956 - val_loss: 0.0821 - val_acc: 0.9808
	Epoch 17/20
	25000/25000 [==============================] - 12s - loss: 0.5540 - acc: 0.7994 - val_loss: 0.0777 - val_acc: 0.9824 - ETA: 11s - loss: 0.5782 - acc: 0.8258
	Epoch 18/20
	25000/25000 [==============================] - 12s - loss: 0.5525 - acc: 0.8022 - val_loss: 0.0571 - val_acc: 0.9854
	Epoch 19/20
	25000/25000 [==============================] - 12s - loss: 0.5623 - acc: 0.7990 - val_loss: 0.0749 - val_acc: 0.9808
	Epoch 20/20
	25000/25000 [==============================] - 12s - loss: 0.5507 - acc: 0.8041 - val_loss: 0.0577 - val_acc: 0.9846
	Out[17]:
	<keras.callbacks.History at 0x7fd883fc71d0>



#### Save model
Always as good practice to store all your models. Never know when you need which.

```python
name = 'mnist_initializedCNN_'+str(epochs_for_cnn)+'e_'+str(training_samples_for_cnn)+'i.h5'
model2.save(name)
```

#### Results of approach 1

```python
params2 = model2.metrics_names
results2 = (model2.evaluate(cnn_x_test,cnn_y_test,verbose=0))
print params2[0]," : ", results2[0]
print params2[1]," : ", results2[1]
```

    loss  :  0.100158229173
    acc  :  0.9726

>97.26%

Too good!! 
(Easy task, model with too many extra parameters)


#### APPROACH 2  :  Unitialiazed CNN 

Traditional 'only CNN' approach.

```python
model1 = model
model1.compile(optimizer='adadelta', loss='categorical_crossentropy',metrics=['accuracy'])

model1.fit(cnn_x_train, cnn_y_train,
                nb_epoch=epochs_for_cnn,
                batch_size=128,
                shuffle=True,
                verbose=1,
                validation_data=(cnn_x_validation, cnn_y_validation))
```

    Train on 25000 samples, validate on 5000 samples
	Epoch 1/20
	25000/25000 [==============================] - 13s - loss: 1.2829 - acc: 0.5217 - val_loss: 0.4114 - val_acc: 0.9286 - ETA: 9s - loss: 1.5621 - acc: 0.4222 - ETA: 7s - loss: 1.4712 - acc: 0.4525
	Epoch 2/20
	25000/25000 [==============================] - 13s - loss: 1.0242 - acc: 0.6134 - val_loss: 0.2856 - val_acc: 0.9572
	Epoch 3/20
	25000/25000 [==============================] - 13s - loss: 0.9088 - acc: 0.6571 - val_loss: 0.2024 - val_acc: 0.9686
	Epoch 4/20
	25000/25000 [==============================] - 13s - loss: 0.8292 - acc: 0.6838 - val_loss: 0.1384 - val_acc: 0.9730 - ETA: 10s - loss: 0.8526 - acc: 0.6840 - ETA: 7s - loss: 0.8367 - acc: 0.6836 - ETA: 4s - loss: 0.8354 - acc: 0.6831
	Epoch 5/20
	25000/25000 [==============================] - 12s - loss: 0.7787 - acc: 0.6990 - val_loss: 0.1423 - val_acc: 0.9768
	Epoch 6/20
	25000/25000 [==============================] - 12s - loss: 0.7459 - acc: 0.7142 - val_loss: 0.1391 - val_acc: 0.9764
	Epoch 7/20
	25000/25000 [==============================] - 12s - loss: 0.7349 - acc: 0.7194 - val_loss: 0.0994 - val_acc: 0.9802 - ETA: 10s - loss: 0.7540 - acc: 0.7135
	Epoch 8/20
	25000/25000 [==============================] - 12s - loss: 0.7087 - acc: 0.7330 - val_loss: 0.0934 - val_acc: 0.9798 - ETA: 6s - loss: 0.7199 - acc: 0.7282 - ETA: 6s - loss: 0.7227 - acc: 0.7279
	Epoch 9/20
	25000/25000 [==============================] - 12s - loss: 0.6758 - acc: 0.7504 - val_loss: 0.0871 - val_acc: 0.9808 - ETA: 0s - loss: 0.6766 - acc: 0.7498
	Epoch 10/20
	25000/25000 [==============================] - 12s - loss: 0.6612 - acc: 0.7539 - val_loss: 0.0938 - val_acc: 0.9798 - ETA: 5s - loss: 0.6697 - acc: 0.7510 - ETA: 4s - loss: 0.6659 - acc: 0.7531
	Epoch 11/20
	25000/25000 [==============================] - 12s - loss: 0.6476 - acc: 0.7569 - val_loss: 0.0863 - val_acc: 0.9812
	Epoch 12/20
	25000/25000 [==============================] - 12s - loss: 0.6356 - acc: 0.7642 - val_loss: 0.0854 - val_acc: 0.9804
	Epoch 13/20
	25000/25000 [==============================] - 13s - loss: 0.6345 - acc: 0.7648 - val_loss: 0.0828 - val_acc: 0.9828 - ETA: 7s - loss: 0.6425 - acc: 0.7636
	Epoch 14/20
	25000/25000 [==============================] - 12s - loss: 0.6272 - acc: 0.7694 - val_loss: 0.0782 - val_acc: 0.9832
	Epoch 15/20
	25000/25000 [==============================] - 12s - loss: 0.6276 - acc: 0.7681 - val_loss: 0.0761 - val_acc: 0.9836 - ETA: 4s - loss: 0.6234 - acc: 0.7695
	Epoch 16/20
	25000/25000 [==============================] - 12s - loss: 0.6220 - acc: 0.7668 - val_loss: 0.0716 - val_acc: 0.9826 - ETA: 6s - loss: 0.6249 - acc: 0.7641 - ETA: 3s - loss: 0.6186 - acc: 0.7679
	Epoch 17/20
	25000/25000 [==============================] - 12s - loss: 0.6195 - acc: 0.7689 - val_loss: 0.0677 - val_acc: 0.9840 - ETA: 7s - loss: 0.6067 - acc: 0.7705 - ETA: 7s - loss: 0.6072 - acc: 0.7700
	Epoch 18/20
	25000/25000 [==============================] - 13s - loss: 0.6181 - acc: 0.7723 - val_loss: 0.0711 - val_acc: 0.9844 - ETA: 5s - loss: 0.6089 - acc: 0.7746
	Epoch 19/20
	25000/25000 [==============================] - 13s - loss: 0.6152 - acc: 0.7732 - val_loss: 0.1066 - val_acc: 0.9788 - ETA: 5s - loss: 0.6230 - acc: 0.7715 - ETA: 5s - loss: 0.6244 - acc: 0.7714 - ETA: 4s - loss: 0.6261 - acc: 0.7696
	Epoch 20/20
	25000/25000 [==============================] - 12s - loss: 0.6144 - acc: 0.7771 - val_loss: 0.0745 - val_acc: 0.9836 - ETA: 8s - loss: 0.6185 - acc: 0.7739
	Out[14]:
	<keras.callbacks.History at 0x7fd888fb2910>



### Save model


```python
name = 'mnist_CNN_'+str(epochs_for_cnn)+'e_'+str(training_samples_for_cnn)+'i.h5'
model1.save(name)
```

### Results of approach 2

```python
params1 = model1.metrics_names
results1 = (model1.evaluate(cnn_x_test,cnn_y_test,verbose=0))

print params1[0]," : ", results1[0]
print params1[1]," : ", results1[1]
```

    loss  :  0.119442222269
	acc  :  0.9706

> 97.06%
Still good!
(The difference in accuracy between the approaches ain't that big here)

#### Analysis of results

Using a CAE to initialize weights of a CNN provided additional accuracy of 0.2% over a CNN only model.
But in the above case, we had 25000 labelled and 25000 unlabelled samples. Consider the case, where we have only 10000 labelled sample (or even less) and similar number of unlabelled samples. In such a case we should see a benefit in accuracy greater than 0.20%. 
Feel free to play around with the number of labelled and unlabelled samples to find out when the benefits are worth the extra training time of the CAE. Use this notebook to get a primary intuition before applying the same to your task.

{% include disqus_comments.html %}
