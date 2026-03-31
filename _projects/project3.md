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

## The Bill of Materials

If you want to replicate this, here is exactly what you need:

* **ESP32 Development Board:** Look for an ESP32-WROOM-32 Dev Kit with pre-soldered headers.
* **Capacitive Soil Moisture Sensor (v1.2 or v2.0):** This is critical. Do not buy the cheap "resistive" sensors with bare metal prongs. They suffer from electrolysis and will corrode away into the soil within weeks. Capacitive sensors are encased and measure moisture via changes in capacitance. 
* **Adafruit 4x AA Battery Holder:** The one with the switch, cover, and header pins.
* **4x AA NiMH Rechargeable Batteries:** High-capacity Eneloops or Ikea Ladda cells work perfectly.
* **Waterproof Junction Box:** An IP65 or IP67 rated plastic enclosure (roughly 100mm x 68mm x 50mm).
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
