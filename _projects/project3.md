---
title: "ESP32 Based automatic irragation system"
date: 2025-07-12
category: Electronics and Hardware
description: "ESP32 based system to monitor and water plants outdoors."
status: In-progress
tools: [Python, C++, Arduino IDE]
github: N/A
---  
# Ditching the Pi: Building a Wireless, AA-Powered ESP32 Soil Sensor

Sometimes, you just need a break from differential equations and advanced engineering mechanics problem sets to build something tangible. I recently set out to create a wireless, outdoor soil moisture sensor, but my initial thought—using a Raspberry Pi—was massive overkill. The Pi is a power-hungry Linux computer, and keeping it alive in the dirt would have been a nightmare. 

I needed an "avionics" approach for the backyard: lightweight, purpose-built, and highly efficient. That meant pivoting to the ESP32 microcontroller and rethinking my power supply entirely. Here is how I designed a "Swap & Go" AA-powered soil sensor that can survive the elements and run for months on a single charge.

---

## Why the ESP32 and Why AA Batteries?

The magic of the ESP32 is its "Deep Sleep" function. It can power down its Wi-Fi radio and drop its current draw to practically nothing (~10µA), wake up on a timer, take a reading, transmit the data, and immediately go back to sleep. 

Initially, I looked into lithium polymer (LiPo) batteries and solar panels, but I wanted something simpler and more resilient to temperature swings. Lithium batteries notoriously struggle in freezing weather. 

My solution? A 4x AA battery pack from Adafruit. 
By loading it up with four NiMH rechargeable batteries (like Eneloops), I get a nominal output of 4.8V (5.2V fully charged, 4.4V discharged). This is the absolute sweet spot for the ESP32. The development board has an onboard voltage regulator that needs at least 4.4V to step down to a clean 3.3V. Plus, the Adafruit pack comes with a built-in on/off switch and standard 0.1" headers, which makes assembly incredibly clean. When the system eventually pings me with a low-battery alert, I just walk outside and swap the AAs.  

The only issue with this design was it wasn't truly autonomous, I would still have to go out and change the battries every now and then. I did a bit of digging and found out about Weather resiliant Li-PO battries from this [paper](https://www.sciencedirect.com/science/article/pii/S0378775322015270). From this I went back to my original idea of a rechargable Li-Po battery; this had the added benefit of being rechargable; that the TP4056 handles assisted by the solar panel; keeping the battery topped up.  
Now just like on your phone, over time the life of the battery degrades if you cycle it too much. To prevent this I would only change the battery to around 80% and start the recharge process when the battery dropped below 20%. I was able to come to this conclusion based on these calculations:  
### The Baseline: 0% to 100% Cycling
When you charge a battery to 100% and drain it to 0%, you are doing a 100% Depth of Discharge (DoD) cycle. 
* **Standard Lifespan:** ~500 cycles
* **Equivalent Full Cycles (EFC):** A measure of total energy moved through the battery over its lifetime. 

$$EFC = \text{Cycles} \times \left(\frac{\text{DoD}}{100}\right)$$

$$EFC_{baseline} = 500 \times 1.0 = 500 \text{ total full discharges}$$

### The 20% to 80% Strategy
By stopping the charge at 80% and plugging it in at 20% you can reduce the battery cycle fatigue.

#### 1. The Depth of Discharge (DoD) Factor
Your new DoD is 60% (80% - 20% = 60%).
Battery fatigue follows an inverse power-law curve. When you reduce the DoD, the number of physical cycles the battery can handle increases exponentially. An accepted empirical formula for cycle life based on DoD is:

$$\text{Cycles}_{new} \approx \text{Cycles}_{100\%} \times \left(\frac{100}{\text{DoD}_{new}}\right)^{1.5}$$

If we only factor in the smaller DoD:

$$\text{Cycles}_{DoD} \approx 500 \times \left(\frac{100}{60}\right)^{1.5} \approx 500 \times (1.66)^{1.5} \approx 1,075 \text{ cycles}$$

