---
layout: post
title: "Embedded Machine Learning - Part 8: Summary"
categories:
  - Projects
  - Electronics
---

We have come to the end of this small project. I hope the writeup contained some useful information and showed that - while a bit complicated at times - simple ML models can be deployed even to small targets with relative ease.

## Results
Let's have a look at some results. I put the board outside, connected to a USB power bank over the course of a couple of days and took pictures of the predictions and the weather, about an hour after the predictions:
![Results 1](/assets/images/weather_forecast/results_1.jpeg)
![Results 2](/assets/images/weather_forecast/results_2.jpeg)
![Results 3](/assets/images/weather_forecast/results_3.jpeg)

The predictions are usually close enough, but not always super accurate:

![Results 4](/assets/images/weather_forecast/results_4.jpeg)

## Where to go from here?
The accuracy of an ML model on the raw sensor data isn't great by itself. Here are a few of my ideas on how to improve the accuracy - given more time to work on this:
- Add a camera: The camera image could be fed to a [Convolutional Neural Network](https://en.wikipedia.org/wiki/Convolutional_neural_network). This could then be used to detect the _current_ weather. This could be used in two ways: 1. It can be used as an additonal input parameter to the model. 2. In addition, it can be used as a label for a prediction that was made in the past. With this, the forecast could continually improved through an online learning process.
- The geolocation as well as the time of the day and the current season might provide additional useful information that will probably help to improve the forecast.
- The dataset should be balanced more carefully so that it
    1. Does not always prefer the most frequent category
    2. Also does not treat all classes as equally likely
  The truth here is probably somewhere in the middle between the two.
- Information on the current wind, rain and UV radiation through additional sensors will probably also provide useful information to learn from
- There are [existing systems](https://en.wikipedia.org/wiki/Zambretti_Forecaster) for short-term weather forecasts. Looking into how to augment these with additional input from a machine learning model would certainly make for an interesting way of extending this project.
- For a proper outdoor installation, the board would definitely need some weather protection. Keeping the power-hungry TFT display on all the time is also not ideal and practically rules out battery powered applications.
