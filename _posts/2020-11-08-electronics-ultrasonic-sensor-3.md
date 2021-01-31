---
layout: post
title: "Building an Ultrasonic Distance Sensor from Scratch - Part 3: Transmitter Code"
categories:
  - Electronics
---

After we went over the fundamentals of the receiver, we will now go into what we need in order to transmit ultrasonic pulses. This will be much shorter than the first section, since we are esentially only (ab-)using a single component to amplify a 40kHz square wave which we generate with our MCU.

## Theory
An ultrasonic transmitter is really just a less capable version of a speaker: Something gets swung back and forth by a changing electric field, creating waves in the medium around it. For us, that means that we need to create a 40kHz square wave. The higher the amplitude of our square wave, the more energy our speaker will transmit (and the stronger the reflection we can measure becomes, meaning more range). The problem is: creating square waves with a positive and negative swing and
sufficient driving power usually requires quite a few components. Not ideal for something that's supposed cheap and lightweight and relatively easy to manufacture. Luckily, someone has already gone through the trouble for us and built a component that already brings most of what we need: The [MAX232](https://en.wikipedia.org/wiki/MAX232) is a component that is originally designed to translate [RS232](https://en.wikipedia.org/wiki/RS-232) voltages that can range from -15V to 15V to friendlier
0-5V TTL levels. It also packs two [charge pumps](https://en.wikipedia.org/wiki/Charge_pump) that are capable of doubling the supply voltage range.

## Circuit Design
The sender circuit is almost a verbatim copy from the reference circuit found in the datasheet

![Sender Circuit](/assets/images/ultrasonic/circuit_sender.png)

Here are the individual pins and what we are using them for:
- `T1OUT/T2OUT`: High voltages outputs that will drive our speaker
- `T1IN/T2IN`: TTL inputs that we use to control the voltage that gets applied to our speaker
- `R1OUT/R2OUT/R1IN/R2IN` These are for receiving RS232 signals. We don't need them, so we'll simply leave them unconnected.
- `C1+/C1-/C2+/C2-/VS+/VS-`: Storage capacitors for the two charge pumps. We'll simply stick with the value from the datasheet here and connect four 1uF ceramic capacitors. 
- `VCC` Supply voltage. There's a PNP transistor connected in series in order to disable the MAX232 when it's not needed. This should be most of the time, since we only use it to transmit a short burst and then wait for the response. Other parts like the MAX222 come with a built-in Shutdown/Enable pin to do this, but they are more expensive and harder to source.




## Code
The code to control the transmitter from the MCU side is relatively simple: Output a 40kHz square wave on one pin and an enable signal for the MAX232 on another pin. We'll pick the enable signal so that 10 full square waves can be transmitted in each pulse. That gives an on time for the MAX232 of:

$$
T_\text{on} = 10 \cdot \frac{1}{40 kHz} = 250 \mu s
$$

To calculate how long the off time needs to be, we need to know how far the pulse travels in our measurement. Let's assume that the farthes object that we would like to measure is 4m away. We'll need to double that because the impulse has to travel to the object and back and then divide the distance by the speed of sound in air. That gives us:

$$
T_\text{off} = \frac{2 \cdot 4m}{343.2 m/s} = 23.3 ms
$$

### Arduino
The first prototype is built using an Arduino. Since we require relatively precise timing, we'll in part circumvent the convenient functions that Arduino IDE offers and manipulate the `Timer/Counter0` of the ATmega328p directly.

```cpp
void setup() {
    pinMode(5, OUTPUT);
    digitalWrite(5, LOW);

    pinMode(6, OUTPUT);

    TCCR0A = (1 << COM0A0) | (1 << WGM01) | (1 << WGM00);
    TCCR0B = (1 << WGM02) | (1 << CS01);
    OCR0A = 24;
}

void loop() {
    PORTD &= ~(1 << 5);
    my_delay(400);
    PORTD |= (1 << 5);
    my_delay(10000);
}

void my_delay(long ms) {
    volatile int count = ms;
    while(count--);
}
```
I found the time of the builtin delay function on the Arduino to be inaccurate for my use case which is why I helped myself with the crude counting delay that you see in the listing. This produces the correct pulse timing for the application on my board.

### STM32
For some more complicated evaluation of the ADC signal, a more capable ARM-Cortex-based MCU will likely be beneficial. This unfortunately brings with it the headache of changing the circuitry to run on 3.3V, rather than the old-fashioned 5V it's running on at the moment. As soon as I get around to doing it, I will update these articles.

### Up next
After we have a working circuit for sender and receiver and can emit pulses, the only thing that is missing is some ADC code the measures the delay between emitting the pulse and receiving the peak of the reflected signal.
