---
layout: post
title: "Embedded Machine Learning - Part 1: Introduction"
categories:
  - Projects
  - Electronics
---

As part of starting a new job, I got pick a project that involved embedded machine learning on an ARM Cortex-M class MCU, as well as environmental sensors. This is a walkthrough of my notes for this project. 

What I came up was training an ML model to use data from environmental sensors to predict the weather in the near future.

Specifically, we will tie together the following pieces of technology:
- Cypress PSOC6 MCU (which is an ARM-Cortex M4 with an FPU), along with Cypress' development environment "Modus Toolbox". Modus Toolbox comes with an Eclipse-based IDE, but also has support for VS Code. Since VS Code is my preferred option, we will also go over how to set this up.
- Environmental sensors that provide input to a machine learning model (in our case values for humidity, temperature and air pressure)
- The Tensorflow lite micro runtime that is used to evaluate the ML model on the PSOC6
- Training a simple model using Tensorflow's Keras API

Here's a quick overview of how things will be coming together in the end:

[![Schematic Overview](/assets/drawings/weather_forecast/schematic_overview.png)](/assets/drawings/weather_forecast/schematic_overview.png)

### What this is:
This is mostly a compilation of my project notes as a reference for someone who aims to implement a similar project.I consider it more of a template that might help getting a quicker start for similar projects.

### What this isn't:
An accurate, ready-to-use weather forecast. This is more a learning project than a real weather station - mainly for the very practical reason that you can get high-quality weather forecasts from various APIs all over the internet. Turns out weather is quite hard to predict and that local data often isn't sufficient to capture its dynamics and therefore the performance of any model that only uses local data is likely to be limited.

That said, I had a lot of fun building it and I hope that my notes can be an inspiration for you to build something similar.

### Disclosure note:
I am an employee of Infineon who recently acquired Cypress. This project was part of my work, however this writeup expresses purely my views, not necessarily those of Cypress or Infineon.

### Code:
If you want to follow along, you can find a minimal working version of this project on [Github](https://github.com/jadeaffenjaeger/embedded_ml_weather).