#### 2. The Voltage Stress Factor
LiPo batteries operate between roughly 3.0V (empty) and 4.2V (100% full). Pushing a battery all the way to 4.2V puts massive chemical stress on the cathode. 

Charging to only 80% drops the peak charging voltage to roughly 4.0V. A common battery engineering rule of thumb is that **every 0.1V drop in peak charge voltage below 4.2V roughly doubles the cycle life**.
Since you dropped the peak voltage by about 0.2V, we apply a voltage multiplier:

$$\text{Voltage Multiplier} \approx 1.5 \text{ to } 2.0 \text{ (conservatively)}$$

When we combine the reduced DoD with the reduced high-voltage stress, the total physical cycles jump significantly:

$$\text{Total Cycles}_{20-80} \approx 1,075 \times 1.5 \approx 1,600 \text{ cycles}$$

### The Final Comparison: Equivalent Full Cycles (EFC)
You are getting 1,600 cycles, but remember, each cycle only uses 60% of the battery. To compare "apples to apples" with our baseline, we calculate the Equivalent Full Cycles:

$$EFC_{20-80} = 1,600 \text{ cycles} \times 0.60 \text{ (DoD)} = 960 \text{ Equivalent Full Cycles}$$

## The Bill of Materials

If you want to replicate this, here is exactly what you need:

* **ESP32 Development Board:** Look for an ESP32-WROOM-32 Dev Kit with pre-soldered headers.
* **Capacitive Soil Moisture Sensor (v1.2 or v2.0):** This is critical. Do not buy the cheap "resistive" sensors with bare metal prongs. They suffer from electrolysis and will corrode away into the soil within weeks. Capacitive sensors are encased and measure moisture via changes in capacitance. 
* **Adafruit Li-PO Battery Holder:** The one with the switch, cover, and header pins.
* **1x Weather hardened Rechargeable Batteries:** High-capacity Eneloops or Ikea Ladda cells work perfectly.
* **Waterproof Junction Box:** An IP65 or IP67 rated plastic enclosure (roughly 100mm x 68mm x 50mm).
* **TP4056 Battery charger** A popular, low-cost, and complete linear charger integrated circuit
* **Cable Gland (PG7):** To create a waterproof seal where the sensor wires exit the box.
* **Misc:** Female-to-Male jumper wires, a few packets of silica gel (desiccant), and optionally some clear nail polish or conformal coating.
![Wiring Diagram]({{ "/assets/esp32.png" | relative_url }})

## The Wiring Strategy (The Battery Saver Hack)

Wiring this up is straightforward, but there is one major hardware trick you need to implement to save battery life. 

1. **Main Power:** Connect the red wire from the battery pack to the **VIN** (or 5V) pin on the ESP32. Connect the black wire to a **GND** pin. *Never connect the battery directly to the 3V3 pin, or you'll bypass the regulator and fry the board.*
2. **Sensor Data:** Connect the analog output (AOUT) of the sensor to **GPIO 34**.
3. **Sensor Power (The Hack):** Do not connect the sensor's VCC to a constant 3.3V pin. If you do, the sensor will draw power 24/7, killing your AA batteries in days. Instead, connect the sensor's VCC to a digital pin (like **GPIO 17**). In your code, you can set this pin `HIGH` right before you take a reading, and set it `LOW` immediately after. 

## Weatherproofing for the Real World

Building the circuit is only half the battle; the other half is making sure the morning dew doesn't short the whole thing out. 

The ESP32 and the battery pack sit snugly inside the waterproof junction box. To get the sensor outside, I drilled a hole in the bottom of the box, installed the PG7 cable gland, and passed the sensor wires through, tightening the nut to compress the rubber seal around the cables. 

Because temperature swings can cause condensation *inside* a sealed box, I tossed two small silica gel packets inside before screwing the lid shut. For extra peace of mind, you can brush a layer of clear nail polish over the exposed components on the ESP32 board (avoiding the USB port and the metal Wi-Fi shield) to act as a poor man's conformal coating.

The hardware is solid, the power math checks out, and it's ready for the dirt. Next up: writing the C++ code to handle the Deep Sleep cycles and Wi-Fi transmission.
