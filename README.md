# Encoder Debouncer
An simple PCB that takes a rotary encoder and debounces one of the pins

<img src="/Images/isometric-render.PNG"/>

## Inspiration
As a hobbyist, I use Arduino frequently to prototype projects, and I've had to use Rotary Encoders from time to time. Anyone who's used a Rotary Encoder can vouch for what a pain they are to use with Arduino's hardware interrupts, because the mechanical nature of the encoder has bounce that needs to be filtered.

There are many techniques out there for eliminating such bouncing, both in software and in hardware, software being easier to implement, but hardware being more reliable, yet more expensive. I decided to make a PCB that implements a hardware debouncing solution, because I can print multiple such PCBs that all work identically.

## Principle of Operation (hardware)
The Rotary Encoder has 2 signal pins. One of those pins is fed into an RC circuit that absorbs the bounce, but turns the digital signal into a slow-rising curve. This is fed into a Schmitt Trigger, which offers hysteresis and turns the rising voltage curve into a clean fast-rising digital signal.

I didn't have any Schmitt triggers laying around, and I didn't want to buy some just for this circuit, so I used a common 555 IC as a Schmitt trigger. It works quite well for this purpose, and I have plenty of them laying around since they are so versatile.

This circuit is heavily based on Jeremy Blum's [Arduino Tutorial #10 on Interrupts and Hardware Debouncing](https://youtu.be/CRJUdf5TTQQ), it quite excellent.

To debounce a single signal, I used a single resistor, a single capacitor, and a 555 IC. To lower the component count, I only debounced one of the 2 signals, which is good enough because we only need one signal to feed into an interrupt pin.


## Principle of Operation (software)
If you've worked with encoders before, you might be familiar with the below diagram of the 2 signal pins
<img src="/Images/timing-diagram.png"/>

You can see how the raw signal has some random bounce, but it stabilizes _before_ the important transitions in the debounced signal (the red transition and green transition).

This assumption is **critical**, and allows us to use a single interrupt, the debounced signal. Choosing a RISING or FALLING edge to trigger the interrupt makes no difference, but once the interrupt is triggered, we only need to check the state of the raw signal to determine which direction we moved in. 

Let's say for example we choose a RISING edge as our interrupt trigger, and let's assume that in the diagram, going to the right is clockwise. If we are turning clockwise, the rising edge will be the one indicated by the red line in the diagram, and we can see that the raw signal is LOW. If we are going counter-clockwise, the rising edge of the debounced signal is now the green vertical line, and we can see the raw signal is now HIGH.

Below is some example code that applies this principle in practice.


## Example code for Arduino Uno
```
const int ENC_DEB = 2;  // Debounced Encoder signal, feeds into interrupt 0 on Uno
const int ENC_RAW = 7;  // Non-Debounced Encoder signal, feeds into any pin

// A counter variable that is incremented/decremented
// whenever the encoder is rotated. Initial value is arbitrary
int counter = 300;

void setup() {
    pinMode(ENC_DEB, INPUT);
    pinMode(ENC_RAW, INPUT);

    // Execute the encoderISR function on a RISING edge
    attachInterrupt(0, encoderISR, RISING);
}

void encoderISR() {
    // If the counter variable is incrementing/decrementing in the opposite direction,
    // flip the increment/decrement below
    if (digitalRead(ENC_RAW))
        counter ++;
    else
        counter --;

    // Print the value of the counter every time it changes
    Serial.println(counter);
}

void loop() {
    // Nothing necessary here for interrupt
}
```

## Schematic
<img src="/Images/schematic.PNG"/>
You can build this schematic on a breadboard and test it. I tried to use parts that are common to most electronics labs.

## Creating a PCB
This Schematic/PCB was created using Altium Designer, but I expect that many people do not have access to the specific software. Therefore, I have included the Gerber files in this repository, which you can view yourself or send to a PCB Manufacturing facility. Make sure your fabrication house can accomodate for the specifications of the board. The specs of the PCB are the following:

Attribute | Value
--- | ---
Min. Hole Size | 15 mil
Min Silkscreen Width | 5 mil
Min Trace Width | 15 mil
Board Dimensions | 535 mil x 745 mil

## Components BOM
Quantity | Part | Package | DigiKey Part Number
-------- | ---- | ------- | -------------------
1 | 555 IC | 8-VSSOP | [296-42639-1-ND](https://www.digikey.com/product-detail/en/texas-instruments/LM555CMMX-NOPB/296-42639-1-ND/5455904)
3 | RES, 10K | SMD 0805 | [311-10KARCT-ND](https://www.digikey.com/product-detail/en/yageo/RC0805JR-0710KL/311-10KARCT-ND/731188)
1 | CAP, 0.1uF | SMD 0805 | [311-3562-1-ND](https://www.digikey.com/product-detail/en/yageo/CC0805JRX7R8BB104/311-3562-1-ND/7648489)
1 | Rotary Encoder | | (see note 1 below)
1 | 5-pin Male Header | 0.1" (2.54 mm) Pitch |

(note 1) The footprint for the encoder has 5 holes, 3 on one side and 2 on the other. The holes are 0.1" apart, and the 2 rows of holes are 0.6" apart. Most cheap rotary encoders have very similar footprints. Worst case you will have to bend the pins a little bit to make them fit nicely

