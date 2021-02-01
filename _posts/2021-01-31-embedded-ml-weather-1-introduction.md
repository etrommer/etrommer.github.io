---
layout: post
title: "Embedded Machine Learning - Part 1: Introduction"
categories:
  - Projects
  - Electronics
---

As part of starting a new job, I got pick a project that involved embedded machine learning on an ARM Cortex-M class MCU, as well as environmental sensors. This is a walkthrough of my notes for this project. 

What I came up was training an ML model to use data from environmental sensors to predict the weather in the near future.

![Results 1](/assets/images/weather_forecast/results_1.jpeg)

Specifically, we will tie together the following pieces of technology:
- Cypress PSOC6 MCU (which is an ARM-Cortex M4 with an FPU), along with Cypress' development environment "Modus Toolbox". Modus Toolbox comes with an Eclipse-based IDE, but also has support for VS Code. Since VS Code is my preferred option, we will also go over how to set this up.
- Environmental sensors that provide input to a machine learning model (in our case values for humidity, temperature and air pressure)
- The Tensorflow lite micro runtime that is used to evaluate the ML model on the PSOC6
- Training a simple model using Tensorflow's Keras API

Here's a quick overview of how things will be coming together in the end:

[![Schematic Overview](/assets/drawings/weather_forecast/schematic_overview.png)](/assets/drawings/weather_forecast/schematic_overview.png)

...and here are the individual parts of the series:
- [Part 1: Introduction](2021-01-31-embedded-ml-weather-1-introduction.md)
- [Part 2: Setting up Modus Toolbox and Hello World on the PSoC 6](2021-01-31-embedded-ml-weather-2-PSOC6-Modus-Toolbox.md) How to get started using the PSoC6 and the ModusToolbox toolkit
- [Part 3: Adding environmental sensors](2021-01-31-embedded-ml-weather-3-Environmental-sensors.md) Interfacing sensor via I²C to gather data for the model
- [Part 4: Running Tensorflow lite micro on the PSoC6](2021-01-31-embedded-ml-weather-4-tensorflow-lite-micro.md) Extracting the Tensorflow lite micro runtime from the Tensorflow repository and running it on an MCU that is not directly supported by the Tensorflow make flow
- [Part 5: Connect TFT display](2021-01-31-embedded-ml-weather-5-connect-tft-display.md) Use the Segger emWin library to display information on the TFT display that comes with the development kit.
- [Part 6: Train Keras Model on weather data](2021-01-31-embedded-ml-weather-6-train-keras-model.md) Use Tensorflows [Keras API](https://keras.io/) to train a simple machine learning model that generates a descriptive label from environmental data
- [Part 7: Deploy Tensorflow lite model](2021-01-31-embedded-ml-weather-7-run-tensorflow-lite-model.md) Convert the model into something that we can evaluate with Tensorflow lite micro and flash to our microcontroller
- [Part 8: Summary](2021-02-01-embedded-ml-weather-8-summary.md) Results and some ideas on how to take this project further

### What this is:
This is mostly a compilation of my project notes as a reference for someone who aims to implement a similar project.I consider it more of a template that might help getting a quicker start for similar projects.

### What this isn't:
An accurate, ready-to-use weather forecast. This is more a learning project than a real weather station - mainly for the very practical reason that you can get high-quality weather forecasts from various APIs all over the internet. Turns out weather is quite hard to predict and that local data often isn't sufficient to capture its dynamics and therefore the performance of any model that only uses local data is likely to be limited.

That said, I had a lot of fun building it and I hope that my notes can be an inspiration for you to build something similar.

### Disclosure note:
I am an employee of Infineon who recently acquired Cypress. This project was part of my work, however this writeup expresses purely my views, not necessarily those of Cypress or Infineon.

### Code:
If you want to follow along, you can find a minimal working version of this project on [Github](https://github.com/jadeaffenjaeger/embedded_ml_weather).
