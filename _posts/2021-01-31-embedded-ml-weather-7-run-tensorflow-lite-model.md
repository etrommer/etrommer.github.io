---
layout: post
title: "Embedded Machine Learning - Part 7: Deploy Tensorflow lite model"
categories:
  - Projects
  - Electronics
---

We are finally ready to put everything together! Here are the most important things to remember when deploying the model to the MCU. All the rest should only be some boring C/C++ coding to provide a few structs that allow us to conveniently pass data around and a few functions calls to display data on the display.

## Normalize measurements
Before we could train our model, we had to normalize the data so that each feature had zero-mean and unit variance. We now need to do the same thing for our measurements, using the means and deviations from our training data set. For this, we use the means and variances obtained from the training data set and scale our measurements to the dimensions that our model expects:

```c++
// Per-feature mean and std and from training set
const normalization_t norm_factors = {
    .humidity_mean = 71.06214581,
    .humidity_std = 21.15321648,
    .temperature_mean = 10.44727886,
    .temperature_std = 9.32877858,
    .pressure_mean = 1020.73639297,
    .pressure_std = 10.90652208
};

void sensors_normalize(measurement_t *measurement, const normalization_t *norm_factors) {
    measurement->humidity = (measurement->humidity - norm_factors->humidity_mean) / norm_factors->humidity_std;
    measurement->temperature = (measurement->temperature - norm_factors->temperature_mean) / norm_factors->temperature_std;
    measurement->pressure = (measurement->pressure / norm_factors->pressure_mean) / norm_factors->pressure_std;
}
```

## Adapt inference code
Before you can run the inference, make sure to adapt the inference code under `<MTB_Project_Dir>/tensorflow/lite/micro/examples/hello_world/main_functions.cpp` so that it takes 3 arguments (the measurements) instead of 1 from the example and passes them on to the model. 

Here's my slightly modified `loop` function:

```c++
void loop(measurement_t *measurement, predictions_t *pred) {

  // Pass measurements to model
  input->data.f[0] = measurement->humidity;
  input->data.f[1] = measurement->temperature;
  input->data.f[2] = measurement->pressure;

  // Run inference, and report any error
  TfLiteStatus invoke_status = interpreter->Invoke();
  if (invoke_status != kTfLiteOk) {
    TF_LITE_REPORT_ERROR(error_reporter, "Invoke failed.\n");
    return;
  }

  // Read the predicted class scores from the model's output tensor
  float max_score = 0;
  uint8_t max_idx = 0;

  for(size_t i = 0; i < NUM_CLASSES; i++) {
    if(output->data.f[i] > max_score) {
      max_score = static_cast<float>(output->data.f[i]);
      max_idx = static_cast<uint8_t>(i);
    }
  }

  pred->class_name = class_names[max_idx];
  pred->class_score = max_score;
  pred->class_type = static_cast<weather_class_t>(max_idx);
}
```
From this, we obtain the ID of the class the model thinks is most likely - given our input data.

