---
layout: post
title: "Embedded Machine Learning - Part 6: Train Keras Model on weather data"
categories:
  - Projects
  - Electronics
---

This is a very exciting part of the project: We will build a simple machine learning model to predict the weather in the near future. This form of a very short-term weather forecast is also called ["nowcasting"](https://en.wikipedia.org/wiki/Nowcasting_(meteorology)). Tensorflow has an [excellent tutorial](https://www.tensorflow.org/tutorials/structured_data/time_series) on building a weather forecast model based on historic data. The main difference is that the tutorial predicts temperature, which could be any number. This makes this tutorial as a _regression_ task. What we will try to predict instead is a _category_ for the weather in the near future (Sun, Rain, Clouds, Snow). This means that ours is a _classification_ task.

## The aim
Taking our measurements for temperature, pressure and humidity, what we would like to do is predict one of three categories for the weather in an hour from now: Sun/Clear, Rain and Clouded. To keep things simple, we will simply take the current measurements. In order to improve the forecast, it would be beneficial to also keep track of the past measurements in the model.

## The dataset
In order to predict the category, we need a dataset that contains measurements for pressure, humidity and temperature. It also needs to contain a descriptive label of the current type of weather (which is often no present). I have stumbled upon the [Historical Hourly Weather Data](https://www.kaggle.com/selfishgene/historical-hourly-weather-data) dataset on Kaggle that seems well-suited to this task.

## Data preparation
For the first model, we will only consider the currently measured values as features. The description of the weather one hour later will be our label. I also found it helpful to discard the measurements in the dataset that were taken in cities from a completely different climate zone. My not very sophisticated method was to simply look at the annual average temperature and select the places whose average temperature came closest to where I live. When considering time series, the fact that the dataset is incomplete causes some issues. This is because when we delete an incomplete entry, we don't have a complete time series around that time step anymore. Because we are only considering individual timesteps for now, we luckily don't have to keep track of the cities or timesteps, but can simply squash everything into one big array of observations when reading the data.

```python
cities = ['Indianapolis', 'Portland', 'Seattle', 'Vancouver', 'Boston', 'Chicago', 'Albuquerque', 'Philadelphia', 'New York', 'Pittsburgh']

X = pd.DataFrame()
X['humidity'] = pd.read_csv('kaggle_data/humidity.csv')[cities].values[:,1:].flatten().astype(np.float32)
X['temperature'] = pd.read_csv('kaggle_data/temperature.csv')[cities].values[:,1:].flatten().astype(np.float32)
X['temperature'] -= 273.15 #Kelvin to ºC
X['pressure'] = pd.read_csv('kaggle_data/pressure.csv')[cities].values[:,1:].flatten().astype(np.float32)
X['label'] = pd.read_csv('kaggle_data/weather_description.csv')[cities].values[:,1:].flatten()
```

We can have a look at which categories exist in the dataset:

```python
>>> X.label.value_counts()

sky is clear                           125133
broken clouds                           48203
light rain                              44670
overcast clouds                         42428
scattered clouds                        35009
mist                                    33695
few clouds                              30564
moderate rain                           14186
fog                                      7173
light snow                               4683
heavy intensity rain                     4407
haze                                     3894
light intensity drizzle                  2973
light intensity shower rain              1791
snow                                     1206
proximity thunderstorm                    941
heavy snow                                855
proximity shower rain                     851
smoke                                     730
drizzle                                   533
thunderstorm                              517
very heavy rain                           371
thunderstorm with light rain              253
dust                                      132
thunderstorm with rain                    110
thunderstorm with heavy rain               70
shower rain                                57
heavy intensity drizzle                    41
light shower snow                          40
freezing rain                              29
light rain and snow                        28
squalls                                    23
proximity thunderstorm with rain           22
light intensity drizzle rain               11
heavy shower snow                           9
shower snow                                 7
sand/dust whirls                            7
heavy intensity shower rain                 6
proximity thunderstorm with drizzle         6
thunderstorm with light drizzle             3
thunderstorm with drizzle                   3
sleet                                       3
proximity moderate rain                     3
volcanic ash                                2
light shower sleet                          2
sand                                        2
heavy thunderstorm                          1
ragged thunderstorm                         1
Name: label, dtype: int64
```

...which are way too many to learn all of them, especially considering the fact that most categories are quite don't have many samples. We will instead assign most of the input features to just 4 large categories and simply ignore the rest.

```python
rain = ['light rain', 'moderate rain', 'heavy intensity rain', 'light intensity drizzle', 'drizzle', 'proximity shower rain', 'proximity thunderstorm', 'very heavy rain']
clouds = ['broken clouds', 'scattered clouds', 'overcast clouds', 'mist', 'haze', 'fog']
snow = ['light snow', 'snow', 'heavy snow', 'light shower snow']
clear = ['sky is clear', 'few clouds']

#Reduce number of categories
X.loc[X.label.isin(rain), 'label'] = 'Rain'
X.loc[X.label.isin(clouds), 'label'] = 'Clouds'
X.loc[X.label.isin(snow), 'label'] = 'Snow'
X.loc[X.label.isin(clear), 'label'] = 'Clear'

# Shift label by one to predict the next label instead of the current one
X.label = X.label.shift(1)

# Remove incomplete lines
X = X.dropna()

# Discard timesteps that are not in our categories
X = X[((X.label == 'Rain') | (X.label == 'Clouds') | (X.label == 'Snow') | (X.label == 'Clear'))]
```

First, let's have a look at how the data is distributed:

![Weather Data Distribution](/assets/images/weather_forecast/weather_distribution.png)

There are a few things that are apparent when looking at this distribution:
- The categories are not nicely separable, at least not with the data we have. This means that our model is likely to not be incredibly accurate. We will  later improve upon this with additional observations from the past.
- The 'Clouds' and 'Clear' categories are of a similar size. 'Rain' and 'Snow' are much smaller. This means that our model can get to a relatively high accuracy, even if it never predicts Snow or Rain. This is the [opposite of what we want from a weather forecast](https://en.wikipedia.org/wiki/Wet_bias). We can somewhat address that by adding additional feature from the smaller classes or removing features from larger classes. This needs to be done carefully, though. When simply sampling an equal amount of data points from each class, the model would, for example, learn that temperatures below 0ºC _always_ correspond to snow. (I know this because this is what happened to me)

## Train model
In order to train, we first turn our remaining descriptive labels into numbers

```python
labels, uniques = X['label'].factorize()
X['label'] = labels
```
We also need to split our dataset into a test, training and holdout set. We will use 70% for training and 15% each for test and validation.
```python
from sklearn.model_selection import train_test_split

num_features = 3
X_train, X_test, y_train, y_test = train_test_split(X.values[:,:num_features], X.values[:,num_features], shuffle=True, test_size=0.3)
X_test, X_val, y_test, y_val = train_test_split(X_test, y_test, shuffle=True, test_size=0.5)
```
Finally, we need to make sure that all our feature dimensions are approximately the same magnitude. For this, we normalize temperature, pressure and humidity to have zero mean and unit variance. It is important to do this only on the training data and then use those values to scale the test and validation set as well. The reason for this is that we don't want the model to only have access to information from the training set.
```python
X_mean = X_train.mean(axis=0)
X_std = X_train.std(axis=0)
X_train = (X_train - X_mean) / X_std
X_test = (X_test - X_mean) / X_std
X_val = (X_val - X_mean) / X_std
```
Now is a good time to retrieve the values for mean and standard deviation for each feature. We will be needing those later, when we deploy our model to the MCU.

```python
>>> X_mean
array([  71.46498214,   11.57996851, 1019.26325   ])
>>> X_std
array([21.39268372,  9.22811948, 10.43932569])
```

With this, we are ready to set up a model. Because our input dimensionality is tiny (with only 3x1 elements), we don't need a lot of neurons either. Here's a simple MLP that works reasonably well on the dataset and trains within a few minutes on a CPU:

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

model = keras.models.Sequential()
model.add(layers.Input(shape=(3,)))
model.add(layers.Dense(16, activation='relu', use_bias=False))
model.add(layers.BatchNormalization())
model.add(layers.Dense(16, activation='relu', use_bias=False))
model.add(layers.BatchNormalization())
model.add(layers.Dense(16, activation='relu', use_bias=False))
model.add(layers.BatchNormalization())
model.add(layers.Dense(len(uniques), activation='softmax'))

model.compile(optimizer='adam', 
              loss=keras.losses.SparseCategoricalCrossentropy(), 
              metrics=['accuracy'])
```
With our model compiled, we can have a look at some model data:
```python
>>> print(model.summary())

Model: "sequential_3"
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
dense_12 (Dense)             (None, 16)                48        
_________________________________________________________________
batch_normalization_9 (Batch (None, 16)                64        
_________________________________________________________________
dense_13 (Dense)             (None, 16)                256       
_________________________________________________________________
batch_normalization_10 (Batc (None, 16)                64        
_________________________________________________________________
dense_14 (Dense)             (None, 16)                256       
_________________________________________________________________
batch_normalization_11 (Batc (None, 16)                64        
_________________________________________________________________
dense_15 (Dense)             (None, 4)                 68        
=================================================================
Total params: 820
Trainable params: 724
Non-trainable params: 96
_________________________________________________________________
None
```
We are finally ready to train our model
```python
model.fit(X_train, y_train, validation_data=(X_test, y_test), epochs=3)
```
this should train around 46% validation accuracy in 3 epochs, which is probably as good as we will get when only considering the current measurements only.

Curiously, the validation accuracy ends up being higher than training accuracy...
```python
>>> model.evaluate(X_val, y_val)

1823/1823 [==============================] - 1s 688us/step - loss: 1.0153 - accuracy: 0.47130s - loss: 1.0
```


## Convert Keras Model to TFlite format
We now need to convert our model from Keras the TFlite format in order to be able to feed it to the TFlite micro runtime environment. We will also quantize the weights and activations to 8-bit Integers instead of 32-bit Floats to reduce the size as well as inference speed. The process is described in a lot of detail in the [Tensorflow Docs](https://www.tensorflow.org/lite/performance/post_training_quantization). Here's only a quick summary. First, we need to provide a generator that supplies sample data to the converter. This helps the converter in determining the range of our input values which in turn allows it to quantize the weights and biases of the model from floating point to Integer values.

```python
def representative_dataset_gen():
    for i in range(500):
    # Get sample input data as a numpy array in a method of your choosing.
        yield([X_train[i][np.newaxis,:].astype(np.float32)])
```

With the generator, we can convert the model using the default settings and write it to a file.
```python
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_dataset_gen
model_tflite = converter.convert()

with open('model.tflite', 'wb') as f:
    f.write(model_tflite)
```

## Convert model to C array
Just like with Bitmap image for the display, we also need to a way to flash the `*.tflite` file we just created to our board. We will do exactly the same thing as before and convert it to a C array. The process is again described in the [Tensorflow Docs](https://www.tensorflow.org/lite/microcontrollers/build_convert#convert_to_a_c_array) as well. Here's the quick overview:

- Convert model with `xxd -i model.tflite > model.cpp`
- Add a header file `model.h`

  ```c
  #ifndef WEATHER_MODEL_H_
  #define WEATHER_MODEL_H_
  
  extern const unsigned char weather_model_tflite[];
  extern const unsigned int weather_model_tflite_len;
  
  #endif  // WEATHER_MODEL_H_
  ```

- Make the array and length in the generated file constant and align to 8 byte:

  ```c
  #include "model.h"

  alignas(8) const unsigned char weather_model_tflite[] = {
      // [...]
  };
  const unsigned int model_tflite_len = <model length>;
  ```

We now have a C-array that we can use to replace the original 'hello world' model that we extracted from the Tensorflow repository. The relevant files that you need to replace are

```bash
<MTB_Project_Dir>/tensorflow/lite/micro/examples/hello_world/model.cpp
<MTB_Project_Dir>/tensorflow/lite/micro/examples/hello_world/model.h
```
There are a few more things that we need to consider before we can run the model on our MCU. We will go over these in the next part.
