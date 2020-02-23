---
layout: post
title: "Building an Ultrasonic Sensor from Scratch - Part 1: Receiver Design"
categories:
  - Electronics
---

## Introduction

I'll assume that you are familiar basic principles of how an ultrasonic distance measuring works. If not, check out [this excellent tutorial](https://lastminuteengineers.com/arduino-sr04-ultrasonic-sensor-tutorial/) first! It'll walk you through how to measure distances with ultrasound.

## Purpose

While there are cheap, ready-made ultrasonic sensors with good support like the ubiquitous [HC-SR04](https://cdn.sparkfun.com/datasheets/Sensors/Proximity/HCSR04.pdf), I needed to roll my own for a project. I needed precise access to the timing, which is not available through the HC-SR04, because it has an integrated MCU that handles all the timing for you. I also wanted to run the analog ouput signal into the MCU's ADC directly. Mainly because I wanted to play around with a few different algorithms to increase the robustness of the object detection and because I was hoping to increase the accuracy by having the resolution of an ADC running at 1MHz and with DMA attached to it. The HC-SR04 only uses a simple thresholding which can only detect a single peak. I bought a handful of [Murata MA40S](https://www.murata.com/~/media/webrenewal/products/sensor/ultrasonic/open/datasheet_maopn.ashx?la=en) transducers and receivers a while back and thought that it would make for an interesting project to build a circuit around them.

## Receiver Design

The receiver circuit is probably the more complicated part of the project. Since the receiver itself only produces a very small signal, we need a circuit that properly amplifies it. For the most part, I followed the [HC-SR04 receiver circuit](http://www.pcserviceselectronics.co.uk/arduino/Ultrasonic/electronics.php#circuit). It uses four operational amplifiers for four separate stages:

1. Inverting High-Pass Amplifier
2. Band-Pass Amplifier
3. Inverting High-Pass Amplifier
4. Comparator/Thresholding Circuit

I've more or less copied the first three stages, but replaced the last one with an envelope detection, the output of which gets fed into an ADC.

Let's go over what each stage does in more detail:

### Stage 1: Inverting High Pass Amplifier

Here's a schematic of what the first stage looks like:

![Inverting High Pass Amplifier](/assets/images/ultrasonic/inverting_amp.png)

now let's go over what it does in detail. It's first order high pass, meaning that it attenuates lower frequencies while letting higher frequencies pass. First order comes from the fact that there is only one element in there that influences the frequency response, namely the capacitor. Now let's do some math to find out how the circuit behaves: 


$$
\begin{align}
A_v &= -\frac{R_2}{R_1}\\
	&= -\frac{47k\Omega}{10k\Omega}\\
	&= -4.7\\
f_C &= \frac{1}{2 \pi \cdot R_1 \cdot C}\\
	&= \frac{1}{2 \pi \cdot 10k\Omega \cdot 10nF}\\
    &= 1592 \text{ kHz}
\end{align}
$$

So we're getting 4.7 times the input signal with a 3db attenuation at 1592 kHz. The fact that the input signal is inverted doesn't really matter to us, since we will detect the envelope later one anyway.

### Stage 2: Bandpass

The second stage is quite a bit more complicated. It's a rather clever circuit known as a [Multiple Feedback Band-Pass Filter](http://www.ecircuitcenter.com/Circuits/MFB_bandpass/MFB_bandpass.htm), and it looks like this:

![Multiple Feedback Band-Pass](/assets/images/ultrasonic/mf_bandpass.png)

It's helps to pick **C1 = C2** so that the calculations are a bit easier. Here are the filter characteristics:


$$
\begin{align}
R_{1,2} &= R_1 || R_2\\
		&= 194 \Omega\\
f_0 	&= \frac{1}{2 \pi \cdot C \sqrt{R_{1,2} \cdot R_3}}\\
		&= \frac{1}{2 \pi \cdot 1nF \cdot \sqrt{ 194 \Omega \cdot 75k\Omega}}\\
		&= 41.7 \text{kHz}\\
BW 		&= \frac{2}{R_3 \cdot C}\\
		&= \frac{2}{75k\Omega \cdot 1nF}\\
        &= 26.7 \text{kHz}\\
Q 		&= \frac{1}{2} \sqrt{\frac{R_3}{R_{1,2}}}\\
		&= \frac{1}{2} \sqrt{\frac{75k\Omega}{194 \Omega}}\\
		&= 9.8
\end{align}
$$


This means that the filter is (roughly) centered around the carrier frequency of 40kHz and will attenuate frequencies that are 26.7kHz above and below that with 3dB. The Q factor tells us how steep the attenuation curve is.

### Stage 3: Inverting amplifier

This stage is essentially the same as the first, which is why I won't go over it in detail. It has a slightly higher **R2** value, which means that is has an overall amplification of **A = 7.5**. 

### Stage 4: Envelope detection
coming soon...

## Simulation
coming soon...

## Putting everything together

coming soon...